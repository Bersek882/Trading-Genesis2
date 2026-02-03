# Trading Genesis 2 - Konzept

## Überblick

Selbstverbesserndes High-Frequency Krypto-Trading-System mit dynamischer Strategie-Entdeckung durch Claude Code Agents.

**Kernprinzipien:**
- 50-100+ Trades pro Tag → schnelle Erfolgsmessung
- 3 aktive Champions (Gold, Silver, Bronze) + 2 Challenger-Slots
- Echtzeit-Optimierung und dynamischer Strategie-Austausch
- Keine hardcoded Strategien - alles wird entdeckt und validiert

---

## Strategie-Hierarchie

```
┌─────────────────────────────────────────────────────────────┐
│                  AKTIVE STRATEGIEN                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CHAMPIONS (Live Trading)         CHALLENGERS (Paper)       │
│  ┌───────────────────┐           ┌───────────────────┐     │
│  │ 🥇 GOLD    (50%)  │           │ Challenger 1      │     │
│  ├───────────────────┤           │ (testet sich)     │     │
│  │ 🥈 SILVER  (30%)  │           ├───────────────────┤     │
│  ├───────────────────┤           │ Challenger 2      │     │
│  │ 🥉 BRONZE  (20%)  │           │ (testet sich)     │     │
│  └───────────────────┘           └───────────────────┘     │
│                                                             │
│  WARTESCHLANGE (Ready to Challenge)                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Strategy A │ Strategy B │ Strategy C │ ... (max 5)  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Kapitalverteilung

| Slot | Anteil | Beschreibung |
|------|--------|--------------|
| Gold | 50% | Beste performende Strategie |
| Silver | 30% | Zweitbeste Strategie |
| Bronze | 20% | Drittbeste Strategie |
| Challenger 1 | Paper | Testet gegen Champions |
| Challenger 2 | Paper | Testet gegen Champions |

### Aufstieg und Abstieg

```
Challenger schlägt Bronze (nach max 24h)?
    │
    ├─ JA → Challenger wird Bronze
    │       Bronze wird in Warteschlange (oder verworfen)
    │       Nächste Strategie aus Warteschlange wird Challenger
    │
    └─ NEIN → Challenger wird verworfen
              Nächste Strategie aus Warteschlange wird Challenger

Innerhalb Champions:
    Bronze schlägt Silver? → Tauschen
    Silver schlägt Gold?   → Tauschen
```

---

## System-Architektur (3 Schichten)

```
┌─────────────────────────────────────────────────────────────┐
│                  ECHTZEIT-SCHICHT                           │
│              (läuft kontinuierlich)                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Signal Agent ──► Risk Agent ──► Executor Agent             │
│       │                              │                      │
│       └──────────┬───────────────────┘                      │
│                  ▼                                          │
│           Monitor Agent (streamt jeden Trade)               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│                ANALYSE-SCHICHT                              │
│           (läuft stündlich / alle 4-6h)                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Analyzer Agent ──► Optimizer Agent                         │
│       │                   │                                 │
│       │                   └──► Parameter-Updates (live)     │
│       │                                                     │
│       └──► Champion vs Challenger Vergleich                 │
│            └──► Rang-Änderungen / Hot-Swap                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│              DISCOVERY-SCHICHT                              │
│            (läuft im Hintergrund)                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Research Agent ──► Parser ──► Validator ──► Coder          │
│                                                │            │
│                                                ▼            │
│                                         Backtest Agent      │
│                                                │            │
│                                                ▼            │
│                                         Evaluator Agent     │
│                                                │            │
│                                                ▼            │
│                                    Warteschlange (max 5)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                             │
│              (koordiniert alles)                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  - Verwaltet Champion/Challenger Status                     │
│  - Entscheidet Hot-Swaps (alle 4-6h)                        │
│  - Steuert Discovery-Pipeline                               │
│  - Notfall-Stops bei extremem Drawdown                      │
│  - Verwaltet pending_requirements.json                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Zeitplan (High-Frequency)

```
┌─────────────────────────────────────────────────────────────┐
│                    ZEITPLAN                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  KONTINUIERLICH:                                            │
│  ├─ Signal Agent generiert Signale                          │
│  ├─ Executor führt Trades aus                               │
│  └─ Monitor trackt jeden Trade in Echtzeit                  │
│                                                             │
│  STÜNDLICH:                                                 │
│  └─ Micro-Optimierung (kleine Parameter-Anpassungen)        │
│                                                             │
│  ALLE 4-6 STUNDEN:                                          │
│  ├─ Performance-Vergleich aller aktiven Strategien          │
│  ├─ Champion-Ranking aktualisieren (Gold/Silver/Bronze)     │
│  ├─ Challenger vs Bronze Vergleich                          │
│  └─ Hot-Swap wenn Challenger signifikant besser             │
│                                                             │
│  TÄGLICH:                                                   │
│  ├─ Walk-Forward Reoptimierung aller Champions              │
│  ├─ Vollständiger Performance-Report                        │
│  └─ Discovery-Pipeline: Warteschlange auffüllen             │
│                                                             │
│  WÖCHENTLICH:                                               │
│  └─ Research Agent sucht neue Strategien im Internet        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Statistische Basis

Bei 50-100 Trades/Tag:

| Zeitraum | Trades | Aussagekraft |
|----------|--------|--------------|
| 6h | ~25 | Trend erkennbar |
| 12h | ~50 | Erste Signifikanz |
| 24h | ~100 | Gute Signifikanz |
| 48h | ~200 | Sehr hohe Signifikanz (optional) |

**Challenger-Testzeit:** Maximum 24h (bei 100+ Trades auch früher bewertbar)

---

## Discovery Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                   DISCOVERY PIPELINE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. RESEARCH AGENT                                          │
│     └─ Sucht im Internet: GitHub, TradingView, Papers       │
│                                                             │
│  2. PARSER AGENT                                            │
│     └─ Extrahiert: Entry/Exit, Indikatoren, Timeframes      │
│                                                             │
│  3. VALIDATOR AGENT                                         │
│     ├─ Prüft Vollständigkeit                                │
│     ├─ Prüft Krypto-Eignung                                 │
│     └─ Prüft technische Machbarkeit                         │
│         │                                                   │
│         ├─ OK → weiter zu Coder                             │
│         └─ FEHLT API → pending_requirements.json            │
│                                                             │
│  4. CODER AGENT                                             │
│     └─ Implementiert als Python/Backtrader Code             │
│                                                             │
│  5. BACKTEST AGENT                                          │
│     └─ Walk-Forward Tests mit Slippage/Fees                 │
│                                                             │
│  6. EVALUATOR AGENT                                         │
│     └─ Ranking nach Sharpe, Drawdown, Win Rate              │
│                                                             │
│  7. WARTESCHLANGE (max 5 Strategien)                        │
│     └─ Bereit für Challenger-Slot                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Validator - Technische Machbarkeit

Wenn Datenquellen fehlen:

```
Validator → Orchestrator: "Strategie X braucht Whale Alert API"
Orchestrator → pending_requirements.json: Eintrag
Strategie Status: PENDING_REQUIREMENTS
```

**pending_requirements.json:**
```json
{
  "pending_strategies": [
    {
      "strategy_id": "whale_momentum_01",
      "strategy_name": "Whale Movement Momentum",
      "missing_requirements": [
        {
          "type": "data_source",
          "name": "Whale Alert API",
          "reason": "Benötigt Echtzeit-Whale-Transaktionen",
          "alternatives": ["Arkham API"]
        }
      ],
      "status": "waiting_for_setup"
    }
  ]
}
```

---

## Challenger-Logik

```
┌─────────────────────────────────────────────────────────────┐
│                 CHALLENGER LIFECYCLE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Strategie kommt aus Warteschlange                       │
│     └─ Wird Challenger (Paper Trading)                      │
│                                                             │
│  2. Läuft maximal 24 Stunden                                │
│     └─ Sammelt mindestens 100 Trades                     │
│                                                             │
│  3. Vergleich mit Bronze Champion                           │
│     │                                                       │
│     ├─ Challenger BESSER (Sharpe > Bronze + 0.1)?           │
│     │   └─ JA: Challenger → Bronze                          │
│     │         Bronze → Warteschlange oder verworfen         │
│     │                                                       │
│     └─ Challenger SCHLECHTER?                               │
│         └─ Challenger verworfen                             │
│            Nächste Strategie aus Warteschlange              │
│                                                             │
│  4. Falls Warteschlange leer:                               │
│     └─ Discovery Pipeline wird priorisiert                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Sicherheitsmechanismen

| Regel | Wert |
|-------|------|
| Mindest-Trades vor Vergleich | 50 |
| Mindest-Testzeit | 24h |
| Maximum-Testzeit | 24h |
| Cooldown nach Swap | 6h |
| Max Swaps pro Tag | 3 |
| Rollback-Fenster | 2h nach Swap |

---

## Selbstverbesserung

### Optimizer Agent

Zwei parallele Aufgaben:

**1. Mikro-Optimierung (stündlich):**
- ATR-Multiplier anpassen
- Threshold-Werte tunen
- Reagiert auf kurzfristige Marktänderungen

**2. Makro-Optimierung (täglich):**
- Walk-Forward Reoptimierung
- Größere Parameter-Änderungen
- Nutzt letzte 24-48h Daten

### Analyzer Agent

Kontinuierliche Streaming-Analyse:
- Rolling Sharpe (4h, 12h, 24h Fenster)
- Rolling Win Rate
- Rolling Drawdown
- Erkennt Performance-Drift in Echtzeit
- Triggert Alerts bei Anomalien

---

## Erfolgsmessung (Composite Score)

### Warum nicht nur Return?

Return allein ist gefährlich:
- **Hohe Returns + Hohe Volatilität** = Glück, nicht Skill
- **Moderate Returns + Konsistenz** = Robuste Strategie

### Composite Score Formel

```
Score = (Sortino × 0.4) + (Calmar × 0.3) + (Profit Factor × 0.2) + (Consistency × 0.1)
```

| Metrik | Gewichtung | Warum |
|--------|------------|-------|
| **Sortino Ratio** | 40% | Bestraft nur Downside-Volatilität |
| **Calmar Ratio** | 30% | Return / Max Drawdown |
| **Profit Factor** | 20% | Gross Profit / Gross Loss |
| **Consistency** | 10% | % profitable 4h-Perioden |

### Alpha vs Return

**Return** = Absolute Performance
**Alpha** = Performance ÜBER dem Benchmark

```
Benchmark = Gesamte Krypto-Marktkapitalisierung (24h Änderung)
Alpha = Strategie-Return - Markt-Return

Beispiel:
  Strategie: +5%
  Krypto-Markt: +8%
  Alpha: -3% (SCHLECHT trotz Gewinn!)
```

**Datenquelle:** CoinGecko API (kostenlos)
```bash
curl "https://api.coingecko.com/api/v3/global"
# → data.market_cap_change_percentage_24h_usd
```

**Regel:** Eine Strategie mit positivem Return aber negativem Alpha wird degradiert.

---

## Overfitting-Schutz

### 1. Benchmark-Vergleich

Jede Strategie wird gegen die Gesamte Krypto-Marktcap gemessen:
```
IF strategy_return > 0 AND alpha < 0:
    → Warnung: "Underperforming vs Market"
    → Strategie wird nicht befördert
```

### 2. Regime-Testing

Backtests müssen in ALLEN Marktphasen bestehen:

| Regime | Erkennung | Mindest-Performance |
|--------|-----------|---------------------|
| **Bull** | Markt +10% in 7d | Positiver Alpha |
| **Bear** | Markt -10% in 7d | Geringerer Verlust als Markt |
| **Sideways** | Markt ±5% in 7d | Positiver Return |

```
Strategie besteht nur wenn:
  - Bull-Regime: Alpha > 0
  - Bear-Regime: Drawdown < Markt-Drawdown
  - Sideways: Return > 0
```

### 3. Parameter-Sensitivität

```
Optimizer testet Parameter ±20%:

Original: RSI Period = 14, Threshold = 30
Test 1: RSI Period = 11, Threshold = 30
Test 2: RSI Period = 17, Threshold = 30
Test 3: RSI Period = 14, Threshold = 24
Test 4: RSI Period = 14, Threshold = 36

Wenn Performance stark schwankt → OVERFIT!
```

**Stabile Strategie:** Kleine Parameter-Änderungen → Kleine Performance-Änderungen

---

## Degradation Monitoring

### Rolling Metrics (Echtzeit)

```
┌─────────────────────────────────────────────────────────────┐
│              ROLLING WINDOWS                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Window    │ Aktualisierung │ Verwendet für                │
│  ──────────┼────────────────┼───────────────────────────── │
│  1h        │ Jede Minute    │ Anomalie-Erkennung           │
│  4h        │ Alle 5 Min     │ Micro-Optimierung            │
│  12h       │ Alle 15 Min    │ Trend-Erkennung              │
│  24h       │ Stündlich      │ Champion-Vergleich           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Alert-Stufen

| Stufe | Trigger | Aktion |
|-------|---------|--------|
| **INFO** | Sharpe fällt um 10% | Logging |
| **WARNING** | Sharpe fällt um 25% | Notification |
| **CRITICAL** | Sharpe fällt um 50% | Auto-Pause + Review |
| **EMERGENCY** | Drawdown > 15% | Sofortiger Stop |

### Auto-Response

```
┌─────────────────────────────────────────────────────────────┐
│              DEGRADATION RESPONSE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Degradation erkannt (Sharpe -25% über 4h)                  │
│                 │                                           │
│                 ▼                                           │
│  ┌─────────────────────────────┐                            │
│  │ 1. Position Size -50%       │                            │
│  │ 2. Optimizer triggern       │                            │
│  │ 3. 2h Beobachtungsfenster   │                            │
│  └─────────────────────────────┘                            │
│                 │                                           │
│     ┌───────────┴───────────┐                               │
│     ▼                       ▼                               │
│  Erholt sich?           Weiter schlecht?                    │
│     │                       │                               │
│     ▼                       ▼                               │
│  Normalbetrieb          Abstufung (Gold→Silver→Bronze)      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Budget Tracking (Virtuelle Konten)

### Konzept

Jede Strategie hat ihr eigenes virtuelles Konto:

```
┌─────────────────────────────────────────────────────────────┐
│                 VIRTUELLE KONTEN                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Gesamt-Budget: $10,000                                     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 🥇 GOLD                                             │   │
│  │ Allokation: 50% = $5,000                            │   │
│  │ Aktuell: $5,234.50 (+4.69%)                         │   │
│  │ Trades heute: 47                                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 🥈 SILVER                                           │   │
│  │ Allokation: 30% = $3,000                            │   │
│  │ Aktuell: $2,987.20 (-0.43%)                         │   │
│  │ Trades heute: 31                                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 🥉 BRONZE                                           │   │
│  │ Allokation: 20% = $2,000                            │   │
│  │ Aktuell: $2,045.00 (+2.25%)                         │   │
│  │ Trades heute: 22                                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Performance Attribution

Jeder Trade wird seiner Strategie zugeordnet:

```python
{
    "trade_id": "t_20240115_001",
    "strategy_id": "strat_2024_001",  // vom Research Agent vergeben
    "strategy_tier": "gold",
    "entry_price": 42150.00,
    "exit_price": 42380.00,
    "pnl": 23.50,
    "pnl_percent": 0.55,
    "virtual_balance_after": 5234.50
}
```

---

## Logging & Datenbank

### Zentrale PostgreSQL Datenbank

Keine verteilten Log-Dateien. Alles in einer DB:

```sql
-- Kern-Tabellen
trades              -- Jeder einzelne Trade
strategy_snapshots  -- Stündliche Strategy-Metriken
alerts              -- Alle Warnungen und Ereignisse
optimizer_runs      -- Parameter-Änderungen
discovery_pipeline  -- Neue Strategien im Prozess
```

### Trade Log Schema

```sql
CREATE TABLE trades (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    strategy_id VARCHAR(50),
    strategy_tier VARCHAR(10),  -- gold/silver/bronze/challenger
    symbol VARCHAR(20),
    side VARCHAR(10),           -- buy/sell
    entry_price DECIMAL(20,8),
    exit_price DECIMAL(20,8),
    quantity DECIMAL(20,8),
    pnl DECIMAL(20,8),
    pnl_percent DECIMAL(10,4),
    fees DECIMAL(20,8),
    slippage DECIMAL(20,8),
    duration_seconds INTEGER,
    metadata JSONB              -- Flexible Zusatzinfos
);
```

### Strategy Snapshot Schema

```sql
CREATE TABLE strategy_snapshots (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    strategy_id VARCHAR(50),
    tier VARCHAR(10),

    -- Performance Metriken
    total_trades INTEGER,
    win_rate DECIMAL(5,2),
    profit_factor DECIMAL(10,4),
    sharpe_ratio DECIMAL(10,4),
    sortino_ratio DECIMAL(10,4),
    calmar_ratio DECIMAL(10,4),
    max_drawdown DECIMAL(10,4),

    -- Composite Score
    composite_score DECIMAL(10,4),
    alpha DECIMAL(10,4),

    -- Budget
    allocated_balance DECIMAL(20,8),
    current_balance DECIMAL(20,8),

    -- Rolling Windows
    sharpe_1h DECIMAL(10,4),
    sharpe_4h DECIMAL(10,4),
    sharpe_24h DECIMAL(10,4)
);
```

---

## Auto-generierter STATUS.md

### Konzept

Alle 4 Stunden generiert das System automatisch einen STATUS.md:

```
┌─────────────────────────────────────────────────────────────┐
│                 STATUS.md GENERATION                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PostgreSQL ──► Analyzer Agent ──► STATUS.md                │
│                                                             │
│  Trigger: Alle 4 Stunden oder bei wichtigen Events          │
│  Output: /status/STATUS.md (wird überschrieben)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### STATUS.md Template

```markdown
# Trading Genesis 2 - Status

**Stand:** 2024-01-15 14:00 UTC
**Uptime:** 3d 14h 22m
**Modus:** Paper Trading

## Portfolio Übersicht

| Metrik | Wert |
|--------|------|
| Gesamt-Balance | $10,266.70 |
| Tages-PnL | +$142.30 (+1.41%) |
| Alpha (vs Markt) | +0.8% |
| Trades heute | 100 |

## Champions

| Rang | Score | PnL 24h | Trades |
|------|-------|---------|--------|
| 🥇 Gold | 2.34 | +$234.50 | 47 |
| 🥈 Silver | 1.98 | -$12.80 | 31 |
| 🥉 Bronze | 1.76 | +$45.00 | 22 |

## Challengers

| Slot | Fortschritt | Trades | vs Bronze |
|------|-------------|--------|-----------|
| Challenger 1 | 18h/24h | 75 | +0.12 |
| Challenger 2 | 6h/24h | 25 | +0.05 |

## Warteschlange

3 Strategien bereit (vom Research Agent entdeckt)

## Alerts (letzte 24h)

- ⚠️ 12:30 - Silver: Sharpe -15% (4h window)
- ✅ 12:45 - Silver: Erholt auf -5%
- 🔄 08:00 - Gold/Silver Swap durchgeführt

## Discovery Pipeline

| Status | Anzahl |
|--------|--------|
| Research | 2 in Arbeit |
| Validation | 1 wartend |
| Backtest | 1 läuft |
| Pending Requirements | 1 (fehlt: API)

---
*Auto-generiert alle 4h*
```

---

## Telegram-Benachrichtigungen

Nutzt den bestehenden Telegram-Bot: `/home/rolf_vps/telegram-bot/`

### Regelmäßige Status-Updates (Cron)

```
┌─────────────────────────────────────────────────────────────┐
│              SCHEDULED NOTIFICATIONS                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Frequenz      │ Inhalt                                     │
│  ─────────────┼─────────────────────────────────────────── │
│  Alle 4h      │ Kurzer Status (PnL, Trades, Alpha)         │
│  Täglich 08:00│ Tages-Report (alle Champions, Alerts)      │
│  Wöchentlich  │ Wochen-Summary + Discovery-Pipeline        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Cron Jobs:**
```bash
# Alle 4 Stunden: Kurz-Status
0 */4 * * * /home/rolf_vps/telegram-bot/send_trading_status.sh

# Täglich 08:00: Tages-Report
0 8 * * * /home/rolf_vps/telegram-bot/send_daily_report.sh

# Sonntags 20:00: Wochen-Summary
0 20 * * 0 /home/rolf_vps/telegram-bot/send_weekly_report.sh
```

### Proaktive Alerts (Event-getriggert)

```
┌─────────────────────────────────────────────────────────────┐
│              PROAKTIVE BENACHRICHTIGUNGEN                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Event                        │ Nachricht                   │
│  ────────────────────────────┼──────────────────────────── │
│  API-Fehler (Exchange)       │ 🚨 Binance API down!        │
│  API-Fehler (CoinGecko)      │ 🚨 Benchmark-Daten fehlen   │
│  Strategie braucht neue API  │ ⚠️ Whale Alert API needed   │
│  Drawdown > 10%              │ 🔴 Drawdown-Warnung!        │
│  Champion-Swap               │ 🔄 Gold: X → Y              │
│  System-Pause                │ ⛔ Trading pausiert          │
│  Challenger schlägt Bronze   │ 🏆 Neuer Champion!          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Nachricht-Formate

**4h Status:**
```
📊 Trading Genesis Status
━━━━━━━━━━━━━━━━━━━━━
PnL 4h: +$45.20 (+0.45%)
Alpha: +0.12%
Trades: 23
🥇 Gold: +$28.50
🥈 Silver: +$12.30
🥉 Bronze: +$4.40
```

Hinweis: Strategienamen werden vom System dynamisch vergeben basierend auf dem was der Research Agent findet. Die Nachricht zeigt nur Rang und Performance - nicht die interne Implementierung.

**Problem-Alert:**
```
🚨 AKTION ERFORDERLICH
━━━━━━━━━━━━━━━━━━━━━
Problem: CoinGecko API Rate Limit
Impact: Benchmark-Berechnung gestoppt
Lösung: API-Key in config eintragen

Details: 429 Too Many Requests seit 14:32
```

---

## Money Management

### Default-System

```
┌────────────────────────────────────────┐
│         DEFAULT MONEY MANAGEMENT       │
├────────────────────────────────────────┤
│ - Risk per Trade: 1-2%                 │
│ - Stop Loss: 2-3x ATR                  │
│ - Trailing Stop: ATR-basiert           │
│ - Take Profit: Min 1:2 R:R             │
│ - Max gleichzeitige Positionen: 5      │
│ - Max Drawdown: 20% → System Pause     │
└────────────────────────────────────────┘
```

### Strategie-spezifisches MM

Parser extrahiert MM-Aspekte. Risk Agent entscheidet:
```
IF Strategie-MM vorhanden AND durch Backtest validiert:
    → Nutze Strategie-MM
ELSE:
    → Nutze Default-MM
```

---

## Paper Trading - Realismus

### Simulation muss enthalten

| Aspekt | Wie simuliert |
|--------|---------------|
| Gebühren | Exchange-spezifisch (Binance: 0.1%) |
| Slippage | ATR-basiert |
| Partial Fills | Volumen-basiert |
| Latenz | 50-500ms Delay |
| Spread | Live Bid/Ask Daten |

### Exchange Testnets

| Exchange | URL | Features |
|----------|-----|----------|
| **Binance** | testnet.binancefuture.com | Echte Preise, virtuelle Orders |
| **Bybit** | api-demo.bybit.com | 50k USDT virtuell |

**Vorteil:** Gleiche API wie Live → nahtloser Umstieg

### Umstieg Paper → Live

```python
# Paper (nur URL-Unterschied)
exchange = ccxt.binance({
    'apiKey': 'TESTNET_KEY',
    'secret': 'TESTNET_SECRET',
    'urls': {'api': 'https://testnet.binancefuture.com'}
})

# Live
exchange = ccxt.binance({
    'apiKey': 'LIVE_KEY',
    'secret': 'LIVE_SECRET'
})
```

---

## Alle Agents

| # | Agent | Schicht | Läuft |
|---|-------|---------|-------|
| 1 | **Orchestrator** | Alle | Kontinuierlich |
| 2 | **Signal Agent** | Echtzeit | Kontinuierlich |
| 3 | **Risk Agent** | Echtzeit | Bei jedem Signal |
| 4 | **Executor Agent** | Echtzeit | Bei validiertem Trade |
| 5 | **Monitor Agent** | Echtzeit | Kontinuierlich |
| 6 | **Analyzer Agent** | Analyse | Stündlich |
| 7 | **Optimizer Agent** | Analyse | Stündlich/Täglich |
| 8 | **Research Agent** | Discovery | Wöchentlich |
| 9 | **Parser Agent** | Discovery | Bei neuer Strategie |
| 10 | **Validator Agent** | Discovery | Bei neuer Strategie |
| 11 | **Coder Agent** | Discovery | Bei validierter Strategie |
| 12 | **Backtest Agent** | Discovery | Bei neuer Strategie |
| 13 | **Evaluator Agent** | Discovery | Nach Backtests |

---

## Technische Infrastruktur

### Kern-APIs (immer benötigt)

| Service | Zweck | Kosten |
|---------|-------|--------|
| Binance Testnet | Paper Trading | Kostenlos |
| Binance API | Live Marktdaten (1m/5m) | Kostenlos |
| GitHub API | Strategy Research | Kostenlos |

### Optionale APIs (strategie-abhängig)

Werden bei Bedarf hinzugefügt:
- News APIs (CryptoPanic, etc.)
- Sentiment APIs (LunarCrush, etc.)
- On-Chain APIs (Glassnode, etc.)
- Whale Tracking (Whale Alert, etc.)

### MCP Server

| Server | Zweck |
|--------|-------|
| ccxt-mcp | Exchange Daten + Trading |
| postgres-mcp | Datenbank |

### Backtesting Engine

**Backtrader:**
- Slippage-Simulation
- Commission/Fees
- Partial Fills
- Live-Trading-fähig

---

## Datenfluss

```
┌─────────────────────────────────────────────────────────────┐
│                     DISCOVERY                               │
└─────────────────────────────────────────────────────────────┘
        │
Internet → Research → Parser → Validator ─┬─► Coder → Backtest
                                          │
                                          └─► PENDING (fehlt API)
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                   WARTESCHLANGE                             │
│              (max 5 getestete Strategien)                   │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                    CHALLENGER                               │
│                (max 2 parallel)                             │
└─────────────────────────────────────────────────────────────┘
        │
        ▼ (wenn besser als Bronze)
┌─────────────────────────────────────────────────────────────┐
│                    CHAMPIONS                                │
│            🥇 Gold │ 🥈 Silver │ 🥉 Bronze                  │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│              ECHTZEIT TRADING                               │
│     Signal → Risk → Executor → Monitor → Analyzer           │
└─────────────────────────────────────────────────────────────┘
        │
        └─► Optimizer (passt Parameter an)
```

---

## Quellen für Research Agent

| Quelle | Typ | Qualität |
|--------|-----|----------|
| GitHub (Pine Script) | Code + Doku | Hoch |
| TradingView Community | Strategien | Mittel |
| arXiv / SSRN | Papers | Hoch |
| Medium / Blogs | Tutorials | Variabel |
| QuantConnect | Algorithmen | Hoch |
