# Frigate NVR — stack isolada

NVR (gravador de câmeras) com detecção de objetos por IA, rodando como uma
**stack Docker separada** — não interfere em nenhum outro serviço do servidor.

## Por que uma stack separada?

- **Mesmo Docker**, projeto isolado. `docker compose up/down` aqui só afeta o Frigate.
- Zero risco de estragar Open WebUI, Loja, ComfyUI, CrewAI, etc.
- Portas escolhidas pra **não conflitar** com as que você já usa
  (`3000, 3001, 5050, 7860, 7861, 7862, 7999, 8000, 8188`).

| Porta | Para quê |
|-------|----------|
| 8971  | Interface web (com login) — `http://192.168.15.11:8971` |
| 8554  | RTSP restream (go2rtc) |
| 8555  | WebRTC (ao vivo) |

## Consumo de recursos — com TETO definido (você decide)

O container tem **limite rígido** de memória e CPU no `docker-compose.yml`. Ele
**nunca** passa disso — não é "poço sem fundo". Ajuste os números conforme o
número de câmeras:

| Câmeras | RAM típica | Sugestão `mem_limit` | Sugestão `cpus` |
|---------|-----------|----------------------|-----------------|
| 1–2     | ~0,8–1,3 GB | `1.5g`             | `1.0`           |
| 3–4     | ~1,3–2 GB   | `2g` (padrão)      | `2.0`           |
| 5–8     | ~2–3,5 GB   | `3g`–`4g`          | `2.0`–`3.0`     |

O que ocupa memória: o processo do Frigate + **um ffmpeg por câmera** (decodifica
o vídeo) + `shm_size` (frames) + o cache em RAM (`tmpfs`, hoje **256 MB**).

Onde mudar, no `docker-compose.yml`:
- `mem_limit` → teto de RAM (o Docker segura o container aqui)
- `cpus` → máximo de núcleos
- `shm_size` → `128mb` p/ 1–2 câmeras, `256mb` p/ ~4
- o `tmpfs size` → cache de gravação em RAM (só aumente se der "cache cheio")

## Detecção na CPU (a GPU fica livre)

Seu painel avisa que a GPU de 16GB roda **uma IA pesada por vez**. Como o Frigate
roda 24/7, ele está configurado pra detectar objetos **na CPU**, deixando a GPU
100% livre pro ComfyUI / CrewAI / Whisper.

- Quer o máximo de eficiência? Um **Google Coral** (~US$60) faz a detecção de
  várias câmeras gastando quase nada. Veja o comentário em `config/config.yml`.
- Quer usar a GPU só pra **decodificar vídeo**? Descomente o bloco `deploy` no
  `docker-compose.yml` e a linha `hwaccel_args: preset-nvidia` no `config.yml`.

## Primeira instalação (no servidor 192.168.15.11)

```bash
cd frigate

# 1. Crie o arquivo de senha
cp .env.example .env
nano .env            # preencha FRIGATE_RTSP_PASSWORD

# 2. Ajuste o caminho de gravação no docker-compose.yml
#    a linha:  - /mnt/nas/frigate:/media/frigate
#    troque /mnt/nas/frigate pela pasta real do seu NAS
nano docker-compose.yml
mkdir -p /mnt/nas/frigate     # (ou o caminho que você escolheu)

# 3. Preencha suas câmeras em config/config.yml (veja abaixo)
nano config/config.yml

# 4. Suba
docker compose up -d
docker compose logs -f        # acompanhe a inicialização (Ctrl+C pra sair)
```

Acesse: **http://192.168.15.11:8971** (na primeira vez ele pede pra criar a senha de admin).

## Como achar o link RTSP da sua câmera

O formato muda por marca. Exemplos comuns (porta 554):

| Marca | Exemplo de URL RTSP (substream) |
|-------|----------------------------------|
| Intelbras / Dahua | `rtsp://admin:SENHA@IP:554/cam/realmonitor?channel=1&subtype=1` |
| Hikvision | `rtsp://admin:SENHA@IP:554/Streaming/Channels/102` |
| Reolink | `rtsp://admin:SENHA@IP:554/h264Preview_01_sub` |
| TP-Link Tapo | `rtsp://usuario:SENHA@IP:554/stream2` |

- **subtype=1 / Channels/102 / _sub / stream2** = substream (baixa resolução) — use pra **detecção**.
- O stream principal (alta resolução) é usado pra **gravação**. Você pode
  configurar os dois; comece só com um pra validar.
- Ferramenta útil pra testar a URL: VLC → "Abrir stream de rede".

## Descobrir o RTSP de uma câmera CareCam (via ONVIF)

As CareCam (HMT) **não revelam o caminho RTSP** — testar caminhos no chute não
funciona (elas devolvem uma sessão vazia pra qualquer path). O jeito certo é
perguntar via **ONVIF** (porta 8899). Para uma câmera nova, troque `CAM` pelo IP:

```bash
CAM=192.168.15.12   # IP da câmera nova
# 1) autentica no ONVIF (admin, senha em branco) e pega os perfis:
head -c 20 /dev/urandom > /tmp/n.bin; C=$(date -u +%Y-%m-%dT%H:%M:%SZ)
N=$(base64 -w0 /tmp/n.bin); cat /tmp/n.bin > /tmp/d.bin; printf '%s' "$C" >> /tmp/d.bin
D=$(openssl dgst -sha1 -binary /tmp/d.bin | base64 -w0)
for T in 000 001; do
  curl -s -m 8 "http://$CAM:8899/onvif/device_service" \
   -H 'Content-Type: application/soap+xml' \
   -d "<s:Envelope xmlns:s=\"http://www.w3.org/2003/05/soap-envelope\" xmlns:trt=\"http://www.onvif.org/ver10/media/wsdl\" xmlns:tt=\"http://www.onvif.org/ver10/schema\"><s:Header><Security s:mustUnderstand=\"1\" xmlns=\"http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd\"><UsernameToken><Username>admin</Username><Password Type=\"http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-username-token-profile-1.0#PasswordDigest\">$D</Password><Nonce>$N</Nonce><Created xmlns=\"http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd\">$C</Created></UsernameToken></Security></s:Header><s:Body><trt:GetStreamUri><trt:StreamSetup><tt:Stream>RTP-Unicast</tt:Stream><tt:Transport><tt:Protocol>RTSP</tt:Protocol></tt:Transport></trt:StreamSetup><trt:ProfileToken>$T</trt:ProfileToken></trt:GetStreamUri></s:Body></s:Envelope>" \
   | grep -oiE 'rtsp://[^<]+'
done
```

Na prática, o padrão dessas câmeras é sempre:
`rtsp://admin:@IP:554/streamtype=0` (principal) e `.../streamtype=1` (sub).
Ou seja, pra uma câmera nova, geralmente basta trocar o IP.

## Adicionar mais câmeras

Em `config/config.yml`:
1. Adicione um stream novo em `go2rtc: -> streams:` (ex.: `camera_fundos:`).
2. Copie o bloco de câmera comentado (`camera_fundos`) e ajuste nome/resolução.
3. `docker compose restart`

## Comandos do dia a dia

```bash
docker compose up -d          # subir
docker compose down           # parar (não apaga gravações)
docker compose restart        # reiniciar após mudar o config
docker compose logs -f        # ver logs
docker compose pull && docker compose up -d   # atualizar o Frigate
```

> **Segurança:** o `.env` (com a senha) fica fora do git. Nunca exponha a porta
> `5000` na internet — ela não tem login. Use sempre a `8971`.
