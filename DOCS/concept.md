# Trading Genesis 2 - Konzept

## Überblick

Selbstverbesserndes Active-Intraday Krypto-Trading-System mit dynamischer Strategie-Entdeckung durch Claude Code Agents.

**Alles ist Paper Trading bis zur manuellen Umstellung auf Echtgeld!**

**Kernprinzipien:**
- 50-100+ Trades pro Tag → schnelle Erfolgsmessung
- 3 aktive Champions (Gold, Silver, Bronze) + 2 Challenger-Slots
- Echtzeit-Optimierung und dynamischer Strategie-Austausch
- Keine hardcoded Strategien - alles wird entdeckt und validiert
- Paper Trading mit echten Marktdaten (Echtgeld-Umstellung erfolgt manuell)

---

## Strategie-Hierarchie

```
┌─────────────────────────────────────────────────────────────┐
│                  AKTIVE STRATEGIEN                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CHAMPIONS (Paper)                CHALLENGERS (Paper)       │
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
| Challenger 1-2 | Paper | Testen gegen Champions |

### Aufstieg und Abstieg

```
Challenger schlägt Bronze (nach max 24h)?
    ├─ JA → Challenger wird Bronze, Bronze in Warteschlange
    └─ NEIN → Challenger verworfen, nächste aus Warteschlange

Innerhalb Champions:
    Bronze schlägt Silver? → Tauschen
    Silver schlägt Gold?   → Tauschen
```

---

## System-Architektur (Hybrid: Python Daemon + Claude CLI)

Das System besteht aus zwei Schichten:
- **Python Daemon** (läuft 24/7): Echtzeit-Execution, Monitoring, Datensammlung
- **Claude CLI** (periodisch, ~12s Latenz): Strategische Entscheidungen, Analyse, Discovery

Claude CLI ist zu langsam für Echtzeit-Entscheidungen. Deshalb: Python entscheidet schnell nach vordefinierten Regeln, Claude optimiert die Regeln periodisch.

```
┌─────────────────────────────────────────────────────────────┐
│              PYTHON DAEMON (24/7, Millisekunden)             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Signal Engine ──► Risk Engine ──► Order Executor           │
│       │                                │                    │
│       └──────────┬─────────────────────┘                    │
│                  ▼                                          │
│           Trade Monitor (streamt jeden Trade → DB)          │
│                                                             │
│  WebSocket Feeds ──► Preis-Engine ──► Indikator-Engine      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                   │
                   ▼ (PostgreSQL als State Store)
┌─────────────────────────────────────────────────────────────┐
│           CLAUDE CLI AGENTS (periodisch, ~12s/Call)          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ANALYSE (alle 1-4h):                                       │
│  ├─ Analyzer Agent: Performance-Drift erkennen              │
│  ├─ Optimizer Agent: Parameter-Updates berechnen            │
│  └─ Orchestrator: Champion-Ranking, Hot-Swaps               │
│                                                             │
│  DISCOVERY (täglich/wöchentlich):                            │
│  ├─ Research Agent: Hypothesen generieren                    │
│  ├─ Coder Agent: Strategien implementieren                  │
│  └─ Evaluator Agent: Backtests bewerten                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Kommunikation

Kein Agent-zu-Agent Messaging. Alles über PostgreSQL:
- Python Daemon schreibt Trades, Metriken, Snapshots in DB
- Claude CLI liest DB-State, trifft Entscheidungen, schreibt Aktionen zurück
- Python Daemon liest Aktionen (Parameter-Updates, Swap-Befehle) und führt aus

---

## Bootstrapping Phase

Das System kann nicht am Tag 1 mit Champions starten. Aufbau in Phasen:

| Phase | Zeitraum | Aktion |
|-------|----------|--------|
| **1 - Seed** | Tag 1-7 | 3 Basis-Strategien starten (Trend-Following, Mean-Reversion, Volatility-Breakout). Alle als Paper-Champions gleichgewichtet. Daten sammeln. |
| **2 - Validierung** | Tag 7-14 | Walk-Forward Tests mit gesammelten Daten. Erste Parameter-Optimierung. Monte Carlo Validierung. |
| **3 - Ranking** | Tag 14-21 | Champion-System aktivieren (Gold/Silver/Bronze nach Performance). Erste Challenger aus Discovery Pipeline. |
| **4 - Vollbetrieb** | Ab Tag 21 | Komplettes System: Champions, Challenger, Discovery Pipeline, Auto-Optimierung. |

Die 3 Seed-Strategien werden manuell oder per Claude CLI initial erstellt. Sie dienen als Startpunkt - das System ersetzt sie sobald bessere entdeckt werden.

---

## Zeitplan

```
KONTINUIERLICH (Python Daemon):
├─ Signal Engine generiert Signale nach Strategie-Regeln
├─ Order Executor führt Trades aus
└─ Monitor trackt jeden Trade in DB

ALLE 1-4 STUNDEN (Claude CLI):
├─ Performance-Vergleich aller aktiven Strategien
├─ Champion-Ranking aktualisieren (Gold/Silver/Bronze)
├─ Challenger vs Bronze Vergleich
└─ Hot-Swap wenn Challenger signifikant besser

TÄGLICH (Claude CLI):
├─ Walk-Forward Reoptimierung aller Champions
├─ Vollständiger Performance-Report
└─ Discovery-Pipeline: Warteschlange auffüllen

WÖCHENTLICH (Claude CLI):
└─ Research Agent generiert neue Strategie-Hypothesen
```

### Statistische Basis

Bei 50-100 Trades/Tag:

| Zeitraum | Trades | Aussagekraft |
|----------|--------|--------------|
| 6h | ~25 | Trend erkennbar |
| 12h | ~50 | Erste Signifikanz |
| 24h | ~100 | Gute Signifikanz |

**Challenger-Testzeit:** Maximum 24h (bei 100+ Trades auch früher bewertbar)

---

## Discovery Pipeline

### Grundproblem

~90% der öffentlich verfügbaren Trading-Strategien sind overfit oder unprofitabel. Einfaches Kopieren von GitHub/TradingView ist naiv.

### Ansatz: Hypothesis Generator + Building Block Combiner

Der Research Agent ist KEIN Internet-Scraper. Er ist ein Hypothesen-Generator:

```
┌─────────────────────────────────────────────────────────────┐
│                   DISCOVERY PIPELINE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. RESEARCH AGENT (Claude CLI)                             │
│     ├─ Analysiert aktuelle Marktstruktur (Volatilität,      │
│     │  Trends, Korrelationen) aus DB-Daten                  │
│     ├─ Generiert Hypothesen: "Bei hoher Volatilität +       │
│     │  Seitwärtstrend → Mean-Reversion könnte profitabel    │
│     │  sein mit engen Bändern"                              │
│     ├─ Kombiniert Building Blocks:                          │
│     │  Entries: Momentum, Mean-Reversion, Breakout,         │
│     │  Pattern Recognition                                  │
│     │  Filters: Volatility, Volume, Trend, Regime           │
│     │  Exits: ATR-Trail, Time-Based, Target                 │
│     └─ Optional: Internet-Research als Inspiration          │
│        (arXiv, SSRN - akademische Quellen bevorzugt)        │
│                                                             │
│  2. CODER AGENT (Claude CLI)                                │
│     └─ Implementiert Hypothese als ausführbare Strategie    │
│                                                             │
│  3. BACKTEST (VectorBT - automatisiert)                     │
│     ├─ Walk-Forward: IS=5d, OOS=2d, WFE > 0.5              │
│     ├─ Monte Carlo: 10.000x Bootstrap, 95% CI Sharpe > 0   │
│     ├─ Regime-Tests: Bull, Bear, Sideways                   │
│     └─ Parameter-Sensitivität: ±20% Stabilität              │
│                                                             │
│  4. EVALUATOR (Claude CLI)                                  │
│     ├─ Benjamini-Hochberg FDR-Korrektur (10%)               │
│     ├─ Prüft Korrelation zu bestehenden Champions           │
│     └─ Ranking nach Composite Score                         │
│                                                             │
│  5. WARTESCHLANGE (max 5 Strategien)                        │
│     └─ Bereit für Challenger-Slot                           │
│                                                             │
│  PENDING: Falls Datenquellen fehlen                         │
│     └─ pending_requirements.json → User benachrichtigen     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Building Blocks

Statt ganze Strategien zu kopieren, kombiniert das System modulare Bausteine:

| Kategorie | Beispiele |
|-----------|-----------|
| **Entry Signals** | Momentum (RSI, MACD), Mean-Reversion (BB, Keltner), Breakout (ATR, Donchian), Volume-Spike |
| **Filters** | Trend (EMA Cross, ADX), Volatilität (ATR-Level), Volume, Regime |
| **Exit Rules** | ATR-Trailing-Stop, Time-Based, Fixed Target, Chandelier Exit |
| **Position Sizing** | Fixed-Fraction, Volatility-Adjusted, Kelly Criterion |

Der Research Agent kombiniert diese Bausteine zu neuen Strategien und parametrisiert sie.

---

## Challenger-Logik

```
1. Strategie kommt aus Warteschlange → wird Challenger (Paper)
2. Läuft maximal 24h, sammelt mindestens 100 Trades
3. Vergleich mit Bronze Champion:
   ├─ Challenger BESSER (Sharpe > Bronze + 0.1)? → Challenger wird Bronze
   └─ Challenger SCHLECHTER? → Verworfen, nächste aus Warteschlange
4. Falls Warteschlange leer → Discovery Pipeline priorisieren
```

### Multiple Testing Correction

Bei vielen getesteten Challengern steigt die Wahrscheinlichkeit, dass einer "zufällig" gut abschneidet. Deshalb:
- **Benjamini-Hochberg FDR** bei 10%: Korrigiert für Mehrfachtestung
- Erst wenn ein Challenger auch nach FDR-Korrektur signifikant besser ist, wird er befördert

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

## Strategy Correlation Constraints

Champions müssen diversifiziert sein. Wenn alle drei die gleiche Logik verwenden, steigt das systemische Risiko.

**Regeln:**
- Max paarweise Korrelation |r| < 0.60 zwischen Champions (gemessen an Equity-Kurven)
- Drawdown-Korrelation prüfen: Fallen alle Champions gleichzeitig? → Zu hohes Risiko
- Mindestens 2 verschiedene Strategy-Familien unter den Champions (z.B. Trend + Mean-Reversion)
- Bei Challenger-Aufstieg: Korrelation mit bestehenden Champions prüfen VOR Swap

**Messung:**
- Rolling Pearson-Korrelation über 24h-Fenster der PnL-Serien
- Berechnet bei jedem Champion-Vergleich (alle 4h)

---

## Pair Selection

Das System handelt nicht nur ein Paar. Pair Selection ist dynamisch:

**Kriterien:**
- **Liquidität** ist Hauptkriterium: Ausreichendes Orderbook-Depth für geplante Position Sizes
- Mindest-24h-Volumen (konfigurierbar, z.B. >$50M)
- USDT-Paare und Krypto/Krypto-Paare (z.B. ETH/BTC) wenn Liquidität gegeben
- Verschiedene Strategien können verschiedene Paare handeln

**Dynamische Auswahl:**
- Täglich: Top-Paare nach Volumen + Volatilität evaluieren
- Strategie-spezifisch: Mean-Reversion funktioniert besser auf hochkorrelierten Paaren, Trend-Following auf volatilen Paaren
- Blacklist für Paare mit bekannten Problemen (Delisting-Risiko, extreme Spreads)

---

## Selbstverbesserung

### Optimizer Agent (Claude CLI, periodisch)

**Mikro-Optimierung (alle 1-4h):**
- ATR-Multiplier, Threshold-Werte, Stop-Distances anpassen
- Basierend auf letzte 4-12h Performance-Daten

**Makro-Optimierung (täglich):**
- Walk-Forward Reoptimierung mit IS=5d, OOS=2d
- Größere Parameter-Änderungen
- Walk-Forward Efficiency (WFE) muss > 0.5 sein

### Analyzer Agent (Claude CLI, alle 1-4h)

- Rolling Sharpe (4h, 12h, 24h Fenster) aus DB berechnen
- Rolling Win Rate, Rolling Drawdown
- Performance-Drift erkennen
- Alerts bei Anomalien triggern

---

## Erfolgsmessung

### Composite Score

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

```
Benchmark = Gesamte Krypto-Marktkapitalisierung (CoinGecko API)
Alpha = Strategie-Return - Markt-Return

Strategie mit positivem Return aber negativem Alpha wird degradiert.
```

### Monte Carlo Validierung

Jede Strategie muss Monte Carlo bestehen bevor sie Champion werden kann:

- **Bootstrap Resampling:** Trade-Sequenz 10.000x zufällig neu ordnen
- **95% Konfidenzintervall** für Sharpe Ratio muss > 0 sein
- **Worst-Case Drawdown** aus Monte Carlo → bestimmt maximale Position Size
- Strategien die nur durch glückliche Trade-Reihenfolge profitabel sind, werden aussortiert

---

## Overfitting-Schutz

### 1. Walk-Forward Analysis
- In-Sample: 5 Tage, Out-of-Sample: 2 Tage
- Walk-Forward Efficiency (WFE) = OOS-Performance / IS-Performance
- WFE muss > 0.5 sein (OOS mindestens halb so gut wie IS)

### 2. Regime-Testing

Backtests müssen in allen Marktphasen bestehen:

| Regime | Erkennung | Mindest-Performance |
|--------|-----------|---------------------|
| **Bull** | Markt +10% in 7d | Positiver Alpha |
| **Bear** | Markt -10% in 7d | Geringerer Verlust als Markt |
| **Sideways** | Markt ±5% in 7d | Positiver Return |

**Erweiterte Regime-Erkennung:**
- DVOL (Deribit Volatility Index) für Volatilitäts-Regime
- Funding Rates als Markt-Sentiment-Indikator
- Fear & Greed Index
- BTC-Altcoin Korrelation (hohe Korrelation = Risk-On/Off Regime)

### 3. Parameter-Sensitivität

Parameter ±20% variieren. Wenn Performance stark schwankt → OVERFIT. Stabile Strategie = kleine Parameter-Änderungen → kleine Performance-Änderungen.

### 4. Multiple Testing Correction

Bei N getesteten Strategien: Benjamini-Hochberg FDR bei 10%. Verhindert dass "zufällig gute" Strategien durchkommen.

---

## Degradation Monitoring

### Rolling Windows

| Window | Aktualisierung | Verwendet für |
|--------|----------------|---------------|
| 1h | Jede Minute | Anomalie-Erkennung |
| 4h | Alle 5 Min | Micro-Optimierung |
| 12h | Alle 15 Min | Trend-Erkennung |
| 24h | Stündlich | Champion-Vergleich |

### Alert-Stufen

| Stufe | Trigger | Aktion |
|-------|---------|--------|
| **INFO** | Sharpe fällt um 10% | Logging |
| **WARNING** | Sharpe fällt um 25% | Telegram Notification |
| **CRITICAL** | Sharpe fällt um 50% | Auto-Pause + Review |
| **EMERGENCY** | Drawdown > 15% | Sofortiger Stop |

### Auto-Response bei Degradation

```
Degradation erkannt (Sharpe -25% über 4h)
    │
    ▼
1. Position Size -50%
2. Optimizer triggern
3. 2h Beobachtungsfenster
    │
    ├─ Erholt sich? → Normalbetrieb
    └─ Weiter schlecht? → Abstufung (Gold→Silver→Bronze)
```

---

## Emergency Procedures & Crash Recovery

### Circuit Breaker (3-stufig)

| Stufe | Trigger | Aktion |
|-------|---------|--------|
| **REDUCE** | Drawdown > 10% (Portfolio) | Position Sizes halbieren, keine neuen Trades für 1h |
| **PAUSE** | Drawdown > 15% oder 3+ Champions im Drawdown | Alle Trades stoppen, nur Monitoring aktiv |
| **FULL STOP** | Drawdown > 20% oder Exchange-Fehler | Alle Positionen schließen, System stoppt, Telegram-Alert |

### Exchange-seitige Stop-Losses

**Immer aktiv, unabhängig vom Bot-Status:**
- Jede Position hat einen Exchange-seitigen Stop-Loss (OCO Order)
- Falls Bot crasht, Exchange schließt Position automatisch
- Stop-Loss = 2-3x ATR vom Entry

### Crash Recovery

```
Bot-Absturz erkannt (systemd Restart=always)
    │
    ▼
1. Health-Check: DB erreichbar? Exchange erreichbar?
2. Offene Positionen prüfen (Exchange-Side)
3. Letzten konsistenten DB-State laden
4. Abgleich: Bot-State vs Exchange-State
5. Inkonsistenzen → Telegram-Alert, manueller Review
6. Alles konsistent → Normalbetrieb fortsetzen
```

### Monitoring

- Health-Check Endpoint: `/health` (HTTP)
- systemd watchdog: Restart bei Timeout
- Telegram-Alert bei jedem Restart

---

## Strategy Versioning

Jede Strategie wird versioniert, um Rollbacks zu ermöglichen:

```
strategies/
├── strat_001_v1/    # Original
├── strat_001_v2/    # Nach erster Optimierung
├── strat_002_v1/
└── registry.json    # Kontrolliert welche Version aktiv ist
```

**registry.json:**
```json
{
  "active_strategies": {
    "gold": { "id": "strat_001", "version": "v2" },
    "silver": { "id": "strat_002", "version": "v1" },
    "bronze": { "id": "strat_003", "version": "v1" }
  }
}
```

**Regeln:**
- Neue Version bei jeder Makro-Optimierung (Parameter-Änderung)
- Mikro-Optimierungen (Threshold-Tuning) überschreiben aktuelle Version
- Rollback: Version in registry.json zurücksetzen, Python Daemon lädt automatisch neu
- Alte Versionen bleiben erhalten (kein Löschen)

---

## Paper Trading - Realismus

### Slippage Model

Nicht nur ATR-basiert, sondern mehrstufig:

| Komponente | Beschreibung |
|------------|--------------|
| **Orderbook-Depth** | Echte Bid/Ask-Tiefe aus WebSocket → tatsächlicher Fill-Preis berechnet |
| **Market Impact** | Almgren-Chriss Modell: Größere Orders bewegen den Preis stärker |
| **Zeitabhängig** | Höhere Slippage in illiquiden Phasen (Nacht, Wochenende) |
| **Spread** | Live Bid/Ask Spread (nicht geschätzt) |

### Weitere Simulation

| Aspekt | Wie simuliert |
|--------|---------------|
| Gebühren | Exchange-spezifisch (z.B. Binance: 0.1%) |
| Partial Fills | Volumen-basiert |
| Latenz | 50-500ms Delay |

### Exchange Testnets

| Exchange | URL | Features |
|----------|-----|----------|
| **Binance** | testnet.binancefuture.com | Echte Preise, virtuelle Orders |
| **Bybit** | api-demo.bybit.com | 50k USDT virtuell |

Gleiche API wie Live → nahtloser Umstieg (nur URL-Änderung).

---

## Backtesting Engine

### VectorBT (primär für Backtests)

- Vektorisierte Berechnung (NumPy/Pandas) → 1000x schneller als event-based Engines
- Ideal für schnelles Durchprobieren vieler Hypothesen
- Monte Carlo Simulation eingebaut
- Python-native, einfache Integration

### NautilusTrader (Produktion, optional)

- Rust-Kern + Python-API → hohe Performance
- Unified Backtest + Live Trading (gleicher Code)
- Event-driven, realistischere Simulation
- Für spätere Produktions-Migration geeignet

**Workflow:** VectorBT für schnelle Discovery-Backtests → NautilusTrader für finale Validierung und Live-Trading.

---

## Budget Tracking (Virtuelle Konten)

Jede Strategie hat ihr eigenes virtuelles Konto:

```
Gesamt-Budget: $10,000 (konfigurierbar)

🥇 GOLD:   50% = $5,000 → Trades, PnL, Balance getrackt
🥈 SILVER: 30% = $3,000 → Trades, PnL, Balance getrackt
🥉 BRONZE: 20% = $2,000 → Trades, PnL, Balance getrackt
```

### Performance Attribution

Jeder Trade wird seiner Strategie zugeordnet:
```json
{
    "trade_id": "t_20240115_001",
    "strategy_id": "strat_001_v2",
    "strategy_tier": "gold",
    "entry_price": 42150.00,
    "exit_price": 42380.00,
    "pnl": 23.50,
    "pnl_percent": 0.55,
    "virtual_balance_after": 5234.50
}
```

---

## Money Management

### Default-System

| Regel | Wert |
|-------|------|
| Risk per Trade | 1-2% |
| Stop Loss | 2-3x ATR |
| Trailing Stop | ATR-basiert |
| Take Profit | Min 1:2 R:R |
| Max gleichzeitige Positionen | 5 |
| Max Drawdown | 20% → System Pause |

### Strategie-spezifisches MM

Wenn eine Strategie eigenes Money Management mitbringt UND dieses durch Backtest validiert ist → Strategie-MM verwenden. Sonst → Default.

---

## Logging & Datenbank

### PostgreSQL (zentral)

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
    metadata JSONB
);
```

### Strategy Snapshot Schema

```sql
CREATE TABLE strategy_snapshots (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    strategy_id VARCHAR(50),
    tier VARCHAR(10),
    total_trades INTEGER,
    win_rate DECIMAL(5,2),
    profit_factor DECIMAL(10,4),
    sharpe_ratio DECIMAL(10,4),
    sortino_ratio DECIMAL(10,4),
    calmar_ratio DECIMAL(10,4),
    max_drawdown DECIMAL(10,4),
    composite_score DECIMAL(10,4),
    alpha DECIMAL(10,4),
    allocated_balance DECIMAL(20,8),
    current_balance DECIMAL(20,8),
    sharpe_1h DECIMAL(10,4),
    sharpe_4h DECIMAL(10,4),
    sharpe_24h DECIMAL(10,4)
);
```

---

## STATUS.md & Telegram

### Auto-generierter STATUS.md (alle 4h)

```markdown
# Trading Genesis 2 - Status

**Stand:** 2024-01-15 14:00 UTC | **Uptime:** 3d 14h | **Modus:** Paper Trading

## Portfolio
| Metrik | Wert |
|--------|------|
| Balance | $10,266.70 |
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

## Alerts (letzte 24h)
- ⚠️ 12:30 - Silver: Sharpe -15%
- ✅ 12:45 - Silver: Erholt auf -5%
- 🔄 08:00 - Gold/Silver Swap durchgeführt
```

Strategienamen werden dynamisch vergeben. Die Nachricht zeigt nur Rang und Performance.

### Telegram-Benachrichtigungen

Nutzt den bestehenden Telegram-Bot: `/home/rolf_vps/telegram-bot/`

**Regelmäßig:**
| Frequenz | Inhalt |
|----------|--------|
| Alle 4h | Kurz-Status (PnL, Trades, Alpha) |
| Täglich 08:00 | Tages-Report (Champions, Alerts) |
| Wöchentlich | Wochen-Summary + Discovery-Pipeline |

**Proaktive Alerts (Event-getriggert):**
| Event | Nachricht |
|-------|-----------|
| API-Fehler | Exchange/Daten-API down |
| Drawdown > 10% | Drawdown-Warnung |
| Champion-Swap | Rang-Änderung |
| System-Pause | Trading pausiert |
| Challenger schlägt Bronze | Neuer Champion |
| Bot-Restart | System neugestartet |

---

## Alle Agents

| # | Agent | Typ | Läuft | Implementierung |
|---|-------|-----|-------|-----------------|
| 1 | **Orchestrator** | Koordination | Alle 1-4h | Claude CLI |
| 2 | **Signal Engine** | Echtzeit | Kontinuierlich | Python Daemon |
| 3 | **Risk Engine** | Echtzeit | Bei jedem Signal | Python Daemon |
| 4 | **Order Executor** | Echtzeit | Bei validiertem Trade | Python Daemon |
| 5 | **Trade Monitor** | Echtzeit | Kontinuierlich | Python Daemon |
| 6 | **Analyzer Agent** | Analyse | Alle 1-4h | Claude CLI |
| 7 | **Optimizer Agent** | Analyse | Alle 1-4h / Täglich | Claude CLI |
| 8 | **Research Agent** | Discovery | Wöchentlich | Claude CLI |
| 9 | **Coder Agent** | Discovery | Bei neuer Hypothese | Claude CLI |
| 10 | **Evaluator Agent** | Discovery | Nach Backtests | Claude CLI |

Reduziert von 13 auf 10: Parser und Validator sind in den Research Agent integriert. Backtest-Ausführung ist automatisiert (VectorBT Script, kein eigener Agent).

---

## Technische Infrastruktur

### Kern-APIs

| Service | Zweck | Kosten |
|---------|-------|--------|
| Binance Testnet | Paper Trading | Kostenlos |
| Binance API | Live Marktdaten + WebSocket | Kostenlos |
| CoinGecko API | Benchmark (Marktcap) | Kostenlos |

### Optionale APIs (strategie-abhängig)

Werden bei Bedarf hinzugefügt (pending_requirements.json):
- News APIs (CryptoPanic, etc.)
- Sentiment APIs (LunarCrush, etc.)
- On-Chain APIs (Glassnode, etc.)
- Whale Tracking (Whale Alert, etc.)

### MCP Server

| Server | Zweck |
|--------|-------|
| ccxt-mcp | Exchange Daten + Trading |
| postgres-mcp | Datenbank |

### Software Stack

| Komponente | Technologie |
|------------|-------------|
| Echtzeit-Engine | Python (asyncio) |
| Backtesting | VectorBT (Primär), NautilusTrader (Optional) |
| Datenbank | PostgreSQL |
| KI-Entscheidungen | Claude Code CLI |
| Process Management | systemd |
| Notifications | Telegram Bot |
