# minecraft-skill

OpenClaw skill pack for Minecraft Java Edition — building server operations, game wiki, server admin, and bot bridge.

## Skills

| Skill | Tier | Description | External Dependencies |
|-------|------|-------------|----------------------|
| `minecraft-building-server` | 1 | Build-server workflows, builder automation, large-scale editing, and material pipelines | None |
| `minecraft-wiki` | 1 | Minecraft Java Edition knowledge and reference skill | None |
| `minecraft-bridge` | 2 | Mineflayer-based local bridge for live bot control via REST | `node`, `mineflayer`, `mineflayer-pathfinder`, `vec3` |
| `minecraft-server-admin` | 3 | Remote Minecraft server administration over RCON | `node`, `rcon-client` |

## Installation

Via clawHub:
```bash
clawhub install minecraft-building-server
clawhub install minecraft-wiki
clawhub install minecraft-bridge
clawhub install minecraft-server-admin
```

Or manually copy the skill directory to `~/.openclaw/skills/`.

## Publishing

```bash
clawhub publish ./minecraft-building-server --slug minecraft-building-server --version 1.0.0
clawhub publish ./minecraft-wiki --slug minecraft-wiki --version 1.0.0
clawhub publish ./minecraft-bridge --slug minecraft-bridge --version 1.0.0
clawhub publish ./minecraft-server-admin --slug minecraft-server-admin --version 1.0.0
```

## Security Notes

- **CORS**: `minecraft-bridge/bridge-server.js` binds to `127.0.0.1` but sets `Access-Control-Allow-Origin: *`. This means a malicious webpage in a local browser could make cross-origin requests to the bridge API. Only run the bridge on trusted machines.
- **RCON passwords**: Always use strong passwords for `MC_RCON_PASSWORD` and restrict RCON port access via firewall. Never commit real credentials.
- **Bot authorization**: On public servers, always get admin permission before running any bot.

## Cross-Skill Dependencies

See `dependency-evaluation.md` for analysis of how these skills interact, particularly around the `minecraft-bridge` service layer.

## License

MIT
