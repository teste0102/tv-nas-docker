# tv-nas-docker — contexto do projeto

Repositório de configurações Docker do servidor de mídia/NAS do usuário.
Cada serviço novo entra como **stack Docker isolada** (pasta própria com seu
`docker-compose.yml`), pra não interferir no que já roda.

> ⚠️ **Nunca gravar senhas neste arquivo** (ele vai pro GitHub). Senhas de
> câmera ficam no `.env` de cada stack (protegido pelo `.gitignore`); a senha
> da interface do Frigate é criada no primeiro acesso à web.

## Acesso ao servidor

- **SSH:** `ssh mkinfocell@192.168.15.11`
- **Usuário Linux:** `mkinfocell`
- **Hostname:** `mkinfocell-Default-string`
- **Pasta do projeto no servidor:** `~/tv-nas-docker` (clonado do GitHub)
- **Idioma do usuário:** Português (responder sempre em PT-BR)

## Hardware

- **GPU:** 16 GB de VRAM — **roda UMA IA pesada por vez**. Serviços de IA
  disputam a GPU, então cargas 24/7 (como o Frigate) devem rodar na **CPU**.

## Serviços que já existem no servidor (portas EM USO — evitar conflito)

Painéis (ligados/desligados sob demanda):
| Serviço | Porta |
|---|---|
| Vigia da Rede | 7999 |
| Assistente IA (chat + ferramentas) | 7862 |
| Whisper (transcrever/traduzir) | 7861 |
| Agente CrewAI (gera/roda código) | 7860 |
| ComfyUI (imagem/vídeo) | 8188 |

Serviços em Docker:
| Serviço | Porta |
|---|---|
| Open WebUI (chat modelos) | 3001 |
| Loja - Frontend | 3000 |
| Loja - Backend/API | 8000 |
| Loja - pgAdmin | 5050 |

**Portas ocupadas:** 3000, 3001, 5050, 7860, 7861, 7862, 7999, 8000, 8188

## Stacks neste repositório

### frigate/ — Frigate NVR (câmeras)
- Stack isolada. Portas: **8971** (web c/ login), **8554** (RTSP), **8555** (WebRTC) — sem conflito.
- **Frigate 0.18.0-rc1 imagem `-tensorrt`.** A 0.17 travava a detecção na Blackwell
  (CUDA 12.x vs. 12.8+ exigido); suporte oficial à série 50 só na 0.18. Voltar p/ estável
  quando a 0.18 sair final. Detector faz "warm up" (~80s) no 1º boot — normal, não é trava.
- **Detecção na GPU** (RTX 5060 Ti): detector **ONNX** + modelo **YOLOv9-t 320**
  (`config/model_cache/yolov9-t-320.onnx`, gerado no servidor — não vai pro git).
  Decode de vídeo na GPU (`hwaccel_args: preset-nvidia`). VRAM ~1-2 GB.
- GPU compartilhada com o Assistente (ollama ~5,4 GB + python ~2,7 GB); ~8 GB livres.
  Cuidado ao ligar ComfyUI junto (ele quer a placa quase toda).
- Driver do servidor: **595.84 / CUDA 13.2** (suporta Blackwell).
- **Limites:** `mem_limit: 4g`, `cpus: 4.0`, `shm_size: 256mb`, cache tmpfs 256 MB.
- Gravação → variável `FRIGATE_STORAGE` no `.env` (padrão `./storage` local; setar pro NAS depois).
- Formato de gravação 0.17: `record.retain` + `record.alerts` + `record.detections`.

### Câmeras — CareCam Pro (marca HMT, WiFi P2P)
- **App:** CareCam Pro. Firmware `HMT.CM2307`. Várias câmeras (Loja, Loja 5, Loja 2...).
- **RTSP:** usuário `admin`, **senha em branco**. Caminhos:
  - `rtsp://admin:@IP:554/streamtype=0` → principal (2304x1296, **H.265**)
  - `rtsp://admin:@IP:554/streamtype=1` → sub (640x360, **H.265**)
- ⚠️ Esse caminho **só foi descoberto via ONVIF** (porta 8899, GetStreamUri, auth admin/senha-branco).
  Testar caminhos RTSP "no chute" NÃO funciona (a câmera devolve SDP vazio pra qualquer path).
- ⚠️ **H.265** → detecção e gravação OK, mas **ao vivo não toca no Chrome** (só Safari/Edge).
  Ideal: trocar a câmera pra **H.264** no app (mais leve na CPU também).
- Câmera "Loja" = `192.168.15.12`. (rede "J"; IPs por DHCP, podem mudar — considerar IP fixo.)

## Pendências / próximos passos
- [x] Câmera "Loja" configurada no Frigate (RTSP via ONVIF).
- [ ] Descobrir o caminho real do NAS (`df -h`) e setar `FRIGATE_STORAGE` no `.env`.
- [ ] Configurar as outras câmeras (Loja 5, Loja 2...) — mesmo processo ONVIF por IP.
- [ ] (Opcional) Trocar câmeras pra H.264 no app CareCam pra ter ao vivo no Chrome.
- [ ] (Opcional) Fixar IP das câmeras no roteador (DHCP reservation) pra não mudarem.
