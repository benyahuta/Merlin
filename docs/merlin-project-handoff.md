# Merlin Bot — Project Handoff Prompt

## About

Merlin was built by Ben (GitHub: [@benguettlr](https://github.com/benguettlr)), a Duke University student, as a hands-on project to learn cloud deployment, OAuth integrations, and AI agent design. The notes below document the architecture and operations so the project can be picked up and extended.

---

## What is this project?

Merlin is a 24/7 AI health monitoring assistant that runs on AWS and communicates via Telegram. It connects to Ben's Whoop fitness tracker to pull real biometric data (recovery scores, sleep metrics, strain levels, workouts) and acts as a holistic accountability partner for fitness, sleep, nutrition, academics, and overall wellbeing.

**The bot's personality:**
- David Goggins discipline + Andrew Huberman health knowledge + older brother energy
- Gen Z texting style (lowercase, slang: ur, bc, ngl, lowkey, no cap, fs, deadass, ong)
- Short messages (1-2 sentences), minimal punctuation
- Roasts Ben for slacking, hypes him for good choices
- Mirrors Ben's energy — chill when he's chill, full Goggins when he's fired up
- Cares about the whole picture: sleep, recovery, nutrition, mental health, hydration, academics
- Knows Ben goes to Duke and juggles school + fitness
- Learns new slang from Ben over time
- Signature emoji: 🦵
- Bot name: Merlin (@YourBotHandle on Telegram)

---

## Current Infrastructure

### AWS EC2 Instance
- **IP:** <SERVER_IP> (but check — Ben upgraded to a different instance, current is <SERVER_IP>)
- **OS:** Ubuntu 24.04 LTS
- **Instance type:** Was t3.micro, may have been upgraded
- **SSH key:** `your-ssh-key.pem` in Ben's Downloads folder on his Mac
- **SSH command:** `ssh -i ~/Downloads/your-ssh-key.pem ubuntu@<CURRENT_IP>`
- **Region:** us-east-2 (Ohio)

### OpenClaw Platform
- **Version:** 2026.2.9
- **AI Model:** openai/gpt-5.1-codex
- **Channel:** Telegram (was originally WhatsApp, migrated to Telegram)
- **Config directory:** ~/.openclaw/
- **Skills directory:** ~/.openclaw/workspace/skills/
- **Gateway runs as background service** — survives SSH disconnects and laptop closes

### Whoop Integration ✅ WORKING
- **Skill:** whoop-central installed at ~/.openclaw/workspace/skills/whoop-central/
- **Credentials:** ~/.clawdbot/whoop/credentials.json
- **Tokens:** ~/.clawdbot/whoop/token.json
- **Client ID:** <WHOOP_CLIENT_ID>
- **⚠️ Client Secret is exposed** — needs to be regenerated at developer-dashboard.whoop.com
- **Redirect URI (for OAuth via Postman):** https://oauth.pstmn.io/v1/browser-callback
- **Whoop OAuth URLs:**
  - Auth: https://api.prod.whoop.com/oauth/oauth2/auth
  - Token: https://api.prod.whoop.com/oauth/oauth2/token
- **Scopes:** offline read:recovery read:sleep read:workout read:cycles read:profile read:body_measurement
- **Data scripts:**
  - `node src/today.js` — today's recovery, sleep, strain, workouts
  - `node src/recovery.js` — recovery details
  - `node src/sleep.js` — sleep details
  - `node src/strain.js` — strain details
  - `node src/workouts.js` — workout details
  - `node src/summary.js` — summary view
  - `node src/verify.js` — verify auth is working
- **Token refresh:** Access tokens expire every 3600 seconds. The refresh_token should auto-refresh. If auth breaks, re-do the Postman OAuth flow (see below).

### Gmail Watcher
- There's a gmail-watcher component running (seen in gateway logs). Status/config unknown — may need investigation.

---

## Key Commands

```bash
# SSH into server
ssh -i ~/Downloads/your-ssh-key.pem ubuntu@<IP_ADDRESS>

# Start gateway (foreground)
openclaw gateway --force

# Start gateway (background, survives disconnect)
nohup openclaw gateway --force > /dev/null 2>&1 &

# Kill gateway
pkill -f openclaw

# Check status
openclaw status
openclaw doctor

# Check skills
openclaw skills list
openclaw skills info whoop-central

# Install skills
npx clawhub search <query>
npx clawhub install <skill-name>

# Test Whoop data
cd ~/.openclaw/workspace/skills/whoop-central
node src/verify.js
node src/today.js

# View logs
cat /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

---

## How Whoop OAuth Was Solved

The browser-based OAuth flow kept failing with `request_forbidden`. The solution that worked:

1. Add redirect URI `https://oauth.pstmn.io/v1/browser-callback` to the Whoop developer app
2. Use **Postman** desktop app with OAuth 2.0 Authorization Code flow
3. Set "Authorize using browser" checkbox
4. Set Client Authentication to "Send client credentials in body"
5. Complete the flow in browser → get access_token and refresh_token
6. Manually save tokens to `~/.clawdbot/whoop/token.json`
7. Run `node src/verify.js` to confirm

**If tokens expire and refresh fails**, repeat this Postman flow to get new tokens.

---

## What's Working

- ✅ Merlin responds on Telegram 24/7
- ✅ Whoop data pulling (recovery, sleep, strain, workouts)
- ✅ Bot personality configured (Goggins + Huberman + big bro)
- ✅ AWS instance running persistently
- ✅ Gateway auto-restarts

## What's NOT Done Yet / Next Steps

### Priority 1: Reliability
- [ ] Regenerate Whoop Client Secret (exposed in chat)
- [ ] Set up token auto-refresh monitoring (alert if tokens expire)
- [x] Back up project to GitHub: github.com/benguettlr/Merlin

### Priority 2: Automations
- [ ] Daily morning Whoop report (7am ET automatic message with recovery/sleep)
- [ ] Evening activity check (low strain + home all day → gym reminder)
- [ ] Poor recovery → suggest rest day or light workout
- [ ] Sleep reminder if chatting late at night

### Priority 3: New Integrations
- [ ] Google Calendar — enable meeting rescheduling based on recovery
- [ ] Location awareness (geofencing for gym, dorm, library)
- [ ] Academic calendar integration (exam reminders, study scheduling)

### Priority 4: Enhancements
- [ ] Merlin should auto-pull Whoop data when asked naturally (partially working — may need reinforcement)
- [ ] Weekly health trend summaries
- [ ] Workout programming based on recovery scores
- [ ] Nutrition tracking/reminders

---

## Known Issues

1. **Merlin sometimes doesn't know about the Whoop skill** — may need explicit instruction like "run node ~/.openclaw/workspace/skills/whoop-central/src/today.js" or a gateway restart
2. **State dir migration warning** appears on gateway start — cosmetic, doesn't affect function
3. **WhatsApp was abandoned** in favor of Telegram due to connection reliability issues
4. **Mac sleep kills SSH sessions** — always use `nohup` or `tmux` for long-running commands on the server

---

## File Structure on AWS

```
~/.openclaw/                          # Main OpenClaw config
  workspace/
    skills/
      whoop-central/                  # Whoop integration skill
        src/
          setup.js                    # Credential setup
          auth.js                     # OAuth flow
          verify.js                   # Test auth
          today.js                    # Today's data
          recovery.js / sleep.js / strain.js / workouts.js / summary.js
        SKILL.md                      # Skill documentation

~/.clawdbot/                          # Credentials storage
  whoop/
    credentials.json                  # Client ID + Secret
    token.json                        # Access + Refresh tokens
```

---

## Communication Style with Ben

- Be casual and friendly, match his energy
- Give instructions one step at a time
- Have him screenshot when unsure
- Use plain language, avoid jargon
- Celebrate wins enthusiastically
- Be patient — he's learning Terminal/CLI as he goes
- He'll sometimes need to leave for class — make sure long-running processes are backgrounded
