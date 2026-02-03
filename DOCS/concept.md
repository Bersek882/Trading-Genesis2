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
Challenger schlägt Bronze (nach 24-48h)?
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
| 48h | ~200 | Hohe Signifikanz |

**Challenger-Testzeit:** Minimum 24h, Maximum 48h

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
│  2. Läuft 24-48 Stunden                                     │
│     └─ Sammelt mindestens 50-100 Trades                     │
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
| Maximum-Testzeit | 48h |
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
