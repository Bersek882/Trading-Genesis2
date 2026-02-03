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
│  │ VALIDATOR AGENT │ ← Prüft Vollständigkeit & Eignung      │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Agent | Aufgabe | Input | Output |
|-------|---------|-------|--------|
| **Research Agent** | Durchsucht GitHub, TradingView, Papers, Blogs nach Strategien | Suchkriterien | Rohe Strategie-Dokumente |
| **Parser Agent** | Extrahiert Entry/Exit-Regeln, Indikatoren, Timeframes, MM-Regeln | Rohe Dokumente | Strukturierte Strategie-Definition |
| **Validator Agent** | Prüft: Ist vollständig? Passt zu Krypto? Backtestbar? | Strukturierte Def. | Validierte Strategie + Score |

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
| **Coder Agent** | Übersetzt Strategie-Definition in ausführbaren Python-Code | Validierte Strategie | Backtestbarer Code |
| **Backtest Agent** | Führt Walk-Forward Tests auf historischen Daten durch | Code + Historische Daten | Performance-Metriken |
| **Evaluator Agent** | Bewertet: Sharpe, Drawdown, Win Rate, Robustheit | Metriken aller Strategien | Ranking + Empfehlung |

---

## Phase 3: Live Trading

```
┌─────────────────────────────────────────────────────────────┐
│                     LIVE TRADING                            │
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
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Agent | Aufgabe |
|-------|---------|
| **Signal Agent** | Wendet Top-Strategien auf Live-Daten an, generiert Signale |
| **Risk Agent** | Wendet Money Management an (Position Size, Stop Loss, etc.) |
| **Executor Agent** | Führt Trades aus (Paper oder Real) |

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
│                                                             │
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
│   Koordiniert alle Phasen, entscheidet wann was läuft:      │
│                                                             │
│   - Wann neue Strategien suchen?                            │
│   - Wann Backtests starten?                                 │
│   - Welche Strategien aktivieren/deaktivieren?              │
│   - Wann Parameter anpassen?                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Money Management

### Default-System (am Anfang)

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

Der Parser Agent extrahiert MM-Aspekte aus gefundenen Strategien:
- Spezifische Stop Loss Regeln?
- Position Sizing Methode?
- Take Profit Levels?

Der Risk Agent entscheidet:
```
IF Strategie-MM vorhanden AND validiert durch Backtest:
    -> Nutze Strategie-MM
ELSE:
    -> Nutze Default-MM
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

## Datenfluss

```
Internet (Strategien)
       |
       v
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Research   │ --> │   Parser    │ --> │  Validator  │
└─────────────┘     └─────────────┘     └─────────────┘
                                               |
                                               v
                                        ┌─────────────┐
                                        │   Coder     │
                                        └─────────────┘
                                               |
       ┌───────────────────────────────────────┘
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
