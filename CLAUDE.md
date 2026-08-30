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
- **Detecção na CPU** (GPU deixada livre pras outras IAs). Bloco de GPU pronto mas desligado.
- **Limites de recurso definidos:** `mem_limit: 2g`, `cpus: 2.0`, `shm_size: 128mb`, cache tmpfs 256 MB.
- Câmeras: **RTSP** (senha vem do `.env` via `{FRIGATE_RTSP_PASSWORD}`).
- Gravação → pasta do **NAS** (`/mnt/nas/frigate` = placeholder, **confirmar caminho real com `df -h`**).

## Pendências / próximos passos
- [ ] Descobrir o caminho real do NAS no servidor (`df -h`) e ajustar o volume no `frigate/docker-compose.yml`.
- [ ] Preencher `frigate/.env` com a senha RTSP da câmera.
- [ ] Preencher o link RTSP real da(s) câmera(s) em `frigate/config/config.yml` (testar antes no VLC).
- [ ] Marca/modelo das câmeras: (a definir).
