# Trading Genesis 2 - Konzept

## Überblick

Dynamische Strategie-Entdeckung durch Agents - keine hardcoded Strategien.

---

## Phase 1: Strategie-Entdeckung

```
┌─────────────────────────────────────────────────────────────┐
│                   STRATEGY DISCOVERY                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐                                        │
│  │ RESEARCH AGENT  │ ← Sucht im Internet nach Strategien    │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ PARSER AGENT    │ ← Extrahiert strukturierte Daten       │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ VALIDATOR AGENT │ ← Prüft Vollständigkeit & Machbarkeit  │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│      ┌─────────┐                                            │
│      │ OK?     │                                            │
│      └────┬────┘                                            │
│       ╱       ╲                                             │
│     JA        NEIN → meldet an Orchestrator                 │
│      │              → Strategie zurückgestellt              │
│      ▼              → in pending_requirements.json          │
│   CODER AGENT                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Agent | Aufgabe | Input | Output |
|-------|---------|-------|--------|
| **Research Agent** | Durchsucht GitHub, TradingView, Papers, Blogs | Suchkriterien | Rohe Strategie-Dokumente |
| **Parser Agent** | Extrahiert Entry/Exit-Regeln, Indikatoren, Timeframes, MM-Regeln | Rohe Dokumente | Strukturierte Strategie-Definition |
| **Validator Agent** | Prüft Vollständigkeit, Krypto-Eignung, **technische Machbarkeit** | Strukturierte Def. | Validiert ODER Zurückgestellt |

### Validator Agent - Technische Machbarkeit

Der Validator prüft:
1. Sind alle Entry/Exit-Regeln definiert?
2. Passt die Strategie zu Krypto-Märkten?
3. **Haben wir die benötigten Datenquellen?**

Wenn Datenquellen fehlen (z.B. Strategie braucht Whale-Daten, aber kein API konfiguriert):

```
Validator → Orchestrator: "Strategie X braucht Whale Alert API"
                ↓
Orchestrator → pending_requirements.json: Eintrag hinzufügen
                ↓
Strategie Status: "PENDING_REQUIREMENTS"
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
          "reason": "Strategie benötigt Echtzeit-Whale-Transaktionen",
          "alternatives": ["Arkham API", "Etherscan Whale Watch"]
        }
      ],
      "added_at": "2024-01-15T10:30:00Z",
      "status": "waiting_for_setup"
    }
  ]
}
```

Später kann die Technik angepasst werden → Strategie wird reaktiviert.

---

## Phase 2: Backtesting & Bewertung

```
┌─────────────────────────────────────────────────────────────┐
│                   BACKTESTING PIPELINE                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐                                        │
│  │ CODER AGENT     │ ← Implementiert Strategie als Code     │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ BACKTEST AGENT  │ ← Führt Walk-Forward Backtests durch   │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ EVALUATOR AGENT │ ← Bewertet Ergebnisse, Ranking         │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Agent | Aufgabe | Input | Output |
|-------|---------|-------|--------|
| **Coder Agent** | Übersetzt Strategie in Python-Code (Backtrader) | Validierte Strategie | Backtestbarer Code |
| **Backtest Agent** | Walk-Forward Tests mit realistischer Simulation | Code + OHLCV Daten | Performance-Metriken |
| **Evaluator Agent** | Bewertet Sharpe, Drawdown, Win Rate, Robustheit | Metriken | Ranking + Empfehlung |

---

## Phase 3: Paper Trading (dann Live)

```
┌─────────────────────────────────────────────────────────────┐
│                     PAPER / LIVE TRADING                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐      ┌─────────────────┐               │
│  │ SIGNAL AGENT    │ ──── │ RISK AGENT      │               │
│  └────────┬────────┘      └────────┬────────┘               │
│           │                        │                        │
│           └──────────┬─────────────┘                        │
│                      ▼                                      │
│             ┌─────────────────┐                             │
│             │ EXECUTOR AGENT  │                             │
│             └─────────────────┘                             │
│                      │                                      │
│              ┌──────┴──────┐                               │
│              ▼             ▼                               │
│         [TESTNET]     [MAINNET]                            │
│         Paper Mode    Live Mode                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Agent | Aufgabe |
|-------|---------|
| **Signal Agent** | Wendet Top-Strategien auf Live-Daten an |
| **Risk Agent** | Money Management (Position Size, Stop Loss) |
| **Executor Agent** | Orders an Exchange (Testnet oder Mainnet) |

---

## Phase 4: Selbstverbesserung

```
┌─────────────────────────────────────────────────────────────┐
│                   SELF-IMPROVEMENT                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐                                        │
│  │ MONITOR AGENT   │ ← Trackt alle Trades & Performance     │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ ANALYZER AGENT  │ ← Analysiert: Was funktioniert?        │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ OPTIMIZER AGENT │ ← Passt Parameter/Gewichtungen an      │
│  └─────────────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│    Triggert Research Agent für neue Strategien              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Orchestrator

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    ORCHESTRATOR AGENT                       │
│                                                             │
│   Koordiniert alle Phasen:                                  │
│                                                             │
│   - Wann neue Strategien suchen?                            │
│   - Wann Backtests starten?                                 │
│   - Welche Strategien aktivieren/deaktivieren?              │
│   - Wann Parameter anpassen?                                │
│   - Verwaltet pending_requirements.json                     │
│   - Reaktiviert Strategien wenn Requirements erfüllt        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
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
│ - Max Open Positions: 5                │
│ - Max Drawdown: 20% -> Pause           │
└────────────────────────────────────────┘
```

### Strategie-spezifisches MM

Parser Agent extrahiert MM-Aspekte aus Strategien. Risk Agent entscheidet:
```
IF Strategie-MM vorhanden AND validiert durch Backtest:
    -> Nutze Strategie-MM
ELSE:
    -> Nutze Default-MM
```

---

## Paper Trading - Realismus

### Anforderungen für 100% Realismus

| Aspekt | Simulation |
|--------|------------|
| Gebühren | Exchange-spezifisch (Binance: 0.1% Spot) |
| Slippage | ATR-basiert oder Orderbook-basiert |
| Partial Fills | Volumen-basiert |
| Latenz | 50-500ms Delay |
| Spread | Live Bid/Ask |

### Empfohlener Ansatz: Exchange Testnet

| Exchange | Testnet URL | Features |
|----------|-------------|----------|
| **Binance** | testnet.binancefuture.com | Spot + Futures, echte Preise |
| **Bybit** | api-demo.bybit.com | 50k USDT virtuell |

**Vorteil:** Exakt gleiche API wie Live → nahtloser Umstieg

### Umstieg Paper → Live

```python
# Paper Trading (nur URL unterschied)
exchange = ccxt.binance({
    'apiKey': 'TESTNET_KEY',
    'secret': 'TESTNET_SECRET',
    'urls': {'api': 'https://testnet.binancefuture.com'}
})

# Live Trading
exchange = ccxt.binance({
    'apiKey': 'LIVE_KEY',
    'secret': 'LIVE_SECRET'
    # Default URL ist bereits Live
})
```

---

## Alle Agents

| # | Agent | Phase | Läuft |
|---|-------|-------|-------|
| 1 | **Orchestrator** | Alle | Kontinuierlich |
| 2 | **Research Agent** | Discovery | Periodisch |
| 3 | **Parser Agent** | Discovery | Bei neuer Strategie |
| 4 | **Validator Agent** | Discovery | Bei neuer Strategie |
| 5 | **Coder Agent** | Backtest | Bei validierter Strategie |
| 6 | **Backtest Agent** | Backtest | Bei neuer Strategie |
| 7 | **Evaluator Agent** | Backtest | Nach Backtests |
| 8 | **Signal Agent** | Trading | Kontinuierlich |
| 9 | **Risk Agent** | Trading | Bei jedem Signal |
| 10 | **Executor Agent** | Trading | Bei validiertem Trade |
| 11 | **Monitor Agent** | Improvement | Kontinuierlich |
| 12 | **Analyzer Agent** | Improvement | Periodisch |
| 13 | **Optimizer Agent** | Improvement | Bei Bedarf |

---

## Technische Infrastruktur

### Kern-APIs (immer benötigt)

| Service | Zweck | Kosten |
|---------|-------|--------|
| **Binance Testnet** | Paper Trading | Kostenlos |
| **Binance API** | Live Marktdaten, OHLCV | Kostenlos |
| **GitHub API** | Strategy Research | Kostenlos |

### Optionale APIs (strategie-abhängig)

Werden dynamisch hinzugefügt wenn eine Strategie sie benötigt:

- News APIs (CryptoPanic, etc.)
- Sentiment APIs (LunarCrush, etc.)
- On-Chain APIs (Glassnode, etc.)
- Whale Tracking (Whale Alert, etc.)

Diese werden in `pending_requirements.json` erfasst und bei Bedarf eingerichtet.

### MCP Server

| MCP Server | Zweck |
|------------|-------|
| **ccxt-mcp** | Exchange Daten + Trading |
| **postgres-mcp** | Datenbank für Trades, Strategien |

Weitere MCP Server werden bei Bedarf hinzugefügt.

### Backtesting Engine

**Backtrader** (empfohlen):
- Slippage-Simulation (slip_perc, slip_fixed)
- Commission/Fees
- Partial Fills
- Live-Trading-fähig (gleicher Code)

---

## Datenfluss

```
Internet (Strategien)
       |
       v
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Research   │ --> │   Parser    │ --> │  Validator  │
└─────────────┘     └─────────────┘     └─────────────┘
                                               |
                                     ┌────────┴────────┐
                                     v                 v
                              ┌───────────┐    ┌──────────────┐
                              │   Coder   │    │   PENDING    │
                              └─────┬─────┘    │ REQUIREMENTS │
                                    |          └──────────────┘
       ┌────────────────────────────┘
       v
┌─────────────┐     ┌─────────────┐
│  Backtest   │ --> │  Evaluator  │ --> Strategy Pool
└─────────────┘     └─────────────┘
                                               |
       ┌───────────────────────────────────────┘
       v
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Signal    │ --> │    Risk     │ --> │  Executor   │
└─────────────┘     └─────────────┘     └─────────────┘
                                               |
                                               v
                                        ┌─────────────┐
                                        │   Monitor   │
                                        └─────────────┘
                                               |
       ┌───────────────────────────────────────┘
       v
┌─────────────┐     ┌─────────────┐
│  Analyzer   │ --> │  Optimizer  │ --> Feedback Loop
└─────────────┘     └─────────────┘
```

---

## Quellen für Research Agent

| Quelle | Typ | Qualität |
|--------|-----|----------|
| GitHub (Pine Script Repos) | Code + Doku | Hoch |
| TradingView Community | Strategien | Mittel |
| arXiv / SSRN | Papers | Hoch |
| Medium / Blogs | Tutorials | Variabel |
| QuantConnect | Algorithmen | Hoch |
