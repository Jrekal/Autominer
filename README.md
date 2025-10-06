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

# autominer_final_voicevox_full.py
# Full automated local-video -> audio-only Anki mining pipeline
# Voicevox (local engine) used for TTS (voice id 11)
# Keep all other behavior the same as your provided script.
# Run: python autominer_final_voicevox_full.py

import os
import json
import time
import subprocess
import re
import logging
import unicodedata
import base64
import uuid
from pathlib import Path
from datetime import datetime, date
from typing import Dict, Tuple, List

# NLP/API libs
from sudachipy import dictionary, tokenizer
from openai import OpenAI
import genanki
import requests

# ------------------ CONFIG ------------------
# NOTE: changed from a user-specific absolute path to a neutral default so this script
# can be shared safely. Users can edit BASE_FOLDER to point to a custom location.
BASE_FOLDER = Path.home() / "JapaneseAnkiAutoMining"
VIDEOS_FOLDER = BASE_FOLDER / "Videos"
AUDIO_FOLDER = BASE_FOLDER / "Audio"
SENTENCE_AUDIO_FOLDER = AUDIO_FOLDER / "Sentences"
WORD_AUDIO_FOLDER = AUDIO_FOLDER / "Words"
DEF_AUDIO_FOLDER = AUDIO_FOLDER / "Definitions"
ANKI_FOLDER = BASE_FOLDER / "AnkiDecks"

TOGGLE_FILE = BASE_FOLDER / "toggle.txt"
BACKLOG_TXT = BASE_FOLDER / "backlog.txt"   # Plain text backlog: tab-separated records
STATS_FILE = BASE_FOLDER / "stats.json"

FFMPEG_CMD = "ffmpeg"
DAILY_CARD_LIMIT = 30

ANKI_DECK_NAME = "Japanese Audio Sentence Cards"
NOTE_TYPE_NAME = "Japanese automining"

# ========== OPENAI KEY HANDLING (removed embedded personal key) ==========
# Safety: do NOT embed personal keys in shared code. The script will look for
# an API key in the OPENAI_API_KEY constant or the OPENAI_API_KEY environment variable.
# For sharing, we leave the constant unset and instruct users to set the env var.
OPENAI_API_KEY = None
OPENAI_MODEL = "gpt-4o-mini"

# Voicevox config
VOICEVOX_URL = "http://127.0.0.1:50021"  # local engine endpoint
VOICEVOX_SPEAKER_ID = 11  # target voice id (change as desired)

# video & subtitle extensions to detect
VIDEO_EXTS = {".mp4", ".mkv", ".avi", ".mov", ".flv", ".webm", ".wmv"}
SUBTITLE_EXTS = {".srt", ".ass", ".ssa", ".vtt"}

# ------------------ LOGGING ------------------
logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")

# ------------------ ENSURE DIRECTORIES ------------------
for d in (VIDEOS_FOLDER, AUDIO_FOLDER, SENTENCE_AUDIO_FOLDER, WORD_AUDIO_FOLDER, DEF_AUDIO_FOLDER, ANKI_FOLDER):
    d.mkdir(parents=True, exist_ok=True)

if not TOGGLE_FILE.exists():
    TOGGLE_FILE.write_text("OFF", encoding="utf-8")

if not STATS_FILE.exists():
    STATS_FILE.write_text(json.dumps({"last_date": None, "added_today": 0}, ensure_ascii=False), encoding="utf-8")

# ------------------ BLACKLISTS (embedded exactly as provided) ------------------
_blacklist_common_str = r"""
の
に
です
から

"""
# Convert to set of tokens
blacklist_common = set([w.strip() for w in _blacklist_common_str.splitlines() if w.strip()])

_blacklist_aux_str = r"""
の は が を に で と も から まで よ ね ぞ さ し けど けれど けれども へ や か だ です ます ません だった では ではない だったら でしょう かもしれない という というのは ということ というわけ というふう に よう ように ような ようだ らしい みたい そうだ そうな そうに そうする とき ときに ときは ところ ところで ところが もの こと ため ために ための する した して している していた しない しなかった します しました できる できない できた できて できている ある あるだろう ない いた いない なる なった なって なるだろう いらっしゃる くださる もらう あげる くれる おる おられる おっしゃる いただく ござる 自分 それ これ あれ どれ この その あの どの ここ そこ あそこ どこ こちら そちら あちら どちら 私 僕 俺 あたし あなた 君 お前 あんた 彼 彼女 彼ら 彼女ら 私たち 僕たち 俺たち 誰 何 どんな どれほど どのくらい いつ どう どうして なぜ だから それで それでは それなのに それなら それならば そして すると そのうえ そのため しかし でも けれども けど ところが なのに だけど したがって つまり 要するに さて ちなみに だからこそ それなのに けれど けれども ねばならない なければならない なくてはならない なければいけない なくてはいけない よい いい よかった よくない よくなかった 悪い 悪かった 悪くない 高い 低い 安い 新しい 古い 大きい 小さい 多い 少ない 長い 短い 強い 弱い 熱い 寒い 暑い 冷たい 白い 黒い 赤い 青い 黄色い 緑の 茶色の 美しい 綺麗だ 汚い 上手 下手 好き 嫌い 大丈夫 元気 静か 賑やか 便利 不便 無理 簡単 難しい 大事 大切 必要 不要 同じ 違う 違い 同様 全然 全く 本当 嘘 たくさん 少し いっぱい すぐ よく もう まだ また とても とっても 全然 かなり けっこう ぜったい 絶対 もちろん 多分 きっと もし たぶん どうやって どうぞ ありがとう どうも おはよう こんにちは こんばんは さようなら すみません ごめんなさい お願いします よろしく はじめまして
"""
blacklist_aux = set([w.strip() for w in _blacklist_aux_str.split() if w.strip()])

# combine blacklist
blacklist_user = set()  # empty by default; user can add
blacklist = set.union(blacklist_common, blacklist_aux, blacklist_user)
logging.info(f"Blacklists loaded: common={len(blacklist_common)}, aux={len(blacklist_aux)}, total={len(blacklist)}")

# ------------------ SUDACHIPY TOKENIZER ------------------
tokenizer_obj = dictionary.Dictionary().create()
SPLIT_MODE = tokenizer.Tokenizer.SplitMode.C

# Helper: check if a string has at least one CJK kanji character or kana
def contains_japanese_char(s: str) -> bool:
    return bool(re.search(r'[\u3040-\u309F\u30A0-\u30FF\u4E00-\u9FFF]', s))

# Helper: check if string is only kana (hiragana/katakana)
def is_only_kana(s: str) -> bool:
    return bool(re.fullmatch(r'[\u3040-\u309F\u30A0-\u30FF]+', s))

# common tiny interjections to reject (conservative list)
_common_interjections = {"あっ", "ああ", "ええ", "うん", "あれ", "あの", "うわ", "おお", "おい", "えっ", "えー", "あら", "ねえ", "ん", "んん"}

# Characters to remove: directional & zero-width marks etc.
_INVISIBLE_RE = re.compile(r'[\u200B-\u200F\u202A-\u202E\u2060-\u206F\uFEFF]')

# Remove leading/trailing punctuation
_PUNCT_RE = re.compile(r'^[\s\(\)\[\]「」『』【】〈〉《》\u3000\.,:;!？?!。、・…\-\–\—"\'\u2026]+|[\s\(\)\[\]「」『』【】〈〉《》\u3000\.,:;!？?!。、・…\-\–\—"\'\u2026]+$')

def clean_surface(s: str) -> str:
    if not s:
        return ""
    # normalize
    s = unicodedata.normalize("NFKC", s)
    # remove invisible/unwanted control chars
    s = _INVISIBLE_RE.sub("", s)
    # remove common Unicode left/right mark strings like "lrm" spelled out (rare), just strip ascii control-like tokens
    s = s.strip()
    # strip surrounding punctuation/parentheses/etc
    s = _PUNCT_RE.sub("", s)
    # finally collapse whitespace
    s = re.sub(r'\s+', ' ', s).strip()
    return s

# Decide whether to keep a Sudachi morpheme as a candidate token
def should_keep_token(m) -> bool:
    raw = m.surface()
    if not raw:
        return False
    surf = clean_surface(raw)
    if not surf:
        return False

    # Reject tokens that are obvious ascii control words (like 'lrm') or likely not in subtitles
    if re.fullmatch(r'[A-Za-z\-_]{1,6}', surf):
        return False

    # must contain at least one Japanese character (kanji/kana)
    if not contains_japanese_char(surf):
        return False

    pos = m.part_of_speech()  # list of strings like ['名詞','固有名詞','人名','名']
    pos0 = pos[0] if len(pos) > 0 else ""
    pos1 = pos[1] if len(pos) > 1 else ""
    # Filter out junk categories
    if pos0 in ("助詞", "助動詞", "記号", "接頭辞", "接尾辞", "フィラー"):
        return False

    # Exclude proper nouns (固有名詞) — this cuts character/place names
    if pos0 == "名詞" and pos1 == "固有名詞":
        return False

    # Skip pure numbers
    if re.fullmatch(r'\d+', surf):
        return False

    # Skip single-character hiragana/katakana tokens unless it's a kanji (we already check kanji presence)
    if len(surf) == 1 and not re.search(r'[\u4E00-\u9FFF]', surf):
        return False

    # Skip tiny kana-only tokens (interjections/particles) unless they are in whitelist (none by default)
    if is_only_kana(surf) and len(surf) <= 2:
        if surf in _common_interjections:
            return False
        return False

    # blacklist check (user/common)
    if surf in blacklist:
        return False

    # final surface length sanity
    if len(surf) < 1 or len(surf) > 25:
        return False

    # Passed all checks: accept (but return cleaned surface as result consumer should use)
    return True

# function that returns cleaned surface from morpheme (used when adding candidate)
def cleaned_surface_of(m) -> str:
    return clean_surface(m.surface())

# ------------------ BACKLOG (plain text) helpers ------------------
# Each line: word\tcount\tfirst_seen_iso\tadded_flag (true/false)
def read_backlog_txt() -> Dict[str, Dict]:
    data = {}
    if not BACKLOG_TXT.exists():
        return data
    for ln in BACKLOG_TXT.read_text(encoding="utf-8").splitlines():
        if not ln.strip():
            continue
        parts = ln.split("\t")
        word = parts[0]
        count = int(parts[1]) if len(parts) > 1 and parts[1].isdigit() else 0
        first_seen = parts[2] if len(parts) > 2 else datetime.now().isoformat()
        added = parts[3].lower() == "true" if len(parts) > 3 else False
        data[word] = {"count": count, "first_seen": first_seen, "added": added}
    return data

def write_backlog_txt(data: Dict[str, Dict]):
    lines = []
    for w, meta in data.items():
        count = meta.get("count", 0)
        first_seen = meta.get("first_seen", datetime.now().isoformat())
        added = "true" if meta.get("added", False) else "false"
        lines.append(f"{w}\t{count}\t{first_seen}\t{added}")
    BACKLOG_TXT.write_text("\n".join(lines), encoding="utf-8")

# ------------------ STATS JSON helpers ------------------
def read_stats():
    try:
        return json.loads(STATS_FILE.read_text(encoding="utf-8"))
    except Exception:
        return {"last_date": None, "added_today": 0}

def write_stats(s):
    STATS_FILE.write_text(json.dumps(s, ensure_ascii=False, indent=2), encoding="utf-8")

# ------------------ SUBTITLE PARSING (SRT and ASS) ------------------
time_re = re.compile(r"(\d+):(\d+):(\d+)[,.](\d+)")

def srt_time_to_seconds(t: str) -> float:
    t = t.strip().replace(".", ",")
    m = time_re.match(t)
    if not m:
        parts = t.replace(",", ".").split(":")
        if len(parts) == 3:
            h, mm, ss = parts
            return int(h) * 3600 + int(mm) * 60 + float(ss)
        return 0.0
    h, mm, ss, ms = m.groups()
    return int(h)*3600 + int(mm)*60 + int(ss) + int(ms)/1000.0

def parse_srt_blocks(path: Path) -> List[Tuple[int, float, float, str]]:
    text = path.read_text(encoding="utf-8")
    lines = text.replace("\r\n", "\n").replace("\r", "\n").split("\n")
    blocks = []
    i = 0
    while i < len(lines):
        if not lines[i].strip():
            i += 1
            continue
        idx_line = lines[i].strip()
        i += 1
        if i >= len(lines):
            break
        time_line = lines[i].strip()
        i += 1
        if "-->" not in time_line:
            while i < len(lines) and lines[i].strip():
                i += 1
            continue
        start_str, end_str = [t.strip() for t in time_line.split("-->")]
        try:
            start_s = srt_time_to_seconds(start_str)
            end_s = srt_time_to_seconds(end_str)
        except:
            start_s = 0.0
            end_s = start_s + 3.0
        text_lines = []
        while i < len(lines) and lines[i].strip():
            text_lines.append(lines[i].strip())
            i += 1
        sentence = " ".join(text_lines)
        try:
            idx = int(idx_line)
        except:
            idx = None
        blocks.append((idx, start_s, end_s, sentence))
    return blocks

# ASS subtitle parser: look for "Dialogue:" lines
def parse_ass_blocks(path: Path) -> List[Tuple[int, float, float, str]]:
    text = path.read_text(encoding="utf-8", errors="ignore")
    blocks = []
    idx = 0
    for line in text.splitlines():
        line = line.strip()
        if not line:
            continue
        if line.startswith("Dialogue:"):
            rest = line[len("Dialogue:"):].lstrip()
            parts = rest.split(",", 9)
            if len(parts) >= 3:
                try:
                    start_s = srt_time_to_seconds(parts[1].strip().replace(".", ",")) 
                except:
                    start_s = 0.0
                try:
                    end_s = srt_time_to_seconds(parts[2].strip().replace(".", ",")) 
                except:
                    end_s = start_s + 3.0
                text_field = parts[9] if len(parts) >= 10 else ",".join(parts[3:])
                text_field = re.sub(r"\{.*?\}", "", text_field)
                text_field = text_field.replace("\\N", " ").replace("\\n", " ").strip()
                blocks.append((idx, start_s, end_s, text_field))
                idx += 1
    return blocks

def parse_subtitle_file(path: Path) -> List[Tuple[int, float, float, str]]:
    suffix = path.suffix.lower()
    if suffix in {".srt", ".vtt"}:
        return parse_srt_blocks(path)
    if suffix in {".ass", ".ssa"}:
        return parse_ass_blocks(path)
    return parse_srt_blocks(path)

# ------------------ FFmpeg clipping ------------------
def clip_sentence_audio(video_path: Path, start_s: float, end_s: float, out_path: Path) -> bool:
    out_path.parent.mkdir(parents=True, exist_ok=True)
    cmd = [
        FFMPEG_CMD, "-y",
        "-ss", str(start_s),
        "-to", str(end_s),
        "-i", str(video_path),
        "-vn", "-ac", "1", "-ar", "22050",
        str(out_path)
    ]
    try:
        subprocess.run(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, check=True)
        return True
    except Exception as e:
        logging.error(f"FFmpeg error for {video_path.name} [{start_s}-{end_s}]: {e}")
        return False

# ------------------ OpenAI client for gpt-4o-mini ------------------
openai_client = None
def init_openai():
    global openai_client
    if openai_client is not None:
        return True
    key = OPENAI_API_KEY or os.getenv("OPENAI_API_KEY")
    if not key:
        logging.error("OpenAI key not available. Set the OPENAI_API_KEY environment variable or assign OPENAI_API_KEY in the script (not recommended).")
        return False
    openai_client = OpenAI(api_key=key)
    return True

def get_japanese_description(word: str, retries=2) -> str:
    if not init_openai():
        raise RuntimeError("OpenAI client not initialized")
    prompt = (f"Write one concise, natural-sounding Japanese DESCRIPTION (one short sentence) for the word 「{word}」. Don't just give a traditional definition; actually explain what the word means in a similar manner to how the word would be explained to a 15 year old Japanese person. Do not add any sentence enders like んだよ or です or  ます or だよ"
              " Don't translate into any other language. Make it sound like a native speaker explaining the word.")
    for attempt in range(retries + 1):
        try:
            resp = openai_client.chat.completions.create(
                model=OPENAI_MODEL,
                messages=[{"role": "user", "content": prompt}],
                temperature=0.2,
                max_tokens=120
            )
            return resp.choices[0].message.content.strip()
        except Exception as e:
            logging.warning(f"OpenAI attempt {attempt} failed for '{word}': {e}")
            time.sleep(1 + attempt * 2)
    raise RuntimeError(f"OpenAI failed for '{word}' after retries")

# ------------------ VOICEVOX TTS (sync using local engine HTTP API) ------------------
def voicevox_synthesize(text: str, out_wav: Path, speaker: int = VOICEVOX_SPEAKER_ID, timeout: int = 30) -> bool:
    try:
        out_wav.parent.mkdir(parents=True, exist_ok=True)
        q_url = f"{VOICEVOX_URL.rstrip('/')}/audio_query"
        s_url = f"{VOICEVOX_URL.rstrip('/')}/synthesis"
        params = {"text": text, "speaker": speaker}
        r = requests.post(q_url, params=params, timeout=10)
        r.raise_for_status()
        audio_query = r.json()
        synth = requests.post(s_url, params={"speaker": speaker}, json=audio_query, timeout=timeout)
        synth.raise_for_status()
        with open(out_wav, "wb") as f:
            f.write(synth.content)
        logging.info(f"VOICEVOX synthesized to {out_wav.name}")
        return True
    except Exception as e:
        logging.error(f"VOICEVOX synth error (speaker={speaker}) for text start '{text[:30]}...': {e}")
        return False

# ------------------ Anki: AnkiConnect helpers ------------------
ANKI_CONNECT_URL = "http://127.0.0.1:8765"

def try_anki_connect(notes: List[Dict]) -> bool:
    if not notes:
        return False
    payload = {"action": "addNotes", "version": 6, "params": {"notes": notes}}
    try:
        r = requests.post(ANKI_CONNECT_URL, json=payload, timeout=15)
        r.raise_for_status()
        res = r.json()
        if res.get("error"):
            logging.error(f"AnkiConnect error: {res['error']}")
            return False
        return True
    except Exception as e:
        logging.info(f"AnkiConnect not available or failed: {e}")
        return False

def anki_store_media_file(path: Path) -> bool:
    if not path.exists():
        logging.error(f"Media file not found for upload: {path}")
        return False
    try:
        data = path.read_bytes()
        b64 = base64.b64encode(data).decode("ascii")
        payload = {
            "action": "storeMediaFile",
            "version": 6,
            "params": {
                "filename": path.name,
                "data": b64
            }
        }
        r = requests.post(ANKI_CONNECT_URL, json=payload, timeout=20)
        r.raise_for_status()
        res = r.json()
        if res.get("error"):
            logging.error(f"AnkiConnect storeMediaFile error for {path.name}: {res['error']}")
            return False
        return True
    except Exception as e:
        logging.exception(f"Failed to upload media {path.name} to Anki: {e}")
        return False

def export_apkg(notes_for_genanki: List[Dict]) -> Path:
    model = genanki.Model(
        1607392319,
        NOTE_TYPE_NAME,
        fields=[{'name': 'Sentence Audio'}, {'name': 'Word Audio'}, {'name': 'Description Audio'}],
        templates=[{
            'name': 'Card 1',
            'qfmt': '{{Sentence Audio}}<br>{{Word Audio}}',
            'afmt': '{{FrontSide}}<hr id="answer">{{Description Audio}}',
        }]
    )
    deck = genanki.Deck(2059400110, ANKI_DECK_NAME)
    for n in notes_for_genanki:
        note = genanki.Note(
            model=model,
            fields=[
                f"[sound:{n['sentence_audio_file']}]",
                f"[sound:{n['word_audio_file']}]",
                f"[sound:{n['desc_audio_file']}]"
            ],
        )
        deck.add_note(note)
    out = ANKI_FOLDER / f"{ANKI_DECK_NAME}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.apkg"
    genanki.Package(deck).write_to_file(str(out))
    return out

# ------------------ Utilities ------------------
def sanitize_filename(s: str) -> str:
    return re.sub(r'[\\/:*?"<>|]', '_', s)

def is_blacklisted(token: str) -> bool:
    if not token:
        return True
    return token in blacklist

def select_top_words(word_counts: Dict[str, int], limit: int) -> List[str]:
    items = sorted(word_counts.items(), key=lambda x: (-x[1], x[0]))
    return [w for w, c in items][:limit]

# ------------------ MAIN MINING RUN ------------------
def ensure_daily_stats_reset(stats: Dict) -> Dict:
    today = date.today().isoformat()
    if stats.get("last_date") != today:
        stats["last_date"] = today
        stats["added_today"] = 0
    return stats

def find_video_subtitle_pairs() -> List[Tuple[Path, Path]]:
    pairs = []
    for video in sorted(VIDEOS_FOLDER.iterdir()):
        if not video.is_file():
            continue
        if video.suffix.lower() not in VIDEO_EXTS:
            continue
        stem = video.stem
        subtitle_file = None
        for ext in SUBTITLE_EXTS:
            candidate = video.with_suffix(ext)
            if candidate.exists():
                subtitle_file = candidate
                break
        if subtitle_file:
            pairs.append((video, subtitle_file))
    return pairs

def unique_name(base: str, ext: str = ".mp3") -> str:
    ts = datetime.now().strftime("%Y%m%d%H%M%S")
    uid = uuid.uuid4().hex[:8]
    return f"{sanitize_filename(base)}_{ts}_{uid}{ext}"

def mine_all_videos():
    if TOGGLE_FILE.read_text(encoding="utf-8").strip().upper() != "ON":
        logging.info("Toggle is OFF. Set toggle.txt to 'ON' to run mining.")
        return

    logging.info("Mining started.")
    backlog_data = read_backlog_txt()
    stats = read_stats()
    stats = ensure_daily_stats_reset(stats)

    candidate_counts: Dict[str, int] = {}
    candidate_sentences: Dict[str, List[Tuple[Path, float, float, str, int]]] = {}
    processed_videos = 0

    pairs = find_video_subtitle_pairs()
    if not pairs:
        logging.info("No video-subtitle pairs found in Videos/ folder.")
        return

    for video, subtitle in pairs:
        processed_videos += 1
        logging.info(f"Processing video: {video.name} with subtitles {subtitle.name}")
        blocks = parse_subtitle_file(subtitle)
        for idx, start_s, end_s, sentence in blocks:
            if not sentence.strip():
                continue
            # iterate morphemes and use POS & cleaning filtering to avoid proper nouns and particles
            for m in tokenizer_obj.tokenize(sentence, SPLIT_MODE):
                # cleaned candidate
                if not should_keep_token(m):
                    continue

                # Obtain canonical lemma / dictionary form with safe fallbacks
                lemma = None
                if hasattr(m, "dictionary_form"):
                    try:
                        lemma = m.dictionary_form()
                    except Exception:
                        lemma = None
                if not lemma or lemma == "" or lemma == "*":
                    if hasattr(m, "normalized_form"):
                        try:
                            lemma = m.normalized_form()
                        except Exception:
                            lemma = None
                if not lemma:
                    lemma = m.surface()

                tok_clean = clean_surface(lemma)
                if not tok_clean:
                    continue

                # skip if blacklisted/backlog
                if tok_clean in blacklist:
                    continue
                meta = backlog_data.get(tok_clean)
                if meta and meta.get("added"):
                    continue
                candidate_counts[tok_clean] = candidate_counts.get(tok_clean, 0) + 1
                candidate_sentences.setdefault(tok_clean, []).append((video, start_s, end_s, sentence, idx))

    if processed_videos == 0:
        logging.info("No videos processed this run. Exiting.")
        return
    if not candidate_counts:
        logging.info("No candidate words found after filtering. Exiting.")
        return

    remaining = DAILY_CARD_LIMIT - stats.get("added_today", 0)
    if remaining <= 0:
        logging.info("Daily card limit already reached.")
        return

    top_candidates = select_top_words(candidate_counts, remaining * 3)
    chosen = []
    for w in top_candidates:
        if w in candidate_sentences:
            chosen.append(w)
        if len(chosen) >= remaining:
            break

    if not chosen:
        logging.info("No chosen candidates after mapping. Exiting.")
        return

    logging.info(f"Chosen words ({len(chosen)}): {chosen}")

    # initialize OpenAI
    try:
        init_openai()
    except Exception as e:
        logging.error(f"OpenAI init failed: {e}")

    notes_for_genanki = []
    anki_connect_notes = []
    added_this_run = 0

    # Prepare cards and upload media uniquely per note
    for word in chosen:
        try:
            examples = candidate_sentences.get(word, [])
            if not examples:
                continue
            examples.sort(key=lambda e: (-len(e[3]), e[1]))
            video_path, start_s, end_s, sentence_text, srt_idx = examples[0]
            safe_word = sanitize_filename(word)

            canonical_sentence_wav = SENTENCE_AUDIO_FOLDER / f"{video_path.stem}_{srt_idx}_{safe_word}.wav"
            if not canonical_sentence_wav.exists():
                ok = clip_sentence_audio(video_path, start_s, end_s, canonical_sentence_wav)
                if not ok:
                    logging.warning(f"Failed to clip sentence audio for '{word}'. Skipping.")
                    continue

            sentence_media_name = unique_name(f"{video_path.stem}_{srt_idx}_{safe_word}", ".mp3")
            sentence_media_path = SENTENCE_AUDIO_FOLDER / sentence_media_name
            if not sentence_media_path.exists():
                try:
                    subprocess.run([
                        FFMPEG_CMD, "-y", "-i", str(canonical_sentence_wav), "-vn", "-ac", "1", "-ar", "22050", str(sentence_media_path)
                    ], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, check=True)
                except Exception:
                    sentence_media_path = SENTENCE_AUDIO_FOLDER / unique_name(f"{video_path.stem}_{srt_idx}_{safe_word}", ".wav")
                    if not sentence_media_path.exists():
                        sentence_media_path.write_bytes(canonical_sentence_wav.read_bytes())

            try:
                description_text = get_japanese_description(word)
            except Exception as e:
                logging.error(f"OpenAI description failed for '{word}': {e}")
                continue

            canonical_word_wav = WORD_AUDIO_FOLDER / f"{safe_word}_word.wav"
            if not canonical_word_wav.exists():
                voicevox_synthesize(word, canonical_word_wav, speaker=VOICEVOX_SPEAKER_ID)

            word_media_name = unique_name(f"{safe_word}_word", ".mp3")
            word_media_path = WORD_AUDIO_FOLDER / word_media_name
            if not word_media_path.exists() and canonical_word_wav.exists():
                try:
                    subprocess.run([
                        FFMPEG_CMD, "-y", "-i", str(canonical_word_wav), "-vn", "-ac", "1", "-ar", "22050", str(word_media_path)
                    ], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, check=True)
                except Exception:
                    word_media_path = WORD_AUDIO_FOLDER / unique_name(f"{safe_word}_word", ".wav")
                    if not word_media_path.exists():
                        word_media_path.write_bytes(canonical_word_wav.read_bytes())

            canonical_desc_wav = DEF_AUDIO_FOLDER / f"{safe_word}_def.wav"
            if not canonical_desc_wav.exists():
                voicevox_synthesize(description_text, canonical_desc_wav, speaker=VOICEVOX_SPEAKER_ID)

            desc_media_name = unique_name(f"{safe_word}_def", ".mp3")
            desc_media_path = DEF_AUDIO_FOLDER / desc_media_name
            if not desc_media_path.exists() and canonical_desc_wav.exists():
                try:
                    subprocess.run([
                        FFMPEG_CMD, "-y", "-i", str(canonical_desc_wav), "-vn", "-ac", "1", "-ar", "22050", str(desc_media_path)
                    ], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, check=True)
                except Exception:
                    desc_media_path = DEF_AUDIO_FOLDER / unique_name(f"{safe_word}_def", ".wav")
                    if not desc_media_path.exists():
                        desc_media_path.write_bytes(canonical_desc_wav.read_bytes())

            for media_path in (sentence_media_path, word_media_path, desc_media_path):
                if media_path and media_path.exists():
                    ok = anki_store_media_file(media_path)
                    if not ok:
                        logging.warning(f"Failed to upload media to Anki: {media_path.name}")
                    time.sleep(0.12)

            fields = {
                "Sentence Audio": f"[sound:{sentence_media_path.name}]",
                "Word Audio": f"[sound:{word_media_path.name}]" if word_media_path.exists() else "",
                "Description Audio": f"[sound:{desc_media_path.name}]" if desc_media_path.exists() else ""
            }
            anki_note = {
                "deckName": ANKI_DECK_NAME,
                "modelName": NOTE_TYPE_NAME,
                "fields": fields,
                "options": {"allowDuplicate": False},
                "tags": [datetime.now().strftime("%Y-%m-%d")]
            }
            anki_connect_notes.append(anki_note)
            notes_for_genanki.append({
                "sentence_audio_file": sentence_media_path.name,
                "word_audio_file": word_media_path.name if word_media_path.exists() else "",
                "desc_audio_file": desc_media_path.name if desc_media_path.exists() else "",
                "tags": [datetime.now().strftime("%Y-%m-%d")]
            })

            prev = backlog_data.get(word, {})
            backlog_data[word] = {
                "count": candidate_counts.get(word, prev.get("count", 0)),
                "first_seen": prev.get("first_seen", datetime.now().isoformat()),
                "added": False
            }

            logging.info(f"Prepared card for '{word}': sentence={sentence_media_path.name}, word={word_media_path.name if word_media_path.exists() else ''}, desc={desc_media_path.name if desc_media_path.exists() else ''}")
            time.sleep(0.25)

        except Exception as e:
            logging.exception(f"Error preparing card for '{word}': {e}")
            continue

    # Try adding via AnkiConnect
    added_via_connect = False
    if anki_connect_notes:
        added_via_connect = try_anki_connect(anki_connect_notes)

    if added_via_connect:
        logging.info("Added notes to Anki via AnkiConnect.")
        for w in chosen:
            if w in backlog_data:
                backlog_data[w]["added"] = True
        added_this_run = len(anki_connect_notes)
        write_backlog_txt(backlog_data)
    else:
        apkg_path = export_apkg(notes_for_genanki)
        logging.info(f"APKG created at: {apkg_path}. Import into Anki to add cards.")
        for w in chosen:
            prev = backlog_data.get(w, {})
            backlog_data[w] = {
                "count": candidate_counts.get(w, prev.get("count", 0)),
                "first_seen": prev.get("first_seen", datetime.now().isoformat()),
                "added": True
            }
        write_backlog_txt(backlog_data)
        added_this_run = len(notes_for_genanki)

    stats["added_today"] = stats.get("added_today", 0) + added_this_run
    stats["last_date"] = date.today().isoformat()
    write_stats(stats)

    logging.info(f"Mining run complete. Added {added_this_run} cards this run. Daily total now: {stats['added_today']}.")

# ------------------ ENTRYPOINT ------------------
if __name__ == "__main__":
    logging.info("Auto-miner starting.")
    try:
        mine_all_videos()
    except Exception as e:
        logging.exception(f"Unhandled error in mining run: {e}")
