# 🧠 Second Brain

A personal, local-first knowledge system that captures, organizes, and automatically summarizes content from YouTube and GitHub — synced across devices, powered by a local AI, and costing **$0/month**.

![Cost](https://img.shields.io/badge/cost-%240%2Fmonth-brightgreen)
![Stack](https://img.shields.io/badge/stack-Obsidian%20%2B%20Ollama%20%2B%20Python-blue)
![Platform](https://img.shields.io/badge/platform-Ubuntu-orange)

This README is written so that **anyone can follow it from zero**, even with nothing installed yet. Every command is exact — copy, paste, press Enter.

---

## ✨ What it does

- **Captures** X (Twitter) posts and threads with one click, via the official Obsidian Web Clipper

  ![Example of a clipped X post landing in the vault](./screenshots/x-capture-example.png)

- **Watches** chosen YouTube channels and GitHub repos for new content matching your interests
- **Summarizes** everything locally using [Ollama](https://ollama.com) running Hermes 3 — no OpenAI, no Claude API, no cloud LLM bill
- **Organizes** notes automatically into the right topic folder, with consistent frontmatter
- **Syncs** the whole vault between a PC and phone using [Syncthing](https://syncthing.net) — no Obsidian Sync subscription
- **Runs unattended** via a cron job checking for new content every 6 hours

## 🗂️ Final vault structure

```
Second Brain/
├── School/
│   └── 2026/, 2027/...
├── Tech/
│   ├── ML/
│   ├── Cyber/
│   ├── Backend/
│   └── Vibe-Coding/
├── Social Media Research/
│   ├── X/          ← manual clips via Web Clipper
│   └── YouTube/    ← auto-generated
└── Projects/
    └── <one folder per project, created as needed>
```

## 🏗️ Architecture

```
sources.yaml (you edit — channels, repos, keywords)
        │
   ┌────┴────┐
   ▼         ▼
YouTube    GitHub
ingest     ingest
(free      (free
 API)       API)
   │         │
   └────┬────┘
        ▼
  Ollama (Hermes 3) — local, offline, free
        ▼
  Note written into the right vault folder
        ▼
  Synced to phone via Syncthing
```

---

# 📖 Full step-by-step guide

Everything below is exactly what was done to build this, in order. Follow it top to bottom on a fresh Ubuntu machine and you'll end up with the same working system.

## Part 1 — Install Obsidian

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.repo
flatpak install -y flathub md.obsidian.Obsidian
```

Launch it with:
```bash
flatpak run md.obsidian.Obsidian
```

## Part 2 — Create the vault folder structure

```bash
mkdir -p ~/"Second Brain"/School/{2026,2027}
mkdir -p ~/"Second Brain"/Tech/{ML,Cyber,Backend,Vibe-Coding}
mkdir -p ~/"Second Brain"/"Social Media Research"/{X,YouTube}
mkdir -p ~/"Second Brain"/Projects
```

In Obsidian: **Open folder as vault** → select `~/Second Brain`.

## Part 3 — Install Syncthing (free cross-device sync)

```bash
sudo apt update
sudo apt install -y syncthing
```

If that fails with "package not found":
```bash
sudo add-apt-repository universe
sudo apt update
```

Start it and open the control panel:
```bash
systemctl --user enable --now syncthing.service
```
Open `http://localhost:8384` in your browser. Go to **Settings → GUI** and set a username/password.

**Open the firewall ports it needs:**
```bash
sudo ufw allow 22000/tcp
sudo ufw allow 22000/udp
sudo ufw allow 21027/udp
sudo ufw reload
```

### On your phone

Install **F-Droid** (f-droid.org), then install **Syncthing-Fork** from it (the original official Android app is discontinued — this is the actively maintained replacement).

> ⚠️ **If Syncthing-Fork says "not running"** because of a metered-WiFi setting: tap **FORCE START IGNORE RUN CONDITIONS**, then go to its gear icon → **Run Conditions** and allow it on metered WiFi so it starts automatically every time.

> ⚠️ **Grant full storage permission.** Go to Android **Settings → Apps → Syncthing-Fork → Permissions → Files and media** and enable **"Allow management of all files."** Without this, Syncthing can only get *read* access and sync will silently fail to write anything.

### Pairing the two devices

1. On the PC's Syncthing web page: **Actions → Show ID** — shows a QR code.
2. On the phone, in Syncthing-Fork, tap the "add device" icon and **scan the QR code**.
3. Approve the connection request that appears on the PC.

### Sharing the vault folder

1. On the PC: **+ Add Folder** → Folder Path: the output of `readlink -f ~/"Second Brain"` → Sharing tab → tick your phone's name → Save.
2. On the phone: accept the incoming folder notification. If it tries to auto-create the folder and fails with "read-only file system," tap **Add** manually instead and pick a location under **Internal Storage** using the folder-browse icon (not by typing a path).
3. **Double-check Folder Type is "Send & Receive" on BOTH devices** — if either side says "Send Only" or "Receive Only," sync will only go one direction. This was the single most common cause of sync not working during setup.

### Install Obsidian on the phone

From the Play Store, open the app, **Open folder as vault**, and select the folder Syncthing just synced.

## Part 4 — Install the Obsidian Web Clipper (for X posts)

1. In Brave/Chrome, go to the [Obsidian Web Clipper Chrome Web Store page](https://chromewebstore.google.com/detail/obsidian-web-clipper/cnjifjpddelmedmihgijeibhnjfabmlf) and click **Add to Brave**.
2. Right-click its toolbar icon → **Options**.
3. Under **General → Vaults**, type `Second Brain` and press **Enter** (must press Enter, not just click away).
4. Click **New template**. Name it "X post". Set:
   - **Template triggers:** `x.com` and `twitter.com` (one per line, no `https://`)
   - **Note location:** `Social Media Research/X`
   - **Vault:** `Second Brain`
   - **Properties:** add `source` = `x.com`, `author` = `{{author}}`, `url` = `{{url}}`, `date` = `{{date}}`

**To actually clip a post:** open its individual permalink page (click the timestamp), select/highlight the text you want first, *then* click the clipper icon and save — X is JavaScript-heavy, so clipping without first selecting the text often grabs nothing.

## Part 5 — Install the automation pipeline

```bash
chmod +x setup-pipeline.sh
./setup-pipeline.sh
```

This installs Ollama, downloads the Hermes 3 (8B) model (~4.7GB), and installs the Python packages needed (`requests`, `pyyaml`, `python-dotenv`, `youtube-transcript-api`, `python-frontmatter`).

### Get two free API keys

**YouTube Data API key** (free, no card required):
1. Go to [console.cloud.google.com](https://console.cloud.google.com), create a new project.
2. Search for **"YouTube Data API v3"** → click **Enable**.
3. Go to **Credentials → + Create Credentials → API key**.
4. If asked to restrict the key, select **YouTube Data API v3** from the list and continue.

**GitHub personal access token** (free):
1. Go to [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**.
2. Tick **repo**. Leave everything else unchecked.
3. Generate it and copy it immediately — GitHub only shows it once.

### Save your keys

```bash
cp .env.example .env
nano .env
```
Paste each key after its `=` sign (no spaces). Save with `Ctrl+O`, Enter, then exit with `Ctrl+X`.

### Configure what to track

```bash
nano sources.yaml
```

Example entry for a YouTube channel:
```yaml
youtube:
  - channel_id: "UC9x0AN7BWHpCDHSm9NiJFJQ"
    name: "NetworkChuck"
    keywords: ["hacking", "networking", "linux", "cybersecurity", "AI"]
    topic_folder: "Tech/Cyber"
```

To find a channel's ID: use [commentpicker.com/youtube-channel-id.php](https://commentpicker.com/youtube-channel-id.php) and paste the channel's URL or `@handle`.

Example entry for a GitHub repo:
```yaml
github:
  - repo: "ollama/ollama"
    topic_folder: "Tech/Backend"
```

Valid `topic_folder` values: `Tech/ML`, `Tech/Cyber`, `Tech/Backend`, `Tech/Vibe-Coding`, `Social Media Research/YouTube`.

**YAML rules that matter:** each list item starts with `- ` (dash, then a space), and the lines underneath it must line up with 4 spaces of indent.

### Run it

```bash
python3 run_pipeline.py
```

Watch for lines like `wrote note: /home/you/Second Brain/Tech/Cyber/...md` — that means it worked. Each note takes a minute or two since it's running on your own CPU.

> ⚠️ **Known issue (already fixed in this repo's code):** the `youtube-transcript-api` library changed its interface in a later version — `YouTubeTranscriptApi.get_transcript(video_id)` was replaced with an instance-based call. The current `secondbrain/ingest_youtube.py` already uses the fixed version:
> ```python
> ytt_api = YouTubeTranscriptApi()
> fetched = ytt_api.fetch(video_id)
> text = " ".join(snippet.text for snippet in fetched)
> ```
> If you ever see `AttributeError: type object 'YouTubeTranscriptApi' has no attribute 'get_transcript'`, this is why — make sure you're using the version of the file from this repo.

If it finds nothing on a re-run even after a fix, it's because it already marked those videos as "seen." Clear that memory with:
```bash
rm state.json
```

## Part 6 — Schedule it to run automatically

```bash
chmod +x setup-cron.sh
./setup-cron.sh
```

This detects your Python path and project folder automatically and adds this line to your schedule (runs every 6 hours):
```
0 */6 * * * cd ~/second-brain-pipeline && /usr/bin/python3 run_pipeline.py >> pipeline.log 2>&1
```

Check it any time with `crontab -l`. See what happened on each run in `pipeline.log`.

## Part 7 — (Optional) Anki flashcard sync

The notes already include a `## Flashcards` Q/A section, ready for Anki. To sync automatically:

1. In Anki: **Tools → Add-ons → Get Add-ons**, paste code `2055492159` (AnkiConnect), restart Anki.
2. Keep Anki open, then run:
```bash
pip install --user --break-system-packages python-frontmatter
python3 anki_sync.py
```
This creates a deck per topic (e.g. `SecondBrain::Tech/Cyber`) and adds every Q/A pair it finds, tagging cards by topic and source.

---

## 🛠️ Tech stack

| Piece | Tool | Why |
|---|---|---|
| Notes | [Obsidian](https://obsidian.md) | Free, local-first, markdown |
| Sync | [Syncthing](https://syncthing.net) | Free, peer-to-peer, no subscription |
| Local AI | [Ollama](https://ollama.com) + Hermes 3 | Free, private, offline |
| Ingestion | Python + YouTube Data API + GitHub API | Both have genuinely free tiers |
| Flashcards | Anki + AnkiConnect | Free, local |
| X capture | [Obsidian Web Clipper](https://obsidian.md/clipper) | Official, free, no server |

## 📌 Left manual, on purpose

- Anki flashcard review
- Choosing how a note gets written (AI-written / guided / self-typed)
- Turning a repo into a step-by-step "build this yourself" plan
- School PDF ingestion

## 💸 Cost breakdown

| Item | Cost |
|---|---|
| Obsidian | Free |
| Syncthing | Free |
| Ollama + Hermes 3 | Free (local compute) |
| YouTube Data API | Free tier (10,000 units/day) |
| GitHub API | Free tier |
| **Total** | **$0/month** |

---

*Built step by step, no paid APIs, no subscriptions.*
