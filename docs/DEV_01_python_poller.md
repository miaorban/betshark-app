# BetShark — Fejlesztői dokumentáció #1: Python Data Poller

> **Verzió:** 1.0
> **Utoljára frissítve:** 2026-02-18
> **Célközönség:** Backend/data engineer fejlesztő

---

## 1. Áttekintés

A Python Data Poller a BetShark rendszer adatgyűjtő és elemző komponense. Feladata:

1. **Eseményeket tölt le** egy külső sportfogadási API-ból (csak labdarúgás, csak a konfigurált bajnokságok).
2. **Ütemezi a feldolgozást**: minden eseményt pontosan 1 órával a meccs kezdete előtt dolgoz fel.
3. **LLM-alapú pontozást és elemzést futtat**: minden kimenetre kiszámítja a `likelihood_score`, `odds_score`, `overall_score` értékeket, és generál egy ~10 mondatos szöveges elemzést (magyar és angol nyelven).
4. **PostgreSQL adatbázisba menti** az eredményeket, ahonnan a Node.js backend olvassa ki.
5. **Napi limitet tart**: legfeljebb 400 kimenetet dolgoz fel naponta, bajnokság-prioritás szerint.

A Python komponens **nem foglalkozik felhasználói adatokkal** — azt kizárólag a Node.js backend kezeli.

---

## 2. Tech stack

| Eszköz | Verzió | Leírás |
|---|---|---|
| Python | 3.11+ | Fő futtatókörnyezet |
| `psycopg2-binary` | 2.9.9 | PostgreSQL kapcsolat |
| `APScheduler` | 3.10.4 | Ütemező |
| `httpx` | 0.27.0 | Async HTTP kliens az API hívásokhoz |
| `openai` | 1.30.0 | OpenAI API kliens (LLM hívásokhoz) |
| `python-dotenv` | 1.0.1 | .env fájl betöltése |
| `pydantic` | 2.7.1 | Adatvalidáció és modellek |
| `loguru` | 0.7.2 | Naplózás |
| `tenacity` | 8.3.0 | Újrapróbálkozási logika API hibáknál |

Telepítés:

```bash
pip install psycopg2-binary==2.9.9 apscheduler==3.10.4 httpx==0.27.0 \
  openai==1.30.0 python-dotenv==1.0.1 pydantic==2.7.1 \
  loguru==0.7.2 tenacity==8.3.0
```

---

## 3. Projektstruktúra

```
betshark_poller/
├── .env                        # Környezeti változók (nem kerül verziókezelőbe)
├── .env.example                # Sablon a környezeti változókhoz
├── requirements.txt            # Python függőségek
├── main.py                     # Belépési pont, ütemező indítása
├── config.py                   # Konfiguráció betöltése .env-ből
├── db/
│   ├── __init__.py
│   ├── connection.py           # PostgreSQL kapcsolatkezelés (connection pool)
│   └── queries.py              # SQL lekérdezések és írások
├── fetcher/
│   ├── __init__.py
│   ├── api_client.py           # Külső sportfogadási API hívások
│   └── models.py               # Pydantic modellek az API válaszokhoz
├── scorer/
│   ├── __init__.py
│   ├── scoring.py              # Pontozási logika (score_nopred, score_pred, overall)
│   └── llm_client.py           # OpenAI hívások (elemzés + valószínűség)
├── scheduler/
│   ├── __init__.py
│   └── jobs.py                 # APScheduler job definíciók
└── utils/
    ├── __init__.py
    └── hashing.py              # Segédfüggvények
```

---

## 4. Környezeti változók

Hozd létre a `.env.example` fájlt az alábbi tartalommal, majd másold le `.env` névre és töltsd ki:

```dotenv
# PostgreSQL kapcsolat
DATABASE_URL=postgresql://betshark_user:password@localhost:5432/betshark_db

# Külső sportfogadási API (pl. API-Football, Odds API)
SPORTS_API_KEY=your_sports_api_key_here
SPORTS_API_BASE_URL=https://api.the-odds-api.com/v4

# OpenAI
OPENAI_API_KEY=your_openai_api_key_here
OPENAI_MODEL=gpt-4o

# Napi limit
MAX_OUTCOMES_PER_DAY=400

# Feldolgozás időzítése (percben a kickoff előtt)
PROCESS_BEFORE_KICKOFF_MINUTES=60

# Naplózási szint (DEBUG, INFO, WARNING, ERROR)
LOG_LEVEL=INFO

# Időzóna
TIMEZONE=Europe/Budapest
```

---

## 5. Adatbázis beállítása

### 5.1 Megosztott séma

Mind a Python, mind a Node.js komponens ugyanazt a PostgreSQL adatbázist használja. A teljes séma a következő. Ezt egyszer kell lefuttatni, bármely komponens indítása előtt.

```sql
-- Sports data (Python writes, Node.js reads)
CREATE TABLE leagues (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  priority INTEGER NOT NULL,
  country VARCHAR(100),
  external_id VARCHAR(100) UNIQUE
);

CREATE TABLE events (
  id SERIAL PRIMARY KEY,
  external_id VARCHAR(100) UNIQUE,
  home_team VARCHAR(255) NOT NULL,
  away_team VARCHAR(255) NOT NULL,
  league_id INTEGER REFERENCES leagues(id),
  kickoff_time TIMESTAMPTZ NOT NULL,
  status VARCHAR(50) DEFAULT 'pending',
  processed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE markets (
  id SERIAL PRIMARY KEY,
  event_id INTEGER REFERENCES events(id) ON DELETE CASCADE,
  name_hu VARCHAR(255) NOT NULL,
  name_en VARCHAR(255) NOT NULL,
  external_id VARCHAR(100)
);

CREATE TABLE outcomes (
  id SERIAL PRIMARY KEY,
  market_id INTEGER REFERENCES markets(id) ON DELETE CASCADE,
  name_hu VARCHAR(255) NOT NULL,
  name_en VARCHAR(255) NOT NULL,
  likelihood_score NUMERIC(4,1),
  odds_score NUMERIC(4,1),
  overall_score NUMERIC(4,1),
  best_odds NUMERIC(6,3),
  value_label VARCHAR(20),
  analysis_hu TEXT,
  analysis_en TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- User data (Node.js manages)
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE,
  password_hash VARCHAR(255),
  auth_provider VARCHAR(50) NOT NULL,
  auth_provider_id VARCHAR(255),
  language VARCHAR(10) DEFAULT 'hu',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE subscriptions (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  status VARCHAR(50) NOT NULL,
  platform VARCHAR(20) NOT NULL,
  trial_start TIMESTAMPTZ,
  trial_end TIMESTAMPTZ,
  subscription_start TIMESTAMPTZ,
  subscription_end TIMESTAMPTZ,
  platform_receipt TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE used_trials (
  id SERIAL PRIMARY KEY,
  payment_identifier_hash VARCHAR(255) UNIQUE NOT NULL,
  used_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE password_reset_tokens (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  token VARCHAR(255) UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  used BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE legal_acceptances (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE SET NULL,
  device_id VARCHAR(255),
  version VARCHAR(20) NOT NULL,
  accepted_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 5.2 Bajnokság prioritások kezdeti feltöltése

A `leagues` tábla tartalmazza a feldolgozandó bajnokságokat és azok prioritásait. A kisebb `priority` szám = magasabb fontosság (a legfontosabb bajnokságok kapnak előnyt a napi 400-as limit elosztásánál).

```sql
INSERT INTO leagues (name, priority, country, external_id) VALUES
  ('Premier League', 1, 'England', 'soccer_epl'),
  ('La Liga', 2, 'Spain', 'soccer_spain_la_liga'),
  ('Bundesliga', 3, 'Germany', 'soccer_germany_bundesliga'),
  ('Serie A', 4, 'Italy', 'soccer_italy_serie_a'),
  ('Ligue 1', 5, 'France', 'soccer_france_ligue_one'),
  ('Champions League', 6, 'Europe', 'soccer_uefa_champs_league'),
  ('Europa League', 7, 'Europe', 'soccer_uefa_europa_league'),
  ('Eredivisie', 8, 'Netherlands', 'soccer_netherlands_eredivisie'),
  ('Primeira Liga', 9, 'Portugal', 'soccer_portugal_primeira_liga'),
  ('OTP Bank Liga', 10, 'Hungary', 'soccer_hungary_nb_i');
```

### 5.3 Migrációs lépések

1. Hozd létre az adatbázist és a felhasználót:

```bash
psql -U postgres -c "CREATE USER betshark_user WITH PASSWORD 'erős_jelszó';"
psql -U postgres -c "CREATE DATABASE betshark_db OWNER betshark_user;"
```

2. Futtasd a sémát:

```bash
psql -U betshark_user -d betshark_db -f schema.sql
```

3. Töltsd fel a bajnokságokat:

```bash
psql -U betshark_user -d betshark_db -f seed_leagues.sql
```

---

## 6. Lépésről lépésre implementáció

### 6.1 lépés: Konfiguráció betöltése (`config.py`)

**Mit hozz létre:** `config.py` a projekt gyökerében.

**Mit csináljon:** Betölti a `.env` fájlt, és egy `Settings` Pydantic modellbe gyűjti az összes konfigurációs értéket. Indításkor validálja, hogy minden kötelező érték megvan.

```python
# config.py
from pydantic_settings import BaseSettings
from typing import ClassVar

class Settings(BaseSettings):
    database_url: str
    sports_api_key: str
    sports_api_base_url: str
    openai_api_key: str
    openai_model: str = "gpt-4o"
    max_outcomes_per_day: int = 400
    process_before_kickoff_minutes: int = 60
    log_level: str = "INFO"
    timezone: str = "Europe/Budapest"

    class Config:
        env_file = ".env"

settings = Settings()
```

**Megjegyzés:** Ehhez `pydantic-settings` csomag szükséges (`pip install pydantic-settings`).

---

### 6.2 lépés: Adatbázis kapcsolatkezelés (`db/connection.py`)

**Mit hozz létre:** `db/connection.py`

**Mit csináljon:** Egy `psycopg2` connection pool-t kezel. Biztosítja, hogy a párhuzamos job-ok is stabil DB kapcsolatot kapjanak.

```python
# db/connection.py
import psycopg2
from psycopg2 import pool
from config import settings
from loguru import logger

_pool = None

def init_pool():
    global _pool
    _pool = psycopg2.pool.ThreadedConnectionPool(
        minconn=1,
        maxconn=10,
        dsn=settings.database_url
    )
    logger.info("Database connection pool initialized.")

def get_connection():
    if _pool is None:
        raise RuntimeError("Connection pool is not initialized. Call init_pool() first.")
    return _pool.getconn()

def release_connection(conn):
    if _pool:
        _pool.putconn(conn)

def close_pool():
    if _pool:
        _pool.closeall()
        logger.info("Database connection pool closed.")
```

**Megjegyzés:** A `get_connection()` / `release_connection()` párost mindig `try/finally` blokkban használd, hogy kapcsolathiba esetén se maradjon nyitva a kapcsolat.

---

### 6.3 lépés: SQL lekérdezések és írások (`db/queries.py`)

**Mit hozz létre:** `db/queries.py`

**Mit csináljon:** Tartalmazza az összes adatbázis-műveletet: bajnokságok lekérdezése, esemény mentése/frissítése, piacok és kimenetlek mentése, napi kimenetel-számláló.

**Legfontosabb függvények:**

- `get_leagues_by_priority()` — visszaadja a bajnokságokat prioritás szerint rendezve
- `upsert_event(external_id, home_team, away_team, league_id, kickoff_time)` — ha már létezik az esemény, frissíti, ha nem, létrehozza
- `get_pending_events_for_processing()` — visszaadja az eseményeket, amelyek `status = 'pending'` és `kickoff_time <= NOW() + 60 perc`
- `mark_event_processed(event_id)` — beállítja `status = 'processed'`, `processed_at = NOW()`
- `save_market_and_outcomes(event_id, market_data)` — elmenti a piacot és a hozzá tartozó kimeneteleket
- `get_daily_outcome_count()` — visszaadja, hány kimenetet mentettünk ma (UTC alapján)

```python
# db/queries.py — részlet: get_pending_events_for_processing
def get_pending_events_for_processing(conn, minutes_before: int = 60):
    with conn.cursor() as cur:
        cur.execute("""
            SELECT e.id, e.external_id, e.home_team, e.away_team,
                   e.kickoff_time, l.external_id AS league_external_id,
                   l.priority
            FROM events e
            JOIN leagues l ON e.league_id = l.id
            WHERE e.status = 'pending'
              AND e.kickoff_time <= NOW() + INTERVAL '%s minutes'
              AND e.kickoff_time > NOW()
            ORDER BY l.priority ASC, e.kickoff_time ASC
        """, (minutes_before,))
        return cur.fetchall()
```

```python
# db/queries.py — részlet: upsert_event
def upsert_event(conn, external_id, home_team, away_team, league_id, kickoff_time):
    with conn.cursor() as cur:
        cur.execute("""
            INSERT INTO events (external_id, home_team, away_team, league_id, kickoff_time)
            VALUES (%s, %s, %s, %s, %s)
            ON CONFLICT (external_id) DO UPDATE
              SET home_team = EXCLUDED.home_team,
                  away_team = EXCLUDED.away_team,
                  kickoff_time = EXCLUDED.kickoff_time
            RETURNING id
        """, (external_id, home_team, away_team, league_id, kickoff_time))
        conn.commit()
        return cur.fetchone()[0]
```

```python
# db/queries.py — részlet: save_market_and_outcomes
def save_market_and_outcomes(conn, event_id: int, market: dict):
    """
    market = {
      'name_hu': str, 'name_en': str, 'external_id': str,
      'outcomes': [
        { 'name_hu': str, 'name_en': str, 'likelihood_score': float,
          'odds_score': float, 'overall_score': float, 'best_odds': float,
          'value_label': str, 'analysis_hu': str, 'analysis_en': str }
      ]
    }
    """
    with conn.cursor() as cur:
        cur.execute("""
            INSERT INTO markets (event_id, name_hu, name_en, external_id)
            VALUES (%s, %s, %s, %s) RETURNING id
        """, (event_id, market['name_hu'], market['name_en'], market.get('external_id')))
        market_id = cur.fetchone()[0]

        for o in market['outcomes']:
            cur.execute("""
                INSERT INTO outcomes
                  (market_id, name_hu, name_en, likelihood_score, odds_score,
                   overall_score, best_odds, value_label, analysis_hu, analysis_en)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, (
                market_id, o['name_hu'], o['name_en'],
                o['likelihood_score'], o['odds_score'], o['overall_score'],
                o['best_odds'], o.get('value_label'), o['analysis_hu'], o['analysis_en']
            ))
        conn.commit()
```

```python
# db/queries.py — részlet: get_daily_outcome_count
def get_daily_outcome_count(conn) -> int:
    with conn.cursor() as cur:
        cur.execute("""
            SELECT COUNT(*) FROM outcomes
            WHERE created_at >= CURRENT_DATE
        """)
        return cur.fetchone()[0]
```

---

### 6.4 lépés: API kliens modellek (`fetcher/models.py`)

**Mit hozz létre:** `fetcher/models.py`

**Mit csináljon:** Pydantic modellek, amelyek az API válasz struktúráját írják le. Ezek biztosítják, hogy a nyers API adat validált és típusos Python objektumként kerüljön feldolgozásra.

```python
# fetcher/models.py
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class OddsOutcome(BaseModel):
    name: str           # pl. "Home", "Away", "Draw"
    price: float        # az odds értéke

class OddsMarket(BaseModel):
    key: str            # pl. "h2h", "totals", "btts"
    last_update: datetime
    outcomes: list[OddsOutcome]

class BookmakerOdds(BaseModel):
    key: str
    title: str
    markets: list[OddsMarket]

class SportEvent(BaseModel):
    id: str                         # external_id
    sport_key: str
    sport_title: str
    commence_time: datetime         # kickoff_time (UTC)
    home_team: str
    away_team: str
    bookmakers: list[BookmakerOdds]
```

---

### 6.5 lépés: Külső API kliens (`fetcher/api_client.py`)

**Mit hozz létre:** `fetcher/api_client.py`

**Mit csináljon:**
1. Lekéri az összes közelgő futball eseményt a konfigurált bajnokságokhoz (`sport_key` lista a leagues táblából).
2. Minden eseménynek lekéri az odds-ait (best_odds meghatározásához).
3. A Tenacity könyvtárral automatikusan újrapróbálkozik hálózati hiba esetén (max 3-szor, exponenciális visszalépéssel).

```python
# fetcher/api_client.py
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential
from loguru import logger
from config import settings
from fetcher.models import SportEvent

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=10))
async def fetch_events_for_league(league_external_id: str) -> list[SportEvent]:
    """
    Lekéri az adott bajnokság közelgő eseményeit az API-ból, odds-okkal együtt.
    """
    url = f"{settings.sports_api_base_url}/sports/{league_external_id}/odds"
    params = {
        "apiKey": settings.sports_api_key,
        "regions": "eu",
        "markets": "h2h,btts,totals",
        "oddsFormat": "decimal",
        "dateFormat": "iso"
    }
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = client.get(url, params=params)
        response.raise_for_status()
        data = response.json()
        logger.info(f"Fetched {len(data)} events for league {league_external_id}")
        return [SportEvent(**event) for event in data]

def get_best_odds(event: SportEvent, market_key: str, outcome_name: str) -> float | None:
    """
    Az összes bookmaker között megkeresi a legjobb (legmagasabb) odds-ot
    egy adott piac és kimenetel kombinációra.
    """
    best = None
    for bookmaker in event.bookmakers:
        for market in bookmaker.markets:
            if market.key != market_key:
                continue
            for outcome in market.outcomes:
                if outcome.name == outcome_name:
                    if best is None or outcome.price > best:
                        best = outcome.price
    return best
```

**Megjegyzés:** Az odds API-tól függően a `regions`, `markets` paraméterek változhatnak. Az API dokumentációját mindig ellenőrizd. A fenti példa az The Odds API v4 formátumát követi.

---

### 6.6 lépés: Pontozási logika (`scorer/scoring.py`)

**Mit hozz létre:** `scorer/scoring.py`

**Mit csináljon:** Implementálja a BetShark pontozási formulákat. Ez a modul **csak tiszta matematika**, nem hív semmilyen API-t.

**Pontozási formulák:**

- `score_nopred = (likelihood_nopred × 9 + value_nopred × 1) / 10`
- `score_pred = (likelihood_pred × 9 + value_pred × 1) / 10`
- `overall = (score_nopred × 7 + score_pred × 3) / 10`
- **Büntetés:** ha `overall >= 7` ÉS `best_odds < 1.5`: `overall -= 2`
- **Value:** `(100 / likelihood) < best_odds <= (100 / likelihood) + 0.2`
- **Strong Value:** `best_odds > (100 / likelihood) + 0.2`

```python
# scorer/scoring.py

def calculate_value_label(likelihood_percent: float, best_odds: float) -> str | None:
    """
    Meghatározza a value besorolást.
    likelihood_percent: 0-100 közötti szám (pl. 55.0 = 55%)
    best_odds: decimal odds (pl. 1.75)
    """
    if likelihood_percent <= 0:
        return None
    fair_odds = 100.0 / likelihood_percent
    if best_odds > fair_odds + 0.2:
        return "strong_value"
    elif best_odds > fair_odds:
        return "value"
    return None

def calculate_scores(
    likelihood_nopred: float,
    value_nopred: float,
    likelihood_pred: float,
    value_pred: float,
    best_odds: float
) -> dict:
    """
    Kiszámítja a likelihood_score, odds_score és overall_score értékeket.

    Paraméterek (mind 0-10 skálán):
      likelihood_nopred: valószínűségi pontszám előrejelzés nélkül
      value_nopred:      odds értéke előrejelzés nélkül
      likelihood_pred:   valószínűségi pontszám előrejelzéssel
      value_pred:        odds értéke előrejelzéssel

    Visszatér: dict likelihood_score, odds_score, overall_score kulcsokkal
    """
    score_nopred = (likelihood_nopred * 9 + value_nopred * 1) / 10
    score_pred   = (likelihood_pred   * 9 + value_pred   * 1) / 10
    overall      = (score_nopred * 7 + score_pred * 3) / 10

    # Büntetés: magas pontszám, de túl alacsony odds
    if overall >= 7.0 and best_odds < 1.5:
        overall -= 2.0

    # Kerekítés 1 tizedesre
    return {
        "likelihood_score": round(score_nopred, 1),
        "odds_score":       round(score_pred, 1),
        "overall_score":    round(overall, 1)
    }
```

**Megjegyzés:** A `likelihood_nopred`, `value_nopred`, `likelihood_pred`, `value_pred` értékeket az LLM adja vissza (ld. következő lépés). Az LLM-nek 0-10 skálán kell visszaadnia ezeket.

---

### 6.7 lépés: LLM kliens (`scorer/llm_client.py`)

**Mit hozz létre:** `scorer/llm_client.py`

**Mit csináljon:**
1. Meghívja az OpenAI API-t egy strukturált prompttal.
2. A promptban megadja az esemény adatait (csapatok, bajnokság, piac, kimenetel, odds).
3. Az LLM visszaad: `likelihood_nopred`, `value_nopred`, `likelihood_pred`, `value_pred` (mind 0-10 float), `analysis_hu` és `analysis_en` (~10 mondatos szöveges elemzés).
4. A válasz JSON formátumú (OpenAI function calling vagy JSON mode segítségével).

```python
# scorer/llm_client.py
import json
from openai import OpenAI
from tenacity import retry, stop_after_attempt, wait_exponential
from loguru import logger
from config import settings

client = OpenAI(api_key=settings.openai_api_key)

SYSTEM_PROMPT = """
You are a professional football betting analyst. Your task is to evaluate a specific betting outcome.
You will receive:
- Match details (teams, league, kickoff time)
- The betting market and outcome being evaluated
- The best available decimal odds

Return a JSON object with exactly these fields:
{
  "likelihood_nopred": <float 0-10, how likely the outcome is based on statistical data alone>,
  "value_nopred": <float 0-10, how good the odds are vs statistical probability>,
  "likelihood_pred": <float 0-10, how likely based on your expert prediction>,
  "value_pred": <float 0-10, how good the odds are vs your predicted probability>,
  "analysis_hu": "<~10 sentence analysis in Hungarian, professional tone>",
  "analysis_en": "<~10 sentence analysis in English, professional tone>"
}

Be objective and data-driven. The analysis should mention team form, head-to-head record,
injuries if known, and the quality of the odds. Do NOT recommend placing bets.
"""

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=20))
def analyze_outcome(
    home_team: str,
    away_team: str,
    league: str,
    kickoff_time: str,
    market_name_en: str,
    outcome_name_en: str,
    best_odds: float
) -> dict:
    """
    LLM-mel elemez egy fogadási kimenetet. Visszaad egy dict-et a pontszámokkal és elemzésekkel.
    """
    user_message = f"""
Match: {home_team} vs {away_team}
League: {league}
Kickoff: {kickoff_time}
Market: {market_name_en}
Outcome: {outcome_name_en}
Best available odds: {best_odds}

Please analyze this betting outcome and return the JSON response.
"""
    logger.debug(f"Calling LLM for: {home_team} vs {away_team} | {market_name_en}: {outcome_name_en}")

    response = client.chat.completions.create(
        model=settings.openai_model,
        response_format={"type": "json_object"},
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user",   "content": user_message}
        ],
        temperature=0.3
    )

    result = json.loads(response.choices[0].message.content)

    # Validáció: minden kötelező mező legyen jelen
    required_keys = ["likelihood_nopred", "value_nopred", "likelihood_pred",
                     "value_pred", "analysis_hu", "analysis_en"]
    for key in required_keys:
        if key not in result:
            raise ValueError(f"LLM response missing required key: {key}")

    logger.info(f"LLM analysis complete for {outcome_name_en} ({home_team} vs {away_team})")
    return result
```

**Megjegyzés:** A `response_format={"type": "json_object"}` csak a GPT-4o és GPT-4-turbo modellekkel működik megbízhatóan. Mindig validáld a visszakapott dict struktúráját, mert az LLM elvben nem köteles a sémát betartani.

---

### 6.8 lépés: Piaci névfordítás

**Mit hozz létre:** `fetcher/market_names.py`

**Mit csináljon:** Egy szótár, amely az API által visszaadott piac- és kimenetel-kulcsokat lefordítja magyar és angol felhasználóbarát nevekre.

```python
# fetcher/market_names.py

MARKET_NAMES = {
    "h2h": {"hu": "Végeredmény (1X2)", "en": "Match Result (1X2)"},
    "btts": {"hu": "Mindkét csapat szerez gólt", "en": "Both Teams to Score"},
    "totals": {"hu": "Összesített gólok", "en": "Total Goals"},
    "draw_no_bet": {"hu": "Döntetlen visszatérítés", "en": "Draw No Bet"},
    "spreads": {"hu": "Hendikep", "en": "Asian Handicap"},
}

OUTCOME_NAMES = {
    # h2h
    "Home": {"hu": "Hazai győzelem", "en": "Home Win"},
    "Away": {"hu": "Vendég győzelem", "en": "Away Win"},
    "Draw": {"hu": "Döntetlen", "en": "Draw"},
    # btts
    "Yes": {"hu": "Igen", "en": "Yes"},
    "No":  {"hu": "Nem", "en": "No"},
    # totals
    "Over 2.5": {"hu": "Több mint 2,5 gól", "en": "Over 2.5"},
    "Under 2.5": {"hu": "Kevesebb mint 2,5 gól", "en": "Under 2.5"},
    "Over 1.5": {"hu": "Több mint 1,5 gól", "en": "Over 1.5"},
    "Under 1.5": {"hu": "Kevesebb mint 1,5 gól", "en": "Under 1.5"},
}

def get_market_names(api_key: str) -> dict:
    return MARKET_NAMES.get(api_key, {"hu": api_key, "en": api_key})

def get_outcome_names(api_name: str) -> dict:
    return OUTCOME_NAMES.get(api_name, {"hu": api_name, "en": api_name})
```

---

### 6.9 lépés: Ütemező job-ok (`scheduler/jobs.py`)

**Mit hozz létre:** `scheduler/jobs.py`

**Mit csináljon:** Két fő ütemezett job-ot definiál:

1. **`fetch_and_store_events`** — naponta egyszer fut (pl. minden reggel 06:00-kor). Letölti az összes közelgő eseményt az összes konfigurált bajnoksághoz, és elmenti őket az `events` táblába `status='pending'` állapottal.

2. **`process_pending_events`** — 15 percenként fut. Megnézi, mely événényeket kell feldolgozni (amelyek `status='pending'` és kickoff <= 60 perc múlva), futtatja az LLM elemzést, és elmenti az eredményeket. Betartja a napi 400 kimenetes limitet.

```python
# scheduler/jobs.py
import asyncio
from datetime import datetime
from loguru import logger
from db.connection import get_connection, release_connection
from db import queries
from fetcher.api_client import fetch_events_for_league, get_best_odds
from fetcher.market_names import get_market_names, get_outcome_names
from scorer.llm_client import analyze_outcome
from scorer.scoring import calculate_scores, calculate_value_label
from config import settings


def fetch_and_store_events():
    """
    Naponta egyszer fut. Letölti az összes bajnokság eseményeit és elmenti a DB-be.
    """
    logger.info("JOB START: fetch_and_store_events")
    conn = get_connection()
    try:
        leagues = queries.get_leagues_by_priority(conn)

        for league in leagues:
            try:
                events = asyncio.run(fetch_events_for_league(league['external_id']))
                for event in events:
                    queries.upsert_event(
                        conn,
                        external_id=event.id,
                        home_team=event.home_team,
                        away_team=event.away_team,
                        league_id=league['id'],
                        kickoff_time=event.commence_time
                    )
                logger.info(f"Stored {len(events)} events for {league['name']}")
            except Exception as e:
                logger.error(f"Failed to fetch events for {league['name']}: {e}")
    finally:
        release_connection(conn)
    logger.info("JOB END: fetch_and_store_events")


def process_pending_events():
    """
    15 percenként fut. Feldolgozza az esedékes eseményeket (LLM + DB mentés).
    Betartja a napi 400 kimenetes limitet.
    """
    logger.info("JOB START: process_pending_events")
    conn = get_connection()
    try:
        daily_count = queries.get_daily_outcome_count(conn)
        remaining = settings.max_outcomes_per_day - daily_count
        if remaining <= 0:
            logger.info(f"Daily outcome limit reached ({settings.max_outcomes_per_day}). Skipping.")
            return

        events = queries.get_pending_events_for_processing(
            conn, minutes_before=settings.process_before_kickoff_minutes
        )
        logger.info(f"Found {len(events)} events to process. Remaining outcome budget: {remaining}")

        for event in events:
            if remaining <= 0:
                logger.info("Outcome budget exhausted mid-run. Stopping.")
                break

            try:
                # Lekéri az esemény aktuális odds-ait
                event_data = asyncio.run(
                    fetch_events_for_league(event['league_external_id'])
                )
                # Megkeresi az adott eseményt az API válaszban
                api_event = next((e for e in event_data if e.id == event['external_id']), None)
                if not api_event:
                    logger.warning(f"Event {event['external_id']} not found in API response. Skipping.")
                    continue

                # Végigmegy a piacokon és kimeneteleken
                processed_markets = set()
                for bookmaker in api_event.bookmakers:
                    for market in bookmaker.markets:
                        market_key = market.key
                        if market_key in processed_markets:
                            continue
                        processed_markets.add(market_key)

                        market_names = get_market_names(market_key)
                        outcomes_data = []

                        for outcome in market.outcomes:
                            if remaining <= 0:
                                break
                            best_odds = get_best_odds(api_event, market_key, outcome.name)
                            if not best_odds:
                                continue

                            outcome_names = get_outcome_names(outcome.name)

                            # LLM elemzés
                            llm_result = analyze_outcome(
                                home_team=event['home_team'],
                                away_team=event['away_team'],
                                league=event['league_external_id'],
                                kickoff_time=str(event['kickoff_time']),
                                market_name_en=market_names['en'],
                                outcome_name_en=outcome_names['en'],
                                best_odds=best_odds
                            )

                            # Pontozás
                            scores = calculate_scores(
                                likelihood_nopred=llm_result['likelihood_nopred'],
                                value_nopred=llm_result['value_nopred'],
                                likelihood_pred=llm_result['likelihood_pred'],
                                value_pred=llm_result['value_pred'],
                                best_odds=best_odds
                            )

                            # Value label
                            # likelihood_nopred 0-10 skálán van, százalékhoz *10
                            likelihood_pct = llm_result['likelihood_nopred'] * 10
                            value_label = calculate_value_label(likelihood_pct, best_odds)

                            outcomes_data.append({
                                "name_hu": outcome_names['hu'],
                                "name_en": outcome_names['en'],
                                **scores,
                                "best_odds": best_odds,
                                "value_label": value_label,
                                "analysis_hu": llm_result['analysis_hu'],
                                "analysis_en": llm_result['analysis_en']
                            })
                            remaining -= 1

                        if outcomes_data:
                            queries.save_market_and_outcomes(conn, event['id'], {
                                "name_hu": market_names['hu'],
                                "name_en": market_names['en'],
                                "external_id": market_key,
                                "outcomes": outcomes_data
                            })

                queries.mark_event_processed(conn, event['id'])
                logger.info(f"Processed event: {event['home_team']} vs {event['away_team']}")

            except Exception as e:
                logger.error(f"Failed to process event {event['external_id']}: {e}")
                # Nem állítjuk le a teljes job-ot egy esemény hibájánál
    finally:
        release_connection(conn)
    logger.info("JOB END: process_pending_events")
```

**Megjegyzés:** Egy esemény feldolgozásakor az odds-okat újra lekérjük az API-tól (nem a reggeli fetchből), hogy a legfrissebb odds-ok kerüljenek az adatbázisba. Ez plusz API hívást jelent, de fontosabb az adatok frissessége.

---

### 6.10 lépés: Belépési pont és ütemező (`main.py`)

**Mit hozz létre:** `main.py` a projekt gyökerében.

**Mit csináljon:** Inicializálja a DB connection pool-t, regisztrálja az APScheduler job-okat, majd blokkolva futtatja az ütemezőt.

```python
# main.py
import signal
import sys
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger
from loguru import logger
from config import settings
from db.connection import init_pool, close_pool
from scheduler.jobs import fetch_and_store_events, process_pending_events

def shutdown(signum, frame):
    logger.info("Shutdown signal received. Closing DB pool...")
    close_pool()
    sys.exit(0)

def main():
    # Naplózás konfigurálása
    logger.remove()
    logger.add(sys.stdout, level=settings.log_level)
    logger.add("logs/poller_{time}.log", rotation="1 day", retention="7 days", level="DEBUG")

    logger.info("BetShark Python Poller starting...")

    # DB inicializálás
    init_pool()

    # Graceful shutdown kezelése
    signal.signal(signal.SIGINT, shutdown)
    signal.signal(signal.SIGTERM, shutdown)

    # Ütemező
    scheduler = BlockingScheduler(timezone=settings.timezone)

    # Esemény letöltés: minden reggel 06:00-kor
    scheduler.add_job(
        fetch_and_store_events,
        trigger=CronTrigger(hour=6, minute=0, timezone=settings.timezone),
        id="fetch_events",
        name="Fetch upcoming events from API",
        replace_existing=True
    )

    # Esemény feldolgozás: 15 percenként
    scheduler.add_job(
        process_pending_events,
        trigger=IntervalTrigger(minutes=15),
        id="process_events",
        name="Process pending events with LLM scoring",
        replace_existing=True,
        max_instances=1  # Fontos: ne fusson párhuzamosan!
    )

    logger.info("Scheduler started. Jobs registered.")
    logger.info("  - fetch_events: daily at 06:00")
    logger.info("  - process_events: every 15 minutes")

    scheduler.start()

if __name__ == "__main__":
    main()
```

**Megjegyzés:** A `max_instances=1` a `process_events` job-nál kritikus! Ha egy LLM-hívás sokáig tart és a következő 15 perces ciklus is elindul, ne futhasson párhuzamosan ugyanaz a job — az megduplázhatja az adatbázis-írásokat.

---

### 6.11 lépés: Naplózás konfigurálása

A `loguru` könyvtár alapértelmezetten stdout-ra ír. A `main.py`-ban fentebb már beállítottuk, de adj hozzá egy `logs/` könyvtárat és gitignore-ba vedd fel:

```bash
mkdir -p betshark_poller/logs
echo "logs/" >> betshark_poller/.gitignore
echo ".env"  >> betshark_poller/.gitignore
```

---

### 6.12 lépés: `requirements.txt` létrehozása

```
psycopg2-binary==2.9.9
APScheduler==3.10.4
httpx==0.27.0
openai==1.30.0
python-dotenv==1.0.1
pydantic==2.7.1
pydantic-settings==2.2.1
loguru==0.7.2
tenacity==8.3.0
```

---

## 7. Futtatás és deployment

### Lokális futtatás

```bash
cd betshark_poller
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# Töltsd ki a .env fájlt!
python main.py
```

### Manuális tesztelés

Az ütemező megvárása helyett a job-okat közvetlenül is meghívhatod tesztelésre:

```bash
python -c "from scheduler.jobs import fetch_and_store_events; fetch_and_store_events()"
python -c "from scheduler.jobs import process_pending_events; process_pending_events()"
```

### Produktív deployment (Linux szerver, systemd)

Hozz létre egy systemd service fájlt `/etc/systemd/system/betshark-poller.service` névvel:

```ini
[Unit]
Description=BetShark Python Poller
After=network.target postgresql.service

[Service]
User=betshark
WorkingDirectory=/opt/betshark_poller
EnvironmentFile=/opt/betshark_poller/.env
ExecStart=/opt/betshark_poller/venv/bin/python main.py
Restart=on-failure
RestartSec=30
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Aktiválás:

```bash
sudo systemctl daemon-reload
sudo systemctl enable betshark-poller
sudo systemctl start betshark-poller
sudo systemctl status betshark-poller
```

### Docker deployment

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

CMD ["python", "main.py"]
```

```bash
docker build -t betshark-poller .
docker run -d --env-file .env --name betshark-poller betshark-poller
```

### Fontos megfontolások

- **API rate limit:** Az Odds API-nak van havi hívásszám-limitje. A napi 400 kimenetes korlát szem előtt tartja ezt, de győződj meg róla, hogy a konfigurált bajnokságok száma és az API tier összhangban van.
- **OpenAI költség:** Minden kimenetelhez egy LLM hívás szükséges. 400 kimenetel/nap GPT-4o-val körülbelül 2-4 USD/nap. Ez monitorozandó.
- **DB tranzakciók:** Minden job-ban `commit()` csak sikeres feldolgozás után hívódik. Ha hiba van, a nem commitált adatok nem kerülnek az adatbázisba.
- **Időzóna:** Az APScheduler a konfigurált `TIMEZONE` értéket használja. Az adatbázisban minden időpont UTC-ben tárolódik (`TIMESTAMPTZ`). A konverziókat mindig explicit végezd el.
- **Skálázás:** Ha a 400 kimenetes limit kevésnek bizonyul, a `MAX_OUTCOMES_PER_DAY` értéket növelheted, de az API és OpenAI költségek arányosan nőnek.
