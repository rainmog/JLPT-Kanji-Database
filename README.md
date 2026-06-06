# JLPT Kanji Database

Open-source SQLite database for JLPT kanji study. Contains **2,230 kanji** (N5–N1) with example sentences, vocabulary, kana, and practice questions.

## Download

| File | Size | Description |
|------|------|-------------|
| [kanji.db](https://github.com/rainmog/JLPT-Kanji-Database/releases/download/v1.0/kanji.db) | ~22 MB | SQLite3 database |

## Contents

| Table | Rows | Description |
|-------|------|-------------|
| `kanji` | 2,230 | Kanji with readings, meaning, stroke count, JLPT level |
| `sentences` | 14,625 | Example sentences across 9 difficulty levels |
| `vocabulary` | 8,254 | Vocabulary words with readings and meanings |
| `jlpt_questions` | 568 | Practice questions |
| `kana` | 223 | Hiragana/katakana reference |
| `kana_words` | 131 | Kana vocabulary |

### Kanji by level

| JLPT | Count |
|------|-------|
| N5 | 80 |
| N4 | 166 |
| N3 | 367 |
| N2 | 373 |
| N1 | 1,244 |

### Sentences by difficulty

Each kanji has up to 9 example sentences. Difficulty 1–3 = basic (short, common grammar). Difficulty 4–9 = progressive complexity up to formal/academic Japanese.

| Difficulty | Sentences |
|-----------|-----------|
| 1 | 1,615 |
| 2 | 1,633 |
| 3 | 1,623 |
| 4–9 | ~1,626 each |

## Schema

```sql
CREATE TABLE kanji (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    character     TEXT UNIQUE NOT NULL,
    jlpt_level    INTEGER NOT NULL,        -- 1=N5, 2=N4, 3=N3, 4=N2, 5=N1
    on_reading    TEXT,                    -- comma-separated on'yomi
    kun_reading   TEXT,                    -- comma-separated kun'yomi
    meaning       TEXT,                    -- English meaning(s)
    stroke_count  INTEGER
);

CREATE TABLE sentences (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    kanji_id            INTEGER NOT NULL REFERENCES kanji(id),
    difficulty          INTEGER NOT NULL,  -- 1–9
    text_kanji          TEXT NOT NULL,     -- plain sentence text
    text_structured     TEXT NOT NULL,     -- JSON token array (see below)
    english_translation TEXT NOT NULL,
    valid_readings      TEXT NOT NULL      -- JSON array of hiragana readings for target kanji
);

CREATE TABLE vocabulary (
    id                 INTEGER PRIMARY KEY AUTOINCREMENT,
    word               TEXT NOT NULL,
    reading            TEXT NOT NULL,      -- hiragana
    meanings           TEXT NOT NULL,      -- JSON array
    acceptable_answers TEXT NOT NULL,      -- JSON array of accepted inputs
    jlpt_level         INTEGER NOT NULL,
    tags               TEXT NOT NULL       -- JSON array
);
```

### `text_structured` token format

Each sentence's `text_structured` column is a JSON array of token objects:

```json
[
  { "surface": "安い",  "reading": "やすい", "is_kanji": true,  "kanji_char": "安" },
  { "surface": "です",  "reading": "です",   "is_kanji": false, "kanji_char": null }
]
```

- `surface` — the text as it appears in the sentence
- `reading` — hiragana pronunciation
- `is_kanji` — `true` if this token contains the target kanji
- `kanji_char` — the specific target kanji character (null for non-kanji tokens)

## Usage

```python
import sqlite3, json

conn = sqlite3.connect("kanji.db")

# Get all N1 kanji with sentences
rows = conn.execute("""
    SELECT k.character, k.meaning, s.difficulty, s.text_kanji, s.english_translation
    FROM kanji k
    JOIN sentences s ON s.kanji_id = k.id
    WHERE k.jlpt_level = 5
    ORDER BY k.character, s.difficulty
""").fetchall()

# Parse token structure
sentence = conn.execute("SELECT text_structured FROM sentences LIMIT 1").fetchone()[0]
tokens = json.loads(sentence)
for tok in tokens:
    if tok["is_kanji"]:
        print(f"{tok['surface']} ({tok['reading']})")
```

```js
// Node.js (better-sqlite3)
const Database = require('better-sqlite3');
const db = new Database('kanji.db');

const sentences = db.prepare(`
  SELECT s.text_kanji, s.english_translation, s.valid_readings
  FROM kanji k JOIN sentences s ON s.kanji_id = k.id
  WHERE k.character = ? ORDER BY s.difficulty
`).all('雨');
```

## License

Database content (sentences, translations) is original work released for free educational and research use. See [LICENSE](LICENSE) for details.

Built for the [JLPT Kanji & Vocabulary](https://github.com/rainmog/JLPT-Kanji-And-Vocabulary) Android app.
