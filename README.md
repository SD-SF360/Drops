# SF360 IPL Script Engine — Project Map
*Last updated: April 24, 2026*

---

## 1. System Overview

Automated pipeline generating audio drops for IPL matches, delivered via the SF360 fan app (card-based UI). All scripts are written, reviewed and approved by the editorial team before audio generation. No LLM auto-generation in production.

**Base path:** `C:\SD\SF360\SF360IPL_Script\`
**Backend:** `Backend\` | **Frontend:** `Frontend\`
**Stack:** FastAPI + GenVR TTS + FFmpeg + Cloudinary
**Conda env:** `sf360-env`
**Run:** `python -m uvicorn main:app` from `Backend\`

---

## 2. Drop Types

| Drop | Persona | Timing | Script Key |
|---|---|---|---|
| Match Preview | Arjun Mehta | 10AM match day | `fan_pre_match` |
| Match Preview Headline | Arjun Mehta | Display only | `fan_pre_match_headline` |
| Toss Report | Riya Kapoor | Post toss | `toss_report` |
| Post Match Story | Arjun Mehta | Post match | `fan_post_match_script` |
| Post Match Headline | Arjun Mehta | Display only | `fan_post_match_headline` |

---

## 3. Personas

### Arjun Mehta — Lead Anchor
- **Voice ID:** `eyrWyyYs-ArjunM` (confirmed, active)
- **Style:** Harsha Bhogle-esque. Warm, measured, narrative-driven storyteller. Builds to a point. Rich with human context. Comfortable with analysis.
- **Covers:** Match preview, post-match story
- **Tone:** Calm authority. Talks *at* the audience with gravitas.
- **Sentence structure:** Longer builds, paragraph-style delivery
- **TTS emotion:** `happy` for preview, `happy` for post-match
- **EQ profile:** Male voice — highpass 120Hz, cut 150Hz (-4dB), cut 300Hz (-2dB), boost 3.5kHz (+2dB), boost 8kHz (+1dB)
- **Crowd noise:** No — post-match is reflective, not live

### Riya Kapoor — Studio Host
- **Voice ID:** `qi83iL57-Riya` (cloned, active)
- **Style:** Mayanti Langer-esque. Warm, chirpy, efficient. Decent cricket knowledge but doesn't deep-dive into analysis. Talks *to* the fan, not *at* them.
- **Covers:** Toss report, mid-match updates (when required)
- **Tone:** Light energy, engaged, always present. Short punchy sentences. States the implication and moves on.
- **Sentence structure:** Short, efficient. No long builds.
- **TTS emotion:** `happy`
- **EQ profile:** Female voice — highpass 150Hz, cut 400Hz (-3dB), cut 500Hz (-2dB), boost 4.5kHz (+2dB), boost 10kHz (+1dB)
- **Crowd noise:** Yes — live stadium ambience at -18dB underneath
- **One-line summary:** She's the voice you hear at the ground when the toss happens, not the voice reflecting on the game after.

### Kabir Sharma — Field Reporter *(pending voice activation)*
- **Voice ID:** `YqvWBmOE-KabirS` (unconfirmed — needs GenVR activation)
- **Covers:** Match preview (energetic field reporter style)
- **Status:** Currently using Arjun's voice as fallback

### Neha Iyer — Segment Host *(pending)*
- **Voice ID:** Unconfirmed
- **Status:** Not yet developed

### Coach Vikram Rathore — Cricket Analyst *(pending)*
- **Voice ID:** Unconfirmed
- **Status:** Not yet developed

**To activate pending voice IDs:** Contact GenVR support via SF360 Enterprise account

---

## 4. Editorial Process

### Script Writing Protocol
1. Research match data from ESPN API, Sportstar, HindustanTimes
2. Write base draft (Claude assists)
3. Editorial review — fact-check all numbers against scorecards
4. Approve and lock into `schedule.json`
5. Generate audio via admin panel
6. Publish to CDN

### Script Length Guidelines
| Drop Type | Target Word Count |
|---|---|
| Match Preview | 150–160 words |
| Toss Report | 90–110 words |
| Post Match Story | 165–185 words |

### Writing Style Rules
- All numbers written out for TTS (forty-five, not 45)
- Always use full player names when surname alone may be mispronounced (Rishabh Pant, not just Pant)
- Em dashes replaced with hyphens in JSON (encoding safety)
- No fabricated quotes — real verified quotes only
- Cross-check all statistics against actual scorecard, not just match reports
- No hallucinated matchup stats or ball-by-ball claims without verification

### Match Preview Guidelines
- Written at 10AM on match day — XIs may not be confirmed
- Avoid overhyping uncertain team news (fitness, selection)
- Focus on form, pitch, key matchups, stakes
- Reference verified points table positions from the morning of match day
- Do not use pre-match predicted XIs as confirmed

### Toss Report Guidelines
- Written immediately after toss
- Cover: toss result, decision, key XI changes vs previous match, one pitch/conditions line
- Riya's voice — warm, efficient, not analytical
- End with a warm closer ("Should be a close one tonight")
- Keep it under 110 words

### Post Match Guidelines
- Written using: match report, quotes, scorecard
- Lead with the story, not the scorecard
- One or two verified direct quotes maximum
- NRR and points table context in final paragraph
- Never fabricate ball counts, over numbers or stats

---

## 5. Technical Architecture

### File Structure
```
Backend\
  main.py                    — FastAPI app, endpoints
  Intro wo mp3.mp3           — (moved to agents\ folder)
  agents\
    match_script_engine.py   — Manual script loader (no Groq)
    schedule_agent.py        — Reads schedule.json
    genvr_agent.py           — GenVR TTS API
    cloudinary_agent.py      — FFmpeg processing + Cloudinary upload
    mailer_agent.py          — Gmail SMTP email notifications
    Intro wo mp3.mp3         — Intro music sting (6 seconds)
    outro.mp3                — Outro music sting
    crowd_bg.wav             — Crowd ambience (trimmed to 19 seconds)
  data\
    schedule.json            — All match scripts
Frontend\
  index.html
  app.js
  style.css
```

### Key Environment Variables (.env)
```
CLOUDINARY_CLOUD_NAME=dflnsufit
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
GENVR_API_KEY=
GMAIL_SENDER=sashanka.deka@sportsfan360.com
GMAIL_APP_PASSWORD=
GMAIL_RECIPIENT=engineering@sportsfan360.com
```

---

## 6. Audio Processing Pipeline

Every generated audio goes through this chain before Cloudinary upload:

```
GenVR TTS (raw audio)
  → Download to temp file
  → FFmpeg:
      [intro music] + 1sec gap + [voice] + 1sec gap + [outro music]
      + crowd ambience looped at -18dB (Riya only)
      + EQ (persona-specific)
      + compression (ratio 2:1, attack 10ms, release 100ms)
      + loudnorm (-16 LUFS broadcast standard)
  → Upload processed file to Cloudinary
  → Return CDN URL
```

### FFmpeg EQ Chains

**Arjun (male voice):**
```
highpass=120Hz → cut 150Hz (-4dB) → cut 300Hz (-2dB)
→ boost 3.5kHz (+2dB) → boost 8kHz (+1dB)
→ compress ratio 2:1 → loudnorm -16 LUFS
```

**Riya (female voice):**
```
highpass=150Hz → cut 400Hz (-3dB) → cut 500Hz (-2dB)
→ boost 4.5kHz (+2dB) → boost 10kHz (+1dB)
→ compress ratio 2:1 → loudnorm -16 LUFS
```

### FFmpeg Installation
- Path: `C:\ffmpeg\ffmpeg-8.1-essentials_build\bin`
- Added to Windows System PATH

---

## 7. schedule.json Structure

```json
[
  {
    "date": "2026-04-23",
    "teams": ["MI", "CSK"],
    "status": "completed",
    "venue": "Wankhede Stadium, Mumbai",
    "match_id": 1529276,
    "series_id": 1510719,
    "manual_scripts": {
      "fan_pre_match_headline": "...",
      "fan_pre_match": "...",
      "toss_report": "...",
      "fan_post_match_headline": "...",
      "fan_post_match_script": "..."
    }
  }
]
```

**Status values:** `upcoming` | `completed`
All scripts in `manual_scripts` bypass LLM generation entirely.

---

## 8. Admin Panel (Frontend)

**URL:** `http://127.0.0.1:8000` (served separately via index.html)

### Workflow
1. Click **Load Scripts** — fetches `/match-scripts`
2. Each match card shows all available script types
3. Headlines render as styled display text (not editable)
4. Script cards show: persona dropdown (locked), audio/video toggle, Generate button, Publish button

### Persona Dropdown Locking
| Script Type | Locked To |
|---|---|
| `toss_report` | Riya Kapoor |
| `fan_pre_match` | Arjun Mehta (Kabir pending) |
| `fan_post_match_script` | Arjun Mehta |

Other personas greyed out and marked "(unavailable)" until voice IDs are activated.

### Generate vs Publish
- **Generate** — creates audio, uploads to Cloudinary, returns CDN URL. No email.
- **Publish** — same as Generate but also triggers email notification (when enabled)

---

## 9. Email Notification System

**File:** `Backend\agents\mailer_agent.py`
**Toggle:** `MAILER_ENABLED = True/False` (currently False — testing phase)
**Trigger:** Publish button only (not Generate)
**Content:** Headline as subject, audio CDN URL + persona image URL in body
**SMTP:** Gmail (Google Workspace) via App Password

**To enable:** Set `MAILER_ENABLED = True` in `mailer_agent.py`

---

## 10. Data Sources

| Source | Status | Used For |
|---|---|---|
| ESPN API | Active | Match info, toss, XIs, standings |
| Sportstar | Active (200 OK) | Article bodies, quotes |
| HindustanTimes | Active (200 OK) | Article bodies |
| ESPNCricinfo RSS | Headlines only (403 on body) | Titles |
| Cricbuzz | Blocked (403) | Not used |
| Google News | Redirect works | Supplementary |

**Note:** Groq LLM generation is fully disabled. All scripts are human-written.

---

## 11. Cloudinary Structure

```
sf360/
  audio/
    {Team1}_vs_{Team2}_{script_type}_{Persona_Name}_{DDMMYYYY}.mp3
  video/
    {Team1}_vs_{Team2}_{script_type}_{Persona_Name}_{DDMMYYYY}.mp4
```

Example: `sf360/audio/MI_vs_CSK_post_match_script_Arjun_Mehta_23042026.mp3`

---

## 12. Known Pending Items

| Item | Priority | Notes |
|---|---|---|
| Activate Kabir, Neha, Coach Vikram voice IDs | High | Email jignesh@sportsfan360.com |
| Test email notification (mailer) | High | Enable MAILER_ENABLED = True when ready |
| Riya's persona prompt in script_agent (if Groq re-enabled) | Low | Brief exists |
| Scheduled triggers (10AM preview, post-toss, post-match) | Medium | Not built |
| H2H data integration | Low | WIP |
| Story agent (ball-by-ball pre-processing) | Low | Not built |
| Upgrade Groq to paid tier (if re-enabled) | Low | Hits 100k/day free limit |

---

## 13. Session Handoff Protocol

When starting a new conversation, paste this prompt:

> "Continuing SF360 IPL Script Engine project. Reference the project map at SF360_IPL_Script_Engine_ProjectMap.md for full context. Current schedule.json has [X] matches. Today's task: [task]."

Then upload any relevant files (schedule.json, agent files) if changes are needed.

---

*SF360 IPL Script Engine — Internal Documentation*
*Not for distribution*
