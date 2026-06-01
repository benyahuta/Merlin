# Merlin — AI Health Coach Bot

Merlin is a 24/7 AI health monitoring assistant running on AWS, communicating via Telegram (@YourBotHandle). It connects to Ben's Whoop fitness tracker for real biometric data and acts as a holistic accountability partner.

## Where Things Live

### On AWS (the actual running bot)
- **Server IP**: Check latest — was <SERVER_IP>, may have changed
- **Region**: us-east-2 (Ohio)
- **OpenClaw config**: `~/.openclaw/`
- **Skills**: `~/.openclaw/workspace/skills/` (includes whoop-central)
- **Personality**: `~/.openclaw/workspace/SOUL.md` and `IDENTITY.md`
- **Whoop tokens (server)**: `~/.clawdbot/whoop/`
- **Logs**: `/tmp/openclaw/openclaw-$(date +%Y-%m-%d).log`

### On Mac (local config & credentials)
- **Whoop credentials**: `~/.clawdbot/whoop/credentials.json`
- **Whoop tokens**: `~/.clawdbot/whoop/token.json`
- **Whoop TLS certs**: `~/.clawdbot/whoop/tls.crt` and `tls.key`
- **Clawdbot env**: `~/.clawdbot/.env`
- **SSH key**: `~/openclaw-projects/shared/your-ssh-key.pem`

### In This Folder
- `docs/merlin-project-handoff.md` — Full project context, infrastructure, and known issues
- `docs/merlin-proactive-triggers.md` — Spec for proactive messaging system (not yet built)

## Quick SSH Access
```bash
ssh -i ~/openclaw-projects/shared/your-ssh-key.pem ubuntu@<IP_ADDRESS>
```

## Key Commands (on server)
```bash
openclaw gateway --force          # Start gateway
openclaw status                   # Check status
openclaw skills list              # List installed skills
cd ~/.openclaw/workspace/skills/whoop-central && node src/today.js   # Today's Whoop data
```

## Status
- Bot is live on Telegram 24/7
- Whoop integration working (tokens may need refresh via Postman OAuth flow)
- Proactive triggers system not yet built (see docs/)
- Source backed up to GitHub: [github.com/benguettlr/Merlin](https://github.com/benguettlr/Merlin)
