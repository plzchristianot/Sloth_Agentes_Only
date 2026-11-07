# Plan: Sloth en modo solo agentes, manejado desde Hermes (Telegram o WhatsApp)

Estado: propuesta. Nada de esto está hecho todavía.

## Qué se quiere

Usar Sloth sin abrir su ventana: el servidor corre solo (en esta compu o en un servidor), y lo manejas escribiendo por
Telegram o WhatsApp a [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research). Hermes entiende lo
que pides, llama a Sloth y te contesta en el chat. Ejemplos:

- «Hazme un El Diff de https://…» → crea el hilo con la card correcta y te dice cuando acaba, con el video.
- «¿Cómo van los videos?» → la lista corta de hilos trabajando, esperando o con error.
- «Dile al de los dinosaurios que acorte la intro» → manda el mensaje a ese hilo.
- El agente pregunta algo → te llega al chat, y tu respuesta vuelve al hilo.

## Lo que ya tenemos y sirve tal cual

- Toda la app ya se maneja por HTTP: `server/index.ts` expone `/api/*` (crear hilo `POST /api/sessions`, lote
  `POST /api/lotes`, mandar `POST /api/sessions/<id>/send`, parar `/stop`, empezar `/empezar`, archivar, compactar,
  historial…). La interfaz es solo un cliente más.
- `peticionDeConfianza` (`server/origen.ts`) ya deja pasar a procesos locales sin `Origin` (curl, scripts, MCP): un MCP
  que corra en la misma máquina no necesita cambios en la seguridad.
- `scripts/start.mjs --no-open` ya levanta el servidor sin ventana y lo vuelve a levantar si se cae.
- Los agentes ya trabajan sin que nadie mire: `permissionMode: 'bypassPermissions'` en `server/claude.ts`, los lotes
  arrancan solos y el navegador de un hilo cae a Chrome sin ventana (`server/navegador-agente.ts`).
- Ya hay un patrón de MCP sin paquetes que solo habla con `SLOTH_URL`: `server/voz/mcp.mjs` y `server/flow/mcp.mjs`.
- `scripts/hilo.mjs` ya sabe resumir un hilo (turnos, costo, errores, fondo).
- `server/redactar.ts` ya quita secretos de los textos.

## Arquitectura

```
Telegram / WhatsApp
        │
  Hermes (gateway + su LLM, con allowlist de usuarios)
        │  MCP por stdio
  hermes/sloth-mcp.mjs  ──HTTP──▶  Sloth sin interfaz (127.0.0.1:$PORT)  ──▶  Claude / Codex / OpenCode…
                                          │
                                   server/hermes/avisos.ts  ──▶  Hermes (webhook) o Telegram directo
```

Dos sentidos:

1. **Pedir** (chat → Sloth): Hermes llama herramientas de un MCP `sloth`. Es lo mismo que hace hoy la interfaz.
2. **Avisar** (Sloth → chat): cuando un turno acaba, falla, pregunta algo o sale un video, Sloth avisa sin que nadie
   pregunte. Esto es lo nuevo y lo que más hay que comprobar en Hermes (fase 0).

Hermes no reemplaza a los agentes de Sloth: es solo el mensajero. El trabajo lo siguen haciendo las skills y los
agentes de cada card, con sus límites de máquina (`shared/limites.ts`, `CABEN`).

## Fases

### Fase 0 · Comprobar Hermes (½ día)

Antes de programar, en una máquina de prueba:

- Instalar Hermes y conectar el gateway de Telegram (token de BotFather y la lista de usuarios permitidos) y el de
  WhatsApp (su puente, con QR). Anotar qué versión y cómo se limita a tu número o tu usuario.
- Confirmar cómo se registra un MCP por stdio en su configuración (`~/.hermes/config.yaml`) y que pasa variables de
  entorno (`SLOTH_URL`).
- Confirmar cómo **empujar** un mensaje a un chat desde fuera (webhook de entrada, API del gateway o una tarea
  programada suya). De esto depende la fase 4:
  - Si Hermes acepta un webhook o expone un endpoint para mandar: Sloth avisa por ahí, y vale igual para Telegram y
    WhatsApp.
  - Si no: Sloth manda los avisos de Telegram directo con la Bot API (mandar con el mismo bot no choca con Hermes, que
    es quien lee), y en WhatsApp se cae a una tarea de Hermes que pregunta `sloth_novedades` cada pocos minutos.
- Confirmar dónde deja Hermes las fotos, audios y notas de voz que llegan, y si transcribe las notas de voz.
- Medir el límite de archivos que se pueden devolver (Telegram Bot API: 50 MB; WhatsApp: menos). Un final de El Diff
  pesa cerca de 1 GB: hace falta una versión ligera para el chat (fase 3).

Sale: un ADR nuevo (`docs/adr/0017-solo-agentes-con-hermes.md`) con lo elegido.

### Fase 1 · Servidor sin interfaz (1 día)

- `SLOTH_SIN_INTERFAZ=1` (o `--sin-interfaz`) en `server/index.ts`: no crea Vite ni sirve `web/`; `/` contesta un JSON
  de estado. Ahorra memoria y arranca más rápido. Todo lo demás (API, sesiones, redes, radar) igual.
- `npm run agentes`: `scripts/start.mjs --no-open` con esa variable. Ya relanza tras una caída o un «reiniciar».
- Revisar lo que da por hecho una ventana:
  - El navegador de los hilos ya cae a Chrome sin ventana. En Linux sin escritorio, forzar `headless`.
  - Voz (AI Studio), Flow y TikTok usan un Edge con su perfil. Funcionan sin ventana **si la sesión de Google/TikTok
    ya está iniciada**; el inicio de sesión pide ventana. Se hace una vez con la interfaz, o desde la misma
    compu con `npm start`. Documentarlo; no se intenta iniciar sesión por chat.
  - Conectar Claude, Codex u OpenCode (`/api/*/conectar`) abre un navegador. Igual: se conecta una vez con la
    interfaz o con la CLI del agente (`claude login`). Con llave de API (`server/acceso.ts`) no hace falta.
- Servicio: unidad de systemd (Linux), launchd (Mac) y tarea programada (Windows) que corran `npm run agentes` al
  encender (ADR 0006: lo de un sistema lleva su par en los otros). Van en `scripts/servicio/` con un LEEME corto.

### Fase 2 · Lo que la interfaz decide, pasa al servidor (1–2 días)

Hoy la interfaz arma parte del pedido. Sin ella, Sloth tiene que poder hacerlo solo:

- **Modelo y esfuerzo**: `web/src/Modelos.tsx` elige el «Optimizado» de la card (`card.modelos`) o el último elegido a
  mano. Pasar esa elección a `shared/` (p. ej. `shared/modelos.ts`) para que `POST /api/sessions` acepte `model`
  vacío y elija el mismo que elegiría la interfaz. La interfaz usa la misma función.
- **Líneas del compositor**: `Formato:`, `Voz:`, `Imágenes:`, `Clips:`, `Subagentes:` (hoy en `web/src/App.tsx`,
  `Flow.tsx`, `Modelos.tsx`). Juntarlas en `shared/compositor.ts`, con valores por defecto (los últimos usados, que
  se guardan en `data/` con `guardar()`), y que el servidor las ponga cuando no vienen.
- **Confirmaciones**: el 409 `{ confirmar: true }` de `pideConfirmar` (más de dos videos) ya es una respuesta clara;
  el MCP la devuelve como texto para que Hermes pregunte «¿sigo?» y reenvíe con `masDeDos: true`.

### Fase 3 · Una API corta para el chat (1–2 días)

Las respuestas de hoy son grandes (historial entero, todas las cards). Un chat necesita poco texto. Nuevo módulo
`server/hermes/index.ts` con el mismo patrón que `redes`/`ideas` (`api(req, res, url)` y su línea en `index.ts`):

| Ruta | Qué devuelve |
| --- | --- |
| `GET /api/chat/estilos` | id, título y una línea de descripción de cada card; si hace lotes y qué pide |
| `GET /api/chat/hilos?filtro=activos\|esperan\|error` | id corto, título, card, estado, fase de la etapa, costo |
| `GET /api/chat/hilos/<id>` | el último texto del agente, la pregunta pendiente si la hay, etapas, archivos finales |
| `GET /api/chat/novedades?desde=<t>` | lo que cambió desde `t` (para la consulta periódica de la fase 0) |
| `GET /api/chat/archivo?ruta=` | un archivo final del hilo; solo dentro de su carpeta de trabajo |
| `POST /api/chat/ligero` | una versión para el chat de un video (720p, que quepa en el límite) con FFmpeg |

- El resumen de un hilo sale de la lógica de `scripts/hilo.mjs`, movida a `server/` para usarla en los dos.
- La pregunta pendiente: el último turno que acabó con `AskUserQuestion`/`request_user_input` o en «?» (lo mismo que
  ya mira `PREGUNTAS` en `server/sessions.ts`). Con el texto de las opciones, para que Hermes las ofrezca.
- Ids cortos: los 4–6 primeros caracteres o el título, para poder escribir «el de los dinosaurios».
- Todo lo que sale pasa por `server/redactar.ts`.

### Fase 4 · El MCP `sloth` y la skill para Hermes (1–2 días)

`hermes/sloth-mcp.mjs`, JavaScript sin paquetes como `server/voz/mcp.mjs`, que solo habla con `SLOTH_URL`:

| Herramienta | Llama a |
| --- | --- |
| `sloth_estilos` | `GET /api/chat/estilos` |
| `sloth_hilos` | `GET /api/chat/hilos` |
| `sloth_ver_hilo` | `GET /api/chat/hilos/<id>` |
| `sloth_nuevo` (card, texto, adjuntos, formato…) | `POST /api/upload` + `POST /api/sessions` |
| `sloth_lote` (card, enlaces) | `POST /api/lotes` |
| `sloth_mandar` (hilo, texto, adjuntos) | `POST /api/sessions/<id>/send` |
| `sloth_parar` / `sloth_empezar` / `sloth_archivar` | las rutas que ya existen |
| `sloth_video` (hilo) | `/api/chat/ligero` y devuelve la ruta para que Hermes la mande |
| `sloth_novedades` | `GET /api/chat/novedades` |
| `sloth_redes` (mirar la cola) | `/api/redes/*`, solo lectura al principio |

- Programar en redes queda fuera de la primera versión: `docs/agents/redes.md` pide enseñar la lista y esperar el sí,
  y por chat es fácil equivocarse. Se añade después, con confirmación explícita.
- `hermes/SKILL.md`: la skill que se instala en Hermes. Dice cuándo usar cada herramienta, que confirme antes de lo
  que gasta (lotes, más de dos videos, parar), que conteste corto, que recuerde «el hilo de esta conversación» y que
  una nota de voz se manda transcrita (o como adjunto con `wav`, si Hermes no transcribe).
- `hermes/LEEME.md`: cómo instalar Hermes, el bloque de `config.yaml` para el MCP y la allowlist.
- Va en `hermes/`, no en `skills/`: las de `skills/` son de los agentes de Sloth y las revisa `comprobar-skills`.

### Fase 5 · Avisos al chat (1–2 días)

`server/hermes/avisos.ts`, dentro del servidor, escuchando lo mismo que el WebSocket (`eventosEnVivo` y el aviso de
sesión en `server/index.ts`), no por `/ws`:

- Avisa de: turno acabado bien (con la última línea del agente), turno con error o caída sin retomar, pregunta
  pendiente, lote terminado o pausado (`Lote.pausa`), video final listo (etapa «Video»), fallo de redes (`AvisoRedes`).
- No avisa de cada turno de un lote: un resumen al acabar el lote. Un hilo no avisa más de una vez por minuto.
- Por dónde, según la fase 0: webhook de Hermes, o Bot API de Telegram, o nada (Hermes consulta `sloth_novedades`).
- Configuración en `data/hermes.json` (con `guardar()`): canal, chat y qué avisar. Los tokens, en las variables del
  usuario como las demás llaves (`server/opciones.ts`), nunca en el repo.

### Fase 6 · Seguridad (½ día, pero no se salta)

Un mensaje de chat termina en un agente con `bypassPermissions`: es una terminal remota en tu compu.

- Allowlist en Hermes (solo tu usuario de Telegram y tu número de WhatsApp). Sin ella, no se arranca: el LEEME lo
  pone como primer paso y el MCP lo comprueba al iniciar si Hermes lo deja saber.
- El puerto de Sloth sigue en `127.0.0.1`. Si Hermes corre en otra máquina: túnel SSH o Tailscale, nunca el puerto
  abierto; y entonces `/api/chat/*` pide `Authorization: Bearer <SLOTH_TOKEN>`, porque `Host` ya no será local.
- `/api/chat/archivo` solo sirve archivos de las carpetas de trabajo de los hilos (sin `..`, con `realpath`).
- Registro de lo que llegó por chat en `data/hermes/registro.jsonl`: quién, cuándo, qué herramienta.
- Las opciones peligrosas (borrar hilos, cards, reiniciar, instalar) no tienen herramienta.

### Fase 7 · Pruebas, documentación y venta (1 día)

- Tests (`node --import tsx --test`): la elección de modelo y las líneas del compositor en `shared/`, el resumen de un
  hilo, los filtros y el ritmo de los avisos, las rutas de `/api/chat` (incluido el `..` de `archivo`), y el MCP con
  un servidor de mentira, como `server/voz-mcp.test.ts`.
- Prueba de punta a punta con un turno grabado (como `server/opencode.grabacion.json`): crear hilo por MCP, recibir el
  aviso, contestar la pregunta.
- `npm run typecheck` y `npm test`.
- Documentación: entrada en `docs/agents/mapa.md`, guía `docs/agents/hermes.md`, el ADR 0017 y una línea en
  `AGENTS.md` (quitando otra: el test lo para en 1.000 palabras).
- Venta: primero solo en el repo. Si sirve, la app que se vende puede traerlo como opción en Opciones, con `VENTA`
  decidiendo qué se ve (`shared/venta.ts`) y su par en Windows, Mac y Linux.

## Orden y tamaño

| Fase | Días | Depende de |
| --- | --- | --- |
| 0 · Comprobar Hermes | ½ | — |
| 1 · Servidor sin interfaz | 1 | — |
| 2 · Decisiones al servidor | 1–2 | 1 |
| 3 · API para el chat | 1–2 | 2 |
| 4 · MCP y skill de Hermes | 1–2 | 3 |
| 5 · Avisos | 1–2 | 0, 3 |
| 6 · Seguridad | ½ | 3, 4 |
| 7 · Pruebas y docs | 1 | todas |

Unos 7–10 días. La primera versión que ya sirve (fases 0–4 y 6, sin avisos: preguntas tú «¿cómo va?») son 4–6 días.

## Lo que queda fuera

- El estudio de video (`Estudio.tsx`, `Editor.tsx`): editar por clics en el cuadro o la línea de tiempo no tiene
  sentido por chat. Se piden los cambios con palabras, como en el chat de la derecha.
- Ideas, Modelos 3D, la comunidad y Opciones: siguen en la interfaz.
- Iniciar sesión en Google, TikTok o los agentes: una vez, con la interfaz.

## Decisiones (7 de noviembre de 2026)

- Solo Telegram, solo en el repo, Hermes y Sloth en la misma máquina. Fuera: WhatsApp, `VENTA` y los servicios de
  Windows y Mac.
- Fase 0 hecha: `docs/adr/0017-solo-agentes-con-hermes.md`. Los avisos van por la plataforma Webhook de Hermes con
  `deliver_only` (sin gastar tokens): la fase 5 se reduce a un `fetch` firmado. La consulta periódica y la Bot API
  directa ya no hacen falta.
- Pendiente: la máquina. La Pi Zero no aguanta Sloth (ver el ADR); hay que elegir entre usarla solo de puente o correr
  todo en otra máquina.
- Versión rápida: fases 1, 2 (solo modelo por defecto), 3, 4 y 5 juntas, sin `ligero` ni redes.
- Agente por defecto: OpenCode con la suscripción Go (`server/opencode.ts`, `server/cuenta-opencode.ts`, ADR 0010). La
  cuenta se conecta una vez por el código de dispositivo, sin navegador en el servidor: `POST /api/opencode/conectar`
  devuelve una URL y un código, se confirman desde el celular, y `GET /api/opencode/cuenta` dice cuándo quedó. Sus
  modelos salen como `opencode:opencode-go/<id>` en `/api/state` → `models`. Un solo `opencode serve` atiende todos los
  hilos, así que gasta menos memoria que un proceso por hilo (Claude Code). La fase 2 (modelo por defecto) toma el
  primero de Go; los límites de Go (5 horas, semana, mes) ya salen en `GET /api/opencode/limites`.
