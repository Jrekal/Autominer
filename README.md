# Japanese Anki Auto-Mining (Voicevox edition)

**What this is**  
A local pipeline that mines Japanese words from local video + subtitle files, clips the sentence audio, synthesizes word/definition audio via a local Voicevox engine, and creates Anki notes (via AnkiConnect or by exporting an `.apkg`).  
This repository contains the script `autominer_final_voicevox_full.py` (main program), plus this README and helper files.

---

## Quick summary of required software

- Python 3.10 or 3.11 (3.8+ will often work, but 3.10/3.11 is recommended)
- `ffmpeg` (command-line binary available in PATH)
- Voicevox (local engine) — running locally, reachable at `http://127.0.0.1:50021`
- Anki (desktop) with the **AnkiConnect** add-on installed and enabled
- SudachiPy tokenizer and a Sudachi dictionary (see instructions)
- pip packages listed in `requirements.txt`

---

## Files included

- `autominer_final_voicevox_full.py` — main script (configure `BASE_FOLDER` etc. if needed)
- `requirements.txt` — Python packages to install
- `README.md` — this file
- `.gitignore` — suggested ignore patterns

---

## Install & setup (Windows example; macOS / Linux notes below)

### 1) Install Python and create virtual environment
1. Install Python 3.10/3.11 from python.org (select “Add to PATH” on Windows).
2. Open Command Prompt / PowerShell and create a venv:

```powershell
cd C:\path\to\project
python -m venv .venv
.venv\Scripts\activate     # PowerShell: .venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install -r requirements.txt
On macOS/Linux:

bash
Copy code
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
2) Install ffmpeg
Windows: download a static build from ffmpeg site, unzip, and add ffmpeg\bin to your PATH.

macOS: brew install ffmpeg

Linux: use system package manager, e.g. sudo apt install ffmpeg

Confirm with:

bash
Copy code
ffmpeg -version
3) Install & run Voicevox (local)
Download & install the Voicevox engine appropriate for your OS (Voicevox provides desktop + engine variants).

Start the engine so the HTTP API is available at http://127.0.0.1:50021.

Confirm with a quick curl (or in browser): http://127.0.0.1:50021 — you should get a response or at least engine reachable.

The script uses voice VOICEVOX_SPEAKER_ID (default in the script: 11). Change it in the config if you prefer a different voice.

4) Install Anki + AnkiConnect
Install Anki Desktop.

From Anki, add-on > Browse add-ons > install add-on by code: AnkiConnect (use current add-on ID from AnkiWeb; the script uses the default AnkiConnect endpoint http://127.0.0.1:8765).

Restart Anki. Ensure Anki is running when you run the script if you want immediate add via AnkiConnect.

5) SudachiPy dictionary (important)
SudachiPy requires a dictionary package. Recommended: sudachidict_full.

Install with pip (already in requirements.txt), but you still need to download dictionary data:

bash
Copy code
python -c "from sudachipy import dictionary; dictionary.Dictionary().create()"
If SudachiPy requests a dictionary package, install it:

bash
Copy code
pip install sudachidict_full
You can also run:

bash
Copy code
python -c "import sudachipy; print('sudachipy ok')"
6) OpenAI credentials (safe usage)
Do not put your OpenAI API key in the code in a public repo.

Set it as an environment variable:

Windows (PowerShell):

powershell
Copy code
setx OPENAI_API_KEY "sk-xxxxxxxx..."
# Restart terminal so new variables take effect
Windows (CMD):

cmd
Copy code
set OPENAI_API_KEY=sk-xxxxxxxx...
macOS / Linux (bash):

bash
Copy code
export OPENAI_API_KEY="sk-xxxxxxxx..."
The script will read OPENAI_API_KEY from env if OPENAI_API_KEY in the script is None.

Configuration (small checklist)
Place videos into BASE_FOLDER/Videos/. Supported subtitle extensions: .srt, .ass, .ssa, .vtt. Subtitles should have the same basename as the video file (e.g., ShowEp1.mkv + ShowEp1.srt).

Set TOGGLE_FILE to ON (file toggle.txt in base folder) to enable processing.

Edit BASE_FOLDER near the top of the script if you prefer a custom folder.

Edit VOICEVOX_SPEAKER_ID if you want a different voice.

DAILY_CARD_LIMIT controls how many cards added per run.

