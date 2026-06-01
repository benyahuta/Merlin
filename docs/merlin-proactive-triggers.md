# Merlin Proactive Messaging System — Claude Code Implementation Spec

## What This Is

Instructions for building a proactive messaging system for Merlin, an AI health coach that runs on OpenClaw on AWS and communicates via Telegram. Right now Merlin only responds when Ben messages him. We want Merlin to initiate conversations based on biometric data, calendar events, time of day, and behavioral patterns — like a friend who's actually paying attention.

---

## Current Infrastructure (What You're Working With)

### Server
- **AWS EC2** running Ubuntu 24.04 LTS
- SSH access: `ssh -i ~/Downloads/your-ssh-key.pem ubuntu@<IP>`
- Gateway runs in background via `nohup openclaw gateway --force`

### OpenClaw Setup
- **Version:** 2026.2.9
- **AI Model:** openai/gpt-5.1-codex
- **Channel:** Telegram (`@YourBotHandle`)
- **Config dir:** `~/.openclaw/`
- **Skills dir:** `~/.openclaw/workspace/skills/`
- **Personality files:** `SOUL.md` and `IDENTITY.md` in the workspace (these define Merlin's voice — edits require gateway restart)

### Whoop Integration (Already Working)
- **Skill location:** `~/.openclaw/workspace/skills/whoop-central/`
- **Credentials:** `~/.clawdbot/whoop/credentials.json`
- **Tokens:** `~/.clawdbot/whoop/token.json`
- **Data scripts (all in `whoop-central/src/`):**
  - `today.js` — today's recovery, sleep, strain, workouts
  - `recovery.js` — recovery details
  - `sleep.js` — sleep details
  - `strain.js` — strain details
  - `workouts.js` — workout details
  - `summary.js` — summary view
  - `verify.js` — verify auth is working
- **Token refresh:** Access tokens expire every 3600 seconds; refresh token rotates and must be saved

### Google Integrations (Already Working)
- Gmail watcher is functional
- Google Calendar access is set up
- Duke Outlook calendar may be synced via Google Calendar

### Messaging
- **Send messages via CLI:** `openclaw message send --channel telegram --target <chat_id> --message "text"`
- **Cron scheduling:** `openclaw cron` (built-in scheduler)
- Ben's Telegram chat ID is needed — check existing gateway config or logs

### Merlin's Personality (Do NOT Change)
- David Goggins discipline + Andrew Huberman health knowledge + older brother energy
- Gen Z texting style: lowercase, slang (ur, bc, ngl, lowkey, no cap, fs, deadass, ong)
- Short messages (1-2 sentences), minimal punctuation
- Roasts Ben for slacking, hypes him for good choices
- Signature emoji: 🦵
- Knows Ben goes to Duke, Eastern timezone

---

## What To Build

A new OpenClaw skill called `proactive-triggers` that polls data sources on intervals and sends Merlin-style messages to Ben via Telegram when trigger conditions are met.

### Architecture

```
~/.openclaw/workspace/skills/proactive-triggers/
├── SKILL.md                    # Skill documentation
├── src/
│   ├── index.js                # Main entry point — runs the polling loop
│   ├── triggers/
│   │   ├── strain-spike.js     # Detects sudden strain jumps
│   │   ├── recovery-morning.js # Morning recovery check-in
│   │   ├── sleep-anomaly.js    # Detects unusual sleep patterns
│   │   ├── rest-day.js         # Multi-day strain accumulation warning
│   │   ├── bedtime-nudge.js    # Late night still-awake detection
│   │   ├── weekly-trends.js    # Weekly pattern analysis (runs Sundays)
│   │   ├── streak-tracker.js   # Habit streak recognition
│   │   ├── correlation.js      # Pattern discovery across data sources
│   │   ├── calendar-energy.js  # Pre-meeting energy check
│   │   ├── dead-time.js        # Calendar gap detection
│   │   ├── post-event.js       # Post-exam/meeting decompression
│   │   ├── random-checkin.js   # Occasional unprompted check-ins
│   │   └── milestone.js        # PR / personal best celebrations
│   ├── state.js                # Tracks what's been sent to avoid spam
│   ├── whoop-poller.js         # Wraps whoop-central data fetching
│   ├── calendar-poller.js      # Wraps Google Calendar data fetching
│   ├── messenger.js            # Sends messages via openclaw CLI
│   └── utils.js                # Time helpers, randomness, formatting
├── data/
│   └── state.json              # Persistent state (last sent times, streaks, baselines)
└── package.json
```

### Core Components

#### 1. State Manager (`state.js`)
Prevents spam by tracking:
- Last message sent per trigger type (with cooldown periods)
- Historical baselines (rolling averages for strain, sleep, HRV)
- Streak counters (consecutive days of good sleep, high strain, etc.)
- Daily message count (cap at ~4-5 proactive messages per day max)

```javascript
// Example state structure
{
  "lastSent": {
    "strain-spike": "2026-02-13T14:30:00Z",
    "recovery-morning": "2026-02-13T07:15:00Z",
    // ...
  },
  "baselines": {
    "avgStrain": 12.5,
    "avgSleep": 7.2,
    "avgHRV": 65,
    "avgRecovery": 58
  },
  "streaks": {
    "sleep7plus": 4,       // consecutive days of 7+ hrs sleep
    "highStrain": 3        // consecutive days of strain > 14
  },
  "dailyMessageCount": 2,
  "dailyMessageDate": "2026-02-13"
}
```

#### 2. Whoop Poller (`whoop-poller.js`)
Wraps the existing whoop-central scripts:
```javascript
// Calls node ~/.openclaw/workspace/skills/whoop-central/src/today.js
// Parses the output into structured data
// Returns: { recovery, sleep, strain, hrv, workouts }
```

#### 3. Messenger (`messenger.js`)
Sends messages with slight timing randomness to feel human:
```javascript
// Adds 0-5 minute random delay before sending
// Uses: openclaw message send --channel telegram --target <chat_id> --message "text"
// Respects quiet hours (11pm - 7am ET, except bedtime nudge)
```

#### 4. Message Generation
Each trigger should generate messages by calling the OpenClaw agent with a prompt, NOT by using hardcoded templates. This way Merlin's personality stays consistent and messages feel natural.

```javascript
// Use openclaw agent to generate the message in Merlin's voice:
// openclaw agent --message "Ben just finished a workout. His strain jumped from 8 to 16. 
//   Send him a short hype message asking what he did, then give 2-3 recovery tips. 
//   Keep it casual, Gen Z style, 2-3 short texts max." --deliver
```

Alternatively, if direct agent invocation is complex, use a simple prompt template that gets passed to the model via the gateway.

---

## Trigger Specifications

### Priority 1 — Build These First

#### Strain Spike Detection
- **Poll interval:** Every 15 minutes
- **Condition:** Current strain > previous reading by 4+ points (or strain > 15 when last reading was < 10)
- **Cooldown:** 4 hours (don't alert on the same workout twice)
- **Message flow:**
  1. Hype message + ask what the workout was
  2. If Ben replies, give recovery recommendations based on strain level and current recovery
- **Example:** "bro you just went crazy 🦵 what was the workout??" → then: "ok after that you need like 3L of water minimum and try to get 8+ hours tonight. your body's gonna be recovering hard"

#### Recovery Score Morning Check-in
- **Poll interval:** Once daily, between 7:00-7:30 AM ET (randomized)
- **Condition:** Always fires (this is the daily touchpoint)
- **Message varies by recovery score:**
  - Green (67-100%): Hype message, suggest it's a good day to push hard
  - Yellow (34-66%): Balanced message, suggest moderate activity
  - Red (0-33%): Concerned message, suggest rest/mobility, offer to adjust schedule
- **Include:** Recovery %, HRV, hours of sleep, strain from yesterday
- **Example (red):** "dawg 23% recovery and 4.5 hrs of sleep?? nah we're taking it easy today. mobility only, drink water, and maybe push that afternoon thing back if you can 🦵"

#### Sleep Anomaly Detection
- **Poll interval:** Once daily, runs after morning check-in data is available
- **Condition:** Sleep duration deviates >1.5 hours from 7-day rolling average OR HRV drops >15% from baseline
- **Cooldown:** 24 hours
- **Example (bad):** "4 hours?? what happened last night bro"
- **Example (great):** "9.2 hours and your HRV is through the roof. today's the day to PR something ngl 🦵"

#### Bedtime Nudge
- **Poll interval:** Check at 11:30 PM, 12:00 AM, 12:30 AM ET
- **Condition:** No sleep onset detected AND it's a weeknight (Sun-Thu)
- **Cooldown:** Once per night
- **Example:** "it's 12:30 and you're still up. whatever it is it'll be there tomorrow. go to bed 🦵"

### Priority 2 — Build After Core Works

#### Rest Day Enforcement
- **Condition:** 3+ consecutive days with strain > 14 AND recovery trending down
- **Cooldown:** 48 hours
- **Example:** "you've been going hard for 4 days straight and your recovery is tanking. take the L today so you can win tomorrow 🦵"

#### Habit Streak Recognition
- **Condition:** 7-day streak of sleep > 7 hours, OR 5-day streak of recovery > 60%
- **Also triggers on streak break:** 3+ day streak broken
- **Example (streak):** "that's 7 days straight of 7+ hours sleep. you're building something. don't break the chain 🦵"
- **Example (broken):** "streak's dead at 5 days. 6.2 hours last night. get back on it tonight"

#### Weekly Trends (Sunday Evening)
- **Schedule:** Sunday 7-8 PM ET
- **Content:** Compare this week vs last week on sleep avg, strain avg, recovery avg, HRV trend
- **Example:** "weekly check 📊 this week vs last: slept 45 min less on average but strain was higher. that math doesn't work long term bro"

#### Post-Event Decompression
- **Condition:** A calendar event tagged as important (exam, presentation, meeting with specific keywords) just ended
- **Cooldown:** 2 hours per event
- **Example:** "that exam is done. how'd it go? go decompress you earned it 🦵"

### Priority 3 — Polish Layer

#### Calendar Energy Check
- **Condition:** Important calendar event in 1-2 hours AND current recovery is red/yellow
- **Example:** "you've got that pitch in 2 hours and your HRV says you're stressed. go walk for 10 minutes"

#### Dead Time Detection
- **Condition:** 2+ hour gap in calendar with no events
- **Cooldown:** Once per day
- **Example:** "you've got nothing from 2-4. that's your window. use it"

#### Correlation Discovery
- **Condition:** After 2+ weeks of data, detect patterns (e.g., better HRV on days with no early meetings)
- **Schedule:** Every 2 weeks
- **Example:** "i've noticed your HRV is always better on days you don't have anything before 10am. maybe protect your mornings?"

#### Random Check-ins
- **Frequency:** 2-3 times per week, random timing during waking hours
- **No data trigger** — just vibes
- **Example:** "what's the move today?" / "you've been quiet, you good?" / "what are we eating today"

#### Milestone Celebrations
- **Condition:** New personal best in any tracked metric
- **Example:** "new HRV record. whatever you did yesterday, do it again 🦵"

---

## Key Implementation Details

### Humanizing the Timing
- **Never send at exact intervals.** Add random jitter of 1-10 minutes to all scheduled messages.
- **Vary message timing day to day.** Morning check-in should land somewhere between 7:00-7:45 AM, not 7:00 sharp every day.
- **Skip messages occasionally.** Maybe 1 in 5 random check-ins just... doesn't fire. Real friends don't text on a schedule.
- **Respect quiet hours:** No messages 11 PM - 7 AM except the bedtime nudge (which is the whole point).

### Daily Message Cap
- **Max 4-5 proactive messages per day.** If multiple triggers fire, prioritize:
  1. Morning check-in (always)
  2. Strain spike / sleep anomaly (health-critical)
  3. Bedtime nudge (if applicable)
  4. Everything else competes for remaining slots

### Message Style Rules
- **1-3 short texts per trigger**, not a wall of text
- **Ask questions** — don't just broadcast data. "what was the workout?" not "your strain increased"
- **Use data to support, not lead.** "you look cooked today" hits different than "your recovery score is 23%"
- **Rotate phrasing** — don't use the same opener twice in a week

### Gateway Restart
After installing the new skill and any config changes:
```bash
openclaw gateway --force &
```
Or if running in tmux/nohup:
```bash
kill $(pgrep -f openclaw-gateway) && nohup openclaw gateway --force > /dev/null 2>&1 &
```

---

## What Ben Needs To Provide / Set Up Before Running Claude Code

### 1. SSH Access to the AWS Server
Claude Code needs to be able to run commands on your AWS EC2 instance. Options:
- **Option A (recommended):** Run Claude Code directly on the server via SSH
  ```bash
  ssh -i ~/Downloads/your-ssh-key.pem ubuntu@<YOUR_CURRENT_IP>
  # Then launch Claude Code from within the server
  ```
- **Option B:** Give Claude Code the SSH key path and server IP so it can SSH in

### 2. Telegram Chat ID
Claude Code needs your Telegram chat ID to send messages. Find it by:
```bash
# On the server, check gateway logs for your chat ID:
cat /tmp/openclaw/openclaw-*.log | grep -i "telegram" | grep -i "chat" | head -20
```
Or check existing OpenClaw config:
```bash
cat ~/.openclaw/openclaw.json | grep -i "target\|chat\|telegram"
```

### 3. Verify Whoop Data Scripts Still Work
Before building triggers on top of Whoop data, confirm the existing scripts return data:
```bash
cd ~/.openclaw/workspace/skills/whoop-central
node src/verify.js    # Should confirm auth works
node src/today.js     # Should return today's data
```
If token is expired, re-auth first.

### 4. Verify Google Calendar Access
```bash
# Check if the Google Calendar integration is accessible
# (exact command depends on how it was set up — check ~/.openclaw/ for google/calendar config)
```

### 5. Check OpenClaw Cron Capabilities
```bash
openclaw cron --help
# OR
openclaw hooks --help
```
This tells Claude Code whether to use OpenClaw's built-in scheduler or fall back to system crontab.

### 6. Share These Files With Claude Code
Point Claude Code at these directories so it understands the existing codebase:
- `~/.openclaw/workspace/` (entire workspace, especially SOUL.md and IDENTITY.md)
- `~/.openclaw/workspace/skills/whoop-central/` (existing Whoop skill to understand data format)
- `~/.openclaw/openclaw.json` (main config)
- `~/.clawdbot/` (credentials structure)

---

## Prompt for Claude Code

Copy-paste this when you start the Claude Code session:

---

**I need you to build a new OpenClaw skill called `proactive-triggers` for my Telegram bot Merlin. Read the full spec I'm about to share, then:**

1. **First, explore the existing codebase** — look at `~/.openclaw/workspace/skills/whoop-central/` to understand how existing skills are structured, how they fetch Whoop data, and how the token refresh works. Also read `~/.openclaw/workspace/SOUL.md` and `IDENTITY.md` to understand Merlin's personality.

2. **Check what scheduling/cron tools OpenClaw provides** — run `openclaw cron --help` and `openclaw hooks --help` to see what's available natively.

3. **Check how to send messages programmatically** — test `openclaw message send --channel telegram --target <CHAT_ID> --message "test"` to confirm messaging works.

4. **Build the skill incrementally:**
   - Start with the state manager and messenger wrapper
   - Then build the morning recovery check-in trigger (simplest, runs once daily)
   - Then add strain spike detection (requires polling)
   - Then bedtime nudge
   - Then sleep anomaly detection
   - Test each one before moving to the next

5. **After each trigger works, register it** with OpenClaw's scheduler or set up a system-level cron/daemon that runs the polling loop.

6. **Do NOT modify SOUL.md or IDENTITY.md** — personality is already set.

7. **Do NOT hardcode message text** — use the OpenClaw agent to generate messages in Merlin's voice based on context/data you pass it.

8. **Test by sending me a real message on Telegram** after each trigger is built so I can confirm it works and sounds right.

**Here's the full spec:**
[PASTE THE CONTENTS OF THIS DOCUMENT ABOVE]

---

## Success Criteria

- [ ] Morning check-in fires between 7-7:30 AM with actual Whoop data
- [ ] Strain spike detection catches a workout within 15 min and sends a message
- [ ] Bedtime nudge fires if still awake past 11:30 PM on a weeknight
- [ ] Sleep anomaly detection flags unusually short/long sleep
- [ ] Messages sound like Merlin (casual, Gen Z, Goggins energy)
- [ ] No more than 5 proactive messages per day
- [ ] No duplicate messages (state tracking works)
- [ ] Quiet hours respected (except bedtime nudge)
- [ ] System survives gateway restart and SSH disconnect
