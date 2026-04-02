# Onboarding: Claude Code Usage Monitor

Esta guía explica cómo cada desarrollador del equipo puede activar el monitoreo de uso de Claude Code en su máquina.

**Requisitos previos**: Claude Code instalado, consentimiento explícito del developer.

---

## Paso 1 — Copiar los scripts al sistema

```bash
sudo cp scripts/claude-usage-log.sh /usr/local/bin/claude-usage-log.sh
sudo cp scripts/claude-session-end.sh /usr/local/bin/claude-session-end.sh
sudo chmod +x /usr/local/bin/claude-usage-log.sh /usr/local/bin/claude-session-end.sh
```

---

## Paso 2 — Configurar el directorio de logs

Los logs se escriben en `~/.claude/team-usage/` por defecto.
Para logs compartidos (recomendado en equipos), apunta a una carpeta sincronizada:

```bash
# Opción A: carpeta local (solo para pruebas)
export CLAUDE_USAGE_LOG_DIR="$HOME/.claude/team-usage"

# Opción B: S3 via s3fs (equipo completo)
# Montar el bucket primero y luego:
export CLAUDE_USAGE_LOG_DIR="/mnt/team-claude-logs"

# Agregar al ~/.zshrc o ~/.bashrc para persistencia
echo 'export CLAUDE_USAGE_LOG_DIR="$HOME/.claude/team-usage"' >> ~/.zshrc
```

---

## Paso 3 — Agregar los hooks a Claude Code

Edita `~/.claude/settings.json` y agrega el bloque `hooks`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/claude-usage-log.sh \"$CLAUDE_TOOL_NAME\" \"$CLAUDE_SESSION_ID\""
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/claude-session-end.sh \"$CLAUDE_SESSION_ID\" \"${CLAUDE_TOTAL_COST_USD:-0}\""
          }
        ]
      }
    ]
  }
}
```

Si ya tienes otros hooks configurados, agrega los bloques dentro de los arrays existentes.

---

## Paso 4 — Verificar que funciona

Inicia una sesión de Claude Code, ejecuta cualquier comando, y luego:

```bash
# Debe existir y contener al menos una línea JSON
cat ~/.claude/team-usage/$(date +%Y-%m-%d).jsonl
```

Salida esperada:
```json
{"ts":"2026-03-29T14:05:03Z","tool":"Bash","session":"sess-abc123","user":"sebasing","project":"PluginClaude"}
```

---

## Qué se captura (y qué NO)

| Se captura | No se captura |
|-----------|--------------|
| Nombre de la herramienta (ej. `Bash`, `Edit`) | Contenido de tus prompts |
| ID de sesión anónimo | Código o archivos editados |
| Tu usuario de OS (`$USER`) | Respuestas de Claude |
| Nombre del proyecto (basename del directorio) | Rutas completas de archivos |
| Costo estimado de la sesión (al cerrar) | Ningún dato personal adicional |

---

## Configurar el cron para reportes diarios

En la máquina donde corre el aggregator (o en tu propia máquina si los logs son locales):

```bash
# Editar crontab
crontab -e

# Agregar esta línea (reporte a las 9:00 AM cada día)
0 9 * * * cd /path/to/PluginClaude/monitor && node dist/index.js >> /tmp/claude-monitor.log 2>&1
```

Asegúrate de tener el archivo `monitor/.env` con `DISCORD_WEBHOOK_URL` configurado antes de activar el cron.

---

## Opt-out

Para dejar de enviar datos, simplemente elimina o comenta el bloque `hooks` de `~/.claude/settings.json`.
