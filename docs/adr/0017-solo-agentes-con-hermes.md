# 0017 · Sloth sin interfaz, manejado desde Telegram con Hermes

Estado: propuesta (fase 0 del plan `docs/plan-solo-agentes.md`). Revisado en la documentación oficial de Hermes el
7 de noviembre de 2026; nada se probó aún en la máquina de destino.

## Decisión

- Solo Telegram, solo en el repo, Hermes y Sloth en la misma máquina.
- **Pedir** (Telegram → Sloth): Hermes carga un MCP por stdio, `hermes/sloth-mcp.mjs`, que solo habla con la API local
  de Sloth (`SLOTH_URL`). Hermes llama sus herramientas `mcp_sloth_<herramienta>`.
- **Avisar** (Sloth → Telegram): Sloth hace `POST http://127.0.0.1:8644/webhooks/sloth` a la plataforma Webhook de
  Hermes, en una ruta con `deliver_only: true`: el texto va directo al chat, sin pasar por el modelo (no gasta tokens).
  No hace falta consulta periódica ni la Bot API directa.

## Lo que dice Hermes

MCP, en `~/.hermes/config.yaml` (se aplica con `/reload-mcp`). Solo recibe las variables que se listan en `env`:

```yaml
mcp_servers:
  sloth:
    command: "node"
    args: ["<repo>/hermes/sloth-mcp.mjs"]
    env:
      SLOTH_URL: "http://127.0.0.1:4848"
```

Telegram, en `~/.hermes/.env`: `TELEGRAM_BOT_TOKEN` (de @BotFather) y `TELEGRAM_ALLOWED_USERS` (ids numéricos). Sin
estar en la lista, el gateway rechaza a todos. Nunca `GATEWAY_ALLOW_ALL_USERS=true`: los agentes de Sloth corren con
`bypassPermissions`. Se arranca con `hermes gateway`.

Avisos, en `~/.hermes/.env`: `WEBHOOK_ENABLED=true` (puerto 8644 por defecto), y en `config.yaml`:

```yaml
platforms:
  webhook:
    enabled: true
    extra:
      routes:
        sloth:
          secret: "<SLOTH_WEBHOOK_SECRETO>"
          deliver: "telegram"
          deliver_only: true
          prompt: "{message}"
          deliver_extra:
            chat_id: "<tu id de Telegram>"
```

Cada aviso se firma con `X-Webhook-Timestamp` y `X-Webhook-Signature-V2`, que es HMAC-SHA256 en hex de
`<timestamp>.<cuerpo>` (±300 s). Respuestas: 200 entregado, 401 firma, 404 ruta, 429 más de 30 por minuto, 502
Telegram lo rechazó. Por ese 429, Sloth junta los avisos.

Notas de voz: Hermes las guarda en `~/.hermes/cache/audio/` y las transcribe (`stt`: `local` con faster-whisper,
`groq` u `openai`). En una máquina chica se usa `groq` u `openai`, o `stt.enabled: false`. Para las fotos, la
documentación no dice dónde quedan: se comprueba al instalar.

Instalación: `curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash -s -- --skip-browser --skip-computer-use`.
Usa Python 3.14 y trae su propio Node y FFmpeg. Linux aarch64 tiene soporte de primer nivel; ARMv6/ARMv7 (32 bits), no.
No publica un mínimo de RAM.

## El modelo de Hermes: OpenCode Go

Decisión: Hermes usa la misma suscripción de OpenCode Go que Sloth. Revisado en la documentación el 7 de noviembre de 2026.

- Hermes trae OpenCode Go como proveedor propio: llave `OPENCODE_GO_API_KEY` en `~/.hermes/.env` y `provider: opencode-go`
  (doc de proveedores de Hermes). No hace falta un endpoint personalizado. La llave se copia de la consola de opencode.ai.
- La doc de OpenCode Go lista a Hermes entre los clientes validados, desde la **v0.21.6**, que manda el encabezado
  `x-opencode-session`. Sin esa versión, no. Es lo que dice la página; no garantiza que siga así.
- Condiciones de Hermes para el modelo: **al menos 64 000 tokens de contexto** y **llamadas a herramientas**. Un modelo
  de Go que no cumpla no sirve, aunque sea barato. Sus tareas auxiliares (títulos, compresión) usan el mismo modelo.
- Los límites de Go son por modelo y se comparten con Sloth: 20 % del mes cada 5 horas, 50 % por semana, 100 % al mes.
  Hermes y los agentes de Sloth gastan del mismo cupo. Los avisos por webhook no gastan nada (`deliver_only`).
- Las condiciones dicen que Go es «para OpenCode y otros agentes de código que hacen peticiones parecidas», y que vigilan
  el abuso. Hermes como agente de chat es uso normal; no se usa Go para otra cosa.

Sin confirmar (se comprueba al instalar):
- La URL base. La doc de Hermes dice `/zen/go/v1` y la de Go dice `https://opencode.ai/inference/go/`. Se usa la que
  traiga Hermes y, si falla, se mira `…/v1/models`.
- Qué ids de modelo acepta Hermes bajo `opencode-go`. Se elige con `hermes model` entre los que él enseñe, y se
  comprueba que lleve herramientas y 64 000 de contexto.
- Si Hermes habla con Go por chat/completions o por messages en cada modelo.

## La máquina: Raspberry Pi Zero

Esto choca con lo que Sloth ya sabe de sí mismo:

- La Pi Zero (la primera, la W) es ARMv6 de 32 bits: ni Hermes ni Node ≥ 20 tienen versión oficial para ella. No sirve.
- La Pi Zero 2 W es aarch64 con **512 MB** de RAM. Guías de terceros dan una Pi 4 con 2 GB como lo mínimo práctico para
  Hermes sin navegador, y una Pi 3 (1 GB) como «no realista». Cada proceso de agente ocupa cientos de MB
  (`server/sessions.ts`, `OCIOSO`), y el servidor de Sloth también.
- Los videos no caben: un render de Remotion abre Chrome, Whisper/Parakeet piden GB, y un lote no arranca con menos de
  15 GB libres en disco (`DISCO`) ni 2 GB de memoria libre (`LIBRE`).

Opciones:

1. **Pi Zero 2 W solo como puente:** Hermes y el MCP en la Pi; Sloth en la computadora de siempre, y la Pi le habla por
   Tailscale. Es lo único que funciona con la Pi, pero ya no es «la misma máquina»: hace falta `SLOTH_TOKEN` y quitar el
   control de `Host` para esas rutas (fase 6 del plan).
2. **Todo en la computadora de siempre**, sin interfaz: lo más rápido y lo que este ADR supone.
3. **Una máquina más grande para todo** (Pi 5 de 8 GB o un mini PC): sirve para chats y cards sin video. Los videos
   siguen pidiendo la computadora con GPU.
