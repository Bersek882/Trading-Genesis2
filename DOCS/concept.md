# Trading Genesis 2 - Konzept

## 1. Überblick

Selbstverbesserndes Active-Intraday Krypto-Trading-System mit dynamischer Strategie-Entdeckung durch Claude Code Agents.

**Alles ist Paper Trading bis zur manuellen Umstellung auf Echtgeld.**

**Kernprinzipien:**
- 50-100+ Trades/Tag → schnelle statistische Signifikanz
- 3 Champions (Gold/Silver/Bronze) + 2 Challenger-Slots
- Strategien werden vom System entdeckt, kombiniert und parametrisiert — basierend auf einem Baukasten bewährter Indikatoren
- Python Daemon entscheidet schnell (Echtzeit), Claude CLI optimiert die Regeln (periodisch)

---

## 2. Markt-Definition

| Parameter | Wert (Initial) | Später erweiterbar |
|-----------|----------------|-------------------|
| **Instrument** | Spot | Perpetual Futures |
| **Richtung** | Long only | Long + Short |
| **Leverage** | Kein (1x) | Bis 3x |
| **Paare** | USDT-Paare + Krypto/Krypto (z.B. ETH/BTC) | — |
| **Liquiditäts-Minimum** | >$50M 24h-Volumen, Spread <0.1% | — |
| **Ordertypen** | Market, Limit, OCO (Stop-Loss + Take-Profit) | Post-Only, Trailing |
| **Rundung** | Per Exchange-Info API (stepSize/tickSize) | — |

**Pair Selection:** Dynamisch. Täglich Top-Paare nach Volumen + Volatilität evaluieren. Verschiedene Strategien können verschiedene Paare handeln. Blacklist für Delisting-Risiko.

---

## 3. System-Architektur

```
┌─────────────────────────────────────────────────────────┐
│         PYTHON DAEMON (24/7, systemd, Millisekunden)    │
├─────────────────────────────────────────────────────────┤
│  WebSocket ──► Preis-Engine ──► Indikator-Engine        │
│  Signal Engine ──► Risk Engine ──► Order Executor       │
│  Trade Monitor ──► alle Trades → PostgreSQL             │
└──────────────────────┬──────────────────────────────────┘
                       │ PostgreSQL = State Store
┌──────────────────────▼──────────────────────────────────┐
│         CLAUDE CLI AGENTS (periodisch, ~12s/Call)       │
├─────────────────────────────────────────────────────────┤
│  Alle 1-4h:  Analyzer, Optimizer, Orchestrator          │
│  Täglich:    Walk-Forward Reoptimierung                  │
│  Wöchentlich: Research Agent (Hypothesen-Generator)      │
└─────────────────────────────────────────────────────────┘
```

**Kommunikation:** Kein Agent-zu-Agent Messaging. Python Daemon schreibt State in DB → Claude liest, entscheidet, schreibt Aktionen zurück → Python führt aus. Claude hat nur READ-Zugriff auf die DB. Nur der Daemon schreibt.

---

## 4. StrategySpec Interface

Jede Strategie ist ein Python-Modul mit festem Interface:

```python
class IStrategy:
    # Pflicht-Attribute
    timeframe: str              # "1m", "5m", "15m"
    allowed_pairs: list[str]    # ["BTC/USDT", "ETH/USDT"]
    stoploss: float             # -0.02 (2% vom Entry)
    max_orders_per_hour: int    # Rate-Limit

    # Pflicht-Methoden
    def generate_signal(self, candles, indicators) -> Signal | None
    def calculate_exit(self, position, candles) -> ExitSignal | None
    def get_position_size(self, balance, risk_budget) -> float

    # Constraints (vom System erzwungen, nicht überschreibbar)
    MAX_LEVERAGE = 1            # Initial kein Leverage
    MAX_POSITION_PCT = 0.10     # Max 10% des Budgets pro Position
    MAX_DAILY_TRADES = 200      # Hard Limit
```

**Sandboxing:** Claude-generierter Strategie-Code läuft in gVisor-sandboxed Docker Containern. Kein Zugriff auf API-Keys, DB oder Netzwerk. Nur strukturierte JSON-Signale als Output. Der Daemon validiert jedes Signal gegen Schema + Range-Checks bevor er handelt.

**Hot-Reload:** Neue Strategie-Version = neuer Prozess starten, alten graceful stoppen. Kein `importlib.reload()`.

---

## 5. Discovery Pipeline

### 5.1 Ansatz: Hypothesis Generator

~90% öffentlich verfügbarer Strategien sind overfit. Der Research Agent kopiert nicht — er generiert Hypothesen:

```
Research Agent (Claude CLI)
  ├─ Analysiert Marktstruktur aus DB (Volatilität, Trends, Korrelationen)
  ├─ Generiert Hypothese: "Mean-Reversion mit engen BB bei Seitwärtstrend"
  ├─ Kombiniert Building Blocks (siehe unten)
  └─ Optional: Akademische Quellen (arXiv, SSRN) als Inspiration
      │
      ▼
Coder Agent (Claude CLI) → Implementiert als IStrategy
      │
      ▼
Automatisierte Validation Gates (VectorBT)
      │
      ▼
Evaluator Agent (Claude CLI) → DSR + FDR-Korrektur
      │
      ▼
Warteschlange (max 5) → bereit für Challenger-Slot
```

### 5.2 Building Blocks

| Kategorie | Beispiele |
|-----------|-----------|
| **Entry** | Momentum (RSI, MACD), Mean-Reversion (BB, Keltner), Breakout (ATR, Donchian), Volume-Spike |
| **Filter** | Trend (EMA, ADX), Volatilität (ATR-Level), Volume, Regime |
| **Exit** | ATR-Trail, Time-Based, Fixed Target, Chandelier |
| **Sizing** | Fixed-Fraction, Volatility-Adjusted, Quarter-Kelly |

### 5.3 Validation Gates

Jede Strategie muss **alle** Gates bestehen:

| # | Gate | Kriterium | Zweck |
|---|------|-----------|-------|
| 1 | **Walk-Forward** | IS=5d, OOS=2d, WFE > 0.5 | Robustheit |
| 2 | **Monte Carlo** | 10.000× Bootstrap, 95% CI Sharpe > 0 | Zufalls-Ausschluss |
| 3 | **Regime-Test** | Profitabel in Bull, Bear UND Sideways | Regime-Robustheit |
| 4 | **Parameter-Sensitivität** | ±20% Variation → Performance stabil | Overfitting-Check |
| 5 | **Cost-Edge** | Profit Factor > 1.5 nach Fees+Slippage | Kosten-Deckung |
| 6 | **DSR** | Deflated Sharpe Ratio > 0 (p < 0.05) | Statistische Signifikanz |
| 7 | **Korrelation** | \|r\| < 0.60 zu bestehenden Champions | Diversifikation |

### 5.4 Deflated Sharpe Ratio (DSR)

Primäres Signifikanz-Gate. Korrigiert gleichzeitig für:
- **N:** Anzahl aller jemals getesteten Strategien (Gesamt-Zähler in DB)
- **Skewness & Kurtosis:** Crypto-Heavy-Tails
- **Stichprobengröße T**

```
DSR = Φ[((SR̂ - SR₀) × √(T-1)) / √(1 - γ₃×SR̂ + ((γ₄-1)/4) × SR̂²)]
```

SR₀ = erwarteter Maximum-Sharpe unter Null-Hypothese (steigt mit N). Strategie besteht nur wenn DSR > 0 bei p < 0.05.

### 5.5 Meta-Overfitting Schutz

- **WFA-Konfiguration einfrieren:** Fenstergrößen, Fitness-Funktion, Parameter-Ranges einmal festlegen, mindestens 3 Monate nicht ändern
- **N_total tracken:** Jede getestete Strategie zählt, auch verworfene. DSR berücksichtigt N_total
- **Logische Begründung:** Jede Strategie braucht eine dokumentierte Hypothese, warum der Edge existiert — statistische Signifikanz allein reicht nicht

---

## 6. Strategie-Hierarchie & Deployment

### 6.1 Champions & Challengers

```
CHAMPIONS (Paper)              CHALLENGERS (Paper)
┌──────────────────┐          ┌──────────────────┐
│ 🥇 GOLD   (50%) │          │ Challenger 1     │
│ 🥈 SILVER (30%) │          │ Challenger 2     │
│ 🥉 BRONZE (20%) │          └──────────────────┘
└──────────────────┘
        ▲                     WARTESCHLANGE (max 5)
        └─── Aufstieg ◄───── getestete Strategien
```

### 6.2 Challenger-Logik

- Läuft max 24h, sammelt min 50 Trades
- **Early Kill:** Wenn nach 30 Trades klar unprofitabel → sofort stoppen
- Vergleich mit Bronze: Challenger besser (Sharpe > Bronze + 0.1)?
- **BHY-FDR Korrektur** bei 10%: Erst nach FDR-Korrektur signifikant besser → Aufstieg

| Regel | Wert |
|-------|------|
| Mindest-Trades | 50 |
| Max Testzeit | 24h |
| Cooldown nach Swap | 6h |
| Max Swaps/Tag | 3 |
| Rollback-Fenster | 2h |

### 6.3 Canary Deployment (Ramp-Up)

Kein direkter Sprung auf volle Allokation. Stufenweiser Aufbau:

| Phase | Kapital | Min. Dauer | Gate zum nächsten Level |
|-------|---------|------------|------------------------|
| **Shadow** | 0% (nur Simulation) | 50 Trades / 2 Wochen | Alle Validation Gates bestanden |
| **Canary** | 1% des Portfolios | 30 Live-Trades | Positive PnL, kein unerwartetes Verhalten |
| **Scaling** | 5% des Portfolios | 100 Live-Trades | Sharpe > 1.0 ann., Max DD < 3% |
| **Full** | Target (20/30/50%) | Ongoing | Konsistente Performance |

**Reverse-Ramp:** Bei Drawdown > Schwelle → sofort eine Stufe zurück. Bei Canary-Phase Drawdown → auf 0% zurück.

### 6.4 Correlation Constraints

- Max paarweise |r| < 0.60 zwischen Champions (Equity-Kurven, rolling 24h)
- Drawdown-Korrelation prüfen: Fallen alle Champions gleichzeitig? → Zu hohes systemisches Risiko
- Min. 2 verschiedene Strategy-Familien unter Champions
- Korrelation VOR Swap prüfen

---

## 7. Risk Management

### 7.1 Portfolio-Level

| Regel | Wert |
|-------|------|
| **Max Net Exposure** | ±30% NAV in eine Richtung |
| **Max Exposure pro Asset** | 40% NAV |
| **Daily Loss Hard Stop** | 2% → alle Trades stoppen |
| **Weekly Loss Limit** | 4% |
| **Monthly Loss Limit** | 6% |

Wenn Gold und Silver gleichzeitig Long BTC → effektiv 80% in eine Richtung → **wird vom Risk Engine blockiert** (Net Exposure > 30%).

### 7.2 Position Sizing

**1-2% Risk per Trade ist bei 50-100 Trades/Tag katastrophal.** Korrekt:

| Parameter | Wert |
|-----------|------|
| **Risk per Trade** | 0.02% - 0.05% des Portfolio-NAV |
| **Sizing-Methode** | Quarter-Kelly (25% des Kelly-Optimums) |
| **Kelly Recalculation** | Täglich, rolling 30d Daten |
| **Stop-Loss** | 2-3× ATR vom Entry |
| **Max gleichzeitige Positionen** | 5 pro Strategie, 10 Portfolio-weit |

**Herleitung:** Daily Loss Budget 2% / 100 Trades = 0.02% per Trade. Quarter-Kelly als Obergrenze.

### 7.3 Circuit Breaker

| Stufe | Trigger | Aktion |
|-------|---------|--------|
| **REDUCE** | Portfolio DD > 5% | Position Sizes halbieren, 1h keine neuen Trades |
| **PAUSE** | Portfolio DD > 10% oder 3+ Champions im DD | Alle Trades stoppen, nur Monitoring |
| **FULL STOP** | Portfolio DD > 15% oder Exchange-Fehler | Alle Positionen schließen, Telegram-Alert |

### 7.4 Exchange-seitige Stop-Losses

**Immer aktiv, unabhängig vom Bot:** Jede Position hat einen OCO-Order auf der Exchange. Falls Bot crasht → Exchange schließt automatisch.

---

## 8. Erfolgsmessung

### 8.1 Composite Score (Rank-basiert)

Metriken haben verschiedene Skalen → **Percentile-Rank-Normalisierung** über alle verglichenen Strategien:

```
Score = 0.30 × rank(Sortino) + 0.30 × rank(Calmar) + 0.25 × rank(ProfitFactor) + 0.15 × rank(Consistency)
```

Wobei `rank()` = Percentile-Rang (0-100) unter allen aktiven + Challenger-Strategien.

**Gewichte für mindestens 3 Monate einfrieren.** Kein Gewichte-Tuning nach ersten Ergebnissen.

| Metrik | Gewicht | Misst |
|--------|---------|-------|
| Sortino | 30% | Risiko-adjustierte Rendite (Downside) |
| Calmar | 30% | Rendite / Max Drawdown |
| Profit Factor | 25% | Brutto-Gewinn / Brutto-Verlust |
| Consistency | 15% | % profitable 4h-Perioden |

### 8.2 Alpha & Benchmark

```
Benchmark = Krypto-Marktkapitalisierung (CoinGecko API)
Alpha = Strategie-Return − Markt-Return
Positiver Return + negativer Alpha → Strategie wird degradiert.
```

---

## 9. Overfitting-Schutz

| Methode | Details |
|---------|---------|
| **Walk-Forward** | IS=5d, OOS=2d, WFE > 0.5. Konfiguration 3 Monate eingefroren. |
| **Monte Carlo** | 10.000× Bootstrap-Resampling der Trade-Sequenz. 95% CI Sharpe > 0. |
| **Regime-Test** | Bull (+10%/7d): Alpha > 0. Bear (−10%/7d): DD < Markt. Sideways (±5%/7d): Return > 0. |
| **Parameter-Sensitivität** | ±20% Variation. Starke Schwankung = Overfit. |
| **DSR** | Deflated Sharpe Ratio korrigiert für N_total, Skewness, Kurtosis. |
| **Meta-Overfitting** | WFA-Config einfrieren. N_total tracken. Logische Hypothese dokumentieren. |
| **Cost-Edge** | Profit Factor > 1.5 nach allen Kosten (Fees + Slippage × 1.5). |

### Echtzeit-Regime-Erkennung

Zwei Stufen:

| Stufe | Methode | Latenz | Update |
|-------|---------|--------|--------|
| **Schnell** | Threshold: Rolling 4h Volatilität > X → "High-Vol" Flag | Instant | Jede Minute |
| **Langsam** | HMM (hmmlearn, 2-3 States) auf täglichen Returns | 1-2 Tage | Alle 4-6h |

Regime beeinflusst: Welche Strategien aktiv sind, Position Sizes, Stop-Distances.

---

## 10. Operations

### 10.1 Agent Decision Logging

Jede Claude-CLI-Entscheidung wird geloggt — vollständig und unveränderbar:

| Feld | Inhalt |
|------|--------|
| `prompt_hash` | SHA-256 des vollen Prompts |
| `response_text` | Volle Claude-Antwort |
| `parsed_action` | Strukturiertes JSON (validiert gegen Schema) |
| `market_snapshot` | Preis, Volumen, Regime zum Zeitpunkt |
| `guardrail_results` | Range-Check, Schema-Check, Exposure-Check |
| `execution_result` | Was tatsächlich passiert ist |

**Speicherung:** Append-only JSONL (Hash-Chain für Tamper-Detection) + PostgreSQL für Queries.

**Guardrails:** Jede Claude-Antwort wird validiert:
- JSON-Schema Validation (ungültig → verwerfen)
- Range-Check: Preise ±5% vom Markt (sonst → Halluzination)
- Rate-Limit: Max N Aktionen pro Stunde
- Exposure-Check: Würde die Aktion Limits verletzen?

### 10.2 Change Governance

| Änderungstyp | Freigabe | Tests vorher |
|--------------|----------|--------------|
| Mikro-Parameter (±10%) | Autonom | Backrecheck |
| Makro-Parameter (neue Indikatoren) | Autonom | Alle Validation Gates |
| Neue Strategie aktivieren | Autonom | Alle Gates + Canary |
| Strategie-Code editieren | **Menschlich** | Code Review |
| System-Config/DB-Schema | **Menschlich** | — |

**Grundregel:** Claude gibt JSON-Anweisungen. Der Python Daemon validiert und führt aus. Claude hat nie direkten Schreibzugriff auf DB oder Exchange.

### 10.3 Degradation Monitoring

| Window | Update | Verwendet für |
|--------|--------|---------------|
| 1h | Jede Min | Anomalie-Erkennung |
| 4h | Alle 5 Min | Micro-Optimierung |
| 12h | Alle 15 Min | Trend-Erkennung |
| 24h | Stündlich | Champion-Vergleich |

**Alert-Stufen:**

| Stufe | Trigger | Aktion |
|-------|---------|--------|
| INFO | Sharpe −10% | Log |
| WARNING | Sharpe −25% | Telegram + Position Size −50% + 2h Beobachtung |
| CRITICAL | Sharpe −50% | Auto-Pause, Optimizer triggern |
| EMERGENCY | DD > 15% | Full Stop, alle Positionen schließen |

### 10.4 Crash Recovery

- **systemd:** `Restart=always`, `RestartSec=10`, `WatchdogSec=300`
- **Bei Neustart:** Offene Positionen prüfen (Exchange-Side), DB-State abgleichen, Inkonsistenzen → Telegram-Alert + manueller Review
- **Health-Check:** HTTP `/health` Endpoint, Cron prüft alle 5 Min

---

## 11. Paper Trading

### Hybrid-Ansatz (Industrie-Standard)

**Nicht Binance Testnet** (unrealistisches Orderbook, monatliche Resets, Preisabweichungen). Stattdessen:

- **Live-Marktdaten** von echter Binance API (WebSocket: Orderbook, Trades, Candles)
- **Lokale Order-Simulation** im Python Daemon

### Slippage Model

| Komponente | Beschreibung |
|------------|--------------|
| Orderbook-Depth | Echte Bid/Ask-Tiefe → Fill-Preis berechnen |
| Market Impact | Almgren-Chriss: Größere Orders bewegen Preis stärker |
| Zeitabhängig | Höhere Slippage in illiquiden Phasen |
| **Sicherheits-Multiplikator** | **1.5×** auf berechnete Slippage (konservativ) |

Weitere Simulation: Exchange-Fees (Binance 0.1%), Partial Fills (volumenbasiert), Latenz (50-500ms).

**Testnet:** Nur für API-Connectivity-Tests und Order-Format-Validierung, nicht für Strategie-Evaluation.

### Execution Quality Feedback (bei Echtgeld-Umstellung)

Vergleichstabelle: Simulierter Fill vs. tatsächlicher Fill → Slippage-Modell kalibrieren.

---

## 12. Datenmanagement

| Aspekt | Lösung |
|--------|--------|
| **Historische Candles** | Binance REST API (1m/5m), gespeichert in PostgreSQL |
| **Orderbook** | Live via WebSocket, nicht historisch (zu viel Storage) |
| **Datenqualität** | Spike-Filter (±X% in einem Candle), Gap-Detection, Validierung |
| **Look-Ahead Bias** | Daten erst nach Candle-Close verfügbar. Strikt erzwungen. |
| **Survivorship Bias** | Universe-Changes loggen (Delistings, neue Listings) |
| **Clock/Time-Sync** | NTP, Exchange-Server-Time als Referenz |

---

## 13. Bootstrapping Phase

| Phase | Zeitraum | Aktion |
|-------|----------|--------|
| **Seed** | Tag 1-7 | 3 Basis-Strategien (Trend, Mean-Reversion, Volatility-Breakout), gleichgewichtet, Daten sammeln |
| **Validierung** | Tag 7-14 | Walk-Forward + Monte Carlo mit gesammelten Daten. Erste Optimierung. |
| **Ranking** | Tag 14-21 | Champion-System aktivieren. Erste Challenger aus Discovery Pipeline. |
| **Vollbetrieb** | Ab Tag 21 | Komplettes System mit allen Gates und Canary Deployment. |

---

## 14. Selbstverbesserung

**Mikro-Optimierung (alle 1-4h, Claude CLI):**
- ATR-Multiplier, Thresholds, Stop-Distances anpassen (max ±10% autonom)
- Basierend auf Rolling-Performance der letzten 4-12h

**Makro-Optimierung (täglich, Claude CLI):**
- Walk-Forward Reoptimierung (IS=5d, OOS=2d, WFE > 0.5)
- Neue Strategie-Version → Canary Deployment

---

## 15. Technische Infrastruktur

### Agents

| # | Agent | Implementierung | Frequenz |
|---|-------|-----------------|----------|
| 1 | Signal Engine | Python Daemon | Kontinuierlich |
| 2 | Risk Engine | Python Daemon | Bei jedem Signal |
| 3 | Order Executor | Python Daemon | Bei validiertem Trade |
| 4 | Trade Monitor | Python Daemon | Kontinuierlich |
| 5 | Orchestrator | Claude CLI | Alle 1-4h |
| 6 | Analyzer | Claude CLI | Alle 1-4h |
| 7 | Optimizer | Claude CLI | Alle 1-4h / Täglich |
| 8 | Research Agent | Claude CLI | Wöchentlich |
| 9 | Coder Agent | Claude CLI | Bei neuer Hypothese |
| 10 | Evaluator | Claude CLI | Nach Backtests |

### Datenbank (PostgreSQL)

```sql
trades              -- Jeder Trade (Entry, Exit, PnL, Fees, Slippage)
strategy_snapshots  -- Stündliche Metriken (Sharpe, Sortino, DD, Score)
alerts              -- Warnungen und Events
optimizer_runs      -- Parameter-Änderungen mit Begründung
discovery_pipeline  -- Strategien im Prozess + N_total Zähler
agent_decisions     -- Vollständiges Decision Log (Prompt, Response, Guardrails)
```

### Strategy Versioning

```
strategies/
├── strat_001_v1/    # Original
├── strat_001_v2/    # Nach Optimierung
└── registry.json    # Aktive Versionen
```

Rollback: Version in registry.json zurücksetzen → Daemon lädt automatisch.

### APIs

| Service | Zweck | Kosten |
|---------|-------|--------|
| Binance API | Live-Marktdaten + WebSocket | Kostenlos |
| CoinGecko | Benchmark (Marktcap) | Kostenlos |
| ccxt-mcp | Exchange-Anbindung | — |
| postgres-mcp | Datenbank | — |

Optionale APIs bei Bedarf: News (CryptoPanic), Sentiment (LunarCrush), On-Chain (Glassnode).

### Telegram

| Frequenz | Inhalt |
|----------|--------|
| Alle 4h | Kurz-Status (PnL, Trades, Alpha, Champions) |
| Täglich 08:00 | Tages-Report |
| Wöchentlich | Summary + Discovery Pipeline |
| Event-getriggert | Drawdown, Swap, Crash, API-Fehler |

### Backtesting

- **VectorBT** (primär): Vektorisiert, 1000× schneller, Monte Carlo eingebaut
- **NautilusTrader** (optional, später): Rust+Python, unified Backtest+Live
