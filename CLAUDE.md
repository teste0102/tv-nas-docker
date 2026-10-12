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

- **GPU:** RTX 5060 Ti, 16 GB de VRAM — **roda UMA IA pesada por vez**. Serviços de IA
  disputam a GPU, então cargas 24/7 (como o Frigate) devem rodar na **CPU**.
- **Driver:** 595.84 / CUDA 13.2 (suporta Blackwell).

### Discos do servidor

| Disco | Device | Tamanho | Montado em | Uso |
|---|---|---|---|---|
| SSD NVMe (sistema) | `/dev/nvme0n1` | 474 GB | `/` | ~88% (~403 GB) — SO, Docker, snaps |
| HDD (sem uso) | `/dev/sdb` | 447 GB | ❌ não montado | — |
| **HDD 1 TB (gravações Frigate)** | `/dev/sdc1` | 916 GB | `/mnt/frigate` | ~24% (~205 GB) |

- O SSD tem muitos **snaps** instalados (Chrome, Firefox, LibreOffice, VS Code, etc.).
- Após reboot, `nvidia-persistenced` pode falhar → rodar `sudo systemctl start nvidia-persistenced` antes do Docker.

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
- Stack isolada. Portas: **8971** (web c/ login), **8554** (RTSP), **8555** (WebRTC).
- **Frigate 0.18.0-rc1 imagem `-tensorrt`.** A 0.17 travava a detecção na Blackwell
  (CUDA 12.x vs. 12.8+ exigido); suporte oficial à série 50 só na 0.18.
- Config em formato **0.18** (`version: 0.18-0`): `live.streams` em vez de `stream_name`,
  `record.continuous/motion` em vez de `record.retain`.
- **Detecção na GPU** (RTX 5060 Ti): detector **ONNX** + modelo **YOLOv9-t 320**
  (`config/model_cache/yolov9-t-320.onnx`, gerado no servidor — não vai pro git).
  Decode de vídeo na GPU (`hwaccel_args: preset-nvidia`). VRAM ~1-2 GB.
- GPU compartilhada com o Assistente (ollama ~5,4 GB + python ~2,7 GB); ~8 GB livres.
  Cuidado ao ligar ComfyUI junto (ele quer a placa quase toda).
- **Limites:** `mem_limit: 4g`, `cpus: 4.0`, `shm_size: 256mb`, cache tmpfs 256 MB.
- **Gravação:** `FRIGATE_STORAGE=/mnt/frigate` no `.env` → disco de 1 TB (`/dev/sdc1`).
  - `recordings/` ~80 GB/dia com 5 câmeras. Retenção: motion 3 dias, alertas 14 dias.
  - O auto-cleanup do Frigate **pode falhar** — monitorar se gravações >3 dias acumulam.
  - Pasta antiga `~/tv-nas-docker/frigate/storage/` já foi limpa (era 16 GB no SSD).
- MQTT desabilitado.

### Câmeras — CareCam Pro (marca HMT, WiFi P2P)
- **App:** CareCam Pro. Firmware `HMT.CM2307`. **5 câmeras configuradas.**
- **RTSP:** usuário `admin`, **senha em branco**. Caminhos:
  - `rtsp://admin:@IP:554/streamtype=0` → principal (2304x1296, **H.265**)
  - `rtsp://admin:@IP:554/streamtype=1` → sub (640x360, **H.265**)
- ⚠️ Caminho RTSP descoberto **só via ONVIF** (porta 8899, GetStreamUri, admin/senha-branco).
- ⚠️ **H.265** → detecção e gravação OK, mas **ao vivo não toca no Chrome** (só Safari/Edge).
  Solução: go2rtc converte substream pra H.264 (`ffmpeg:*_sub#video=h264`) só p/ live.

#### Câmeras configuradas

| Nome (UI) | Chave config | IP | MAC | Status |
|---|---|---|---|---|
| Loja | `loja` | 192.168.15.12 | `a0:ac:78:21:03:d8` | ✅ |
| Loja 5 | `loja5` | 192.168.15.13 | `a0:ac:78:1f:cb:3c` | ✅ |
| Loja 3 | `loja_3` | 192.168.15.19 | `a0:ac:78:21:70:cc` | ✅ |
| Loja 4 | `loja_4` | 192.168.15.10 | `a0:ac:78:20:f5:42` | ✅ |
| Loja 2 | `loja_2` | 192.168.15.14 | `a0:ac:78:23:dd:8e` | ✅ (nova) |

- IPs são **DHCP** e podem mudar. Todas as CareCam têm MAC prefixo `a0:ac:78:*`.
- Pra achar câmera por MAC: `ip neigh | grep -i "a0:ac:78"` (no servidor).
- Cada câmera tem 3 streams no go2rtc: `nome` (principal), `nome_sub` (detecção), `nome_sub_h264` (live).
- Chave com `_` no nome (ex.: `loja_4`) faz o Frigate exibir "Loja 4" na interface.

#### Adicionar câmera nova (receita)
1. Descobrir IP: `ip neigh | grep -i "a0:ac:78"` ou app CareCam Pro.
2. Adicionar 3 streams no `go2rtc.streams` do `config.yml` (principal, sub, sub_h264).
3. Adicionar bloco em `cameras` (detect 640x360 fps 5, record, live com sub_h264).
4. **NÃO mexer** nas câmeras que já funcionam.
5. `docker compose -f frigate/docker-compose.yml restart` no servidor.

## Pendências / próximos passos
- [x] Câmera "Loja" configurada no Frigate (RTSP via ONVIF).
- [x] `FRIGATE_STORAGE=/mnt/frigate` configurado no `.env` (disco de 1 TB).
- [x] Câmeras Loja 5, Loja 3, Loja 4 e Loja 2 adicionadas.
- [x] Gravações antigas limpas (SSD e disco 1 TB).
- [ ] Fixar IP das câmeras no roteador (DHCP reservation) pra não mudarem.
- [ ] (Opcional) Trocar câmeras pra H.264 no app CareCam pra ter ao vivo no Chrome.
- [ ] Monitorar auto-cleanup do Frigate (gravações >3 dias devem ser apagadas automaticamente).
- [ ] Disco `sdb` (447 GB) não está montado — avaliar uso futuro.
