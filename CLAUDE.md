# Trading Genesis 2

Selbstverbesserndes, erfolgreiches Krypto-Trading-System - gebaut mit Claude Code Agents, nicht Algorithmen.

---

## Was ist Claude Code?

Claude Code ist ein CLI-Tool von Anthropic das Claude als autonomen Coding-Agenten bereitstellt. Es kann:
- Code lesen, schreiben, ausführen
- Im Internet recherchieren (WebSearch, WebFetch)
- Dateien und Projekte verwalten
- Shell-Befehle ausführen
- Mit externen APIs via MCP kommunizieren

## Agents & Subagents

**Agent:** Ein autonomer Claude-Prozess der eine Aufgabe selbstständig löst.

**Subagent:** Ein Agent der von einem anderen Agent gestartet wird (via Task Tool). Hat eigenen Kontext, kann spezialisiert sein.

```
Orchestrator (Haupt-Agent)
    │
    ├─► Signal Agent (Subagent)
    ├─► Research Agent (Subagent)
    └─► Analyzer Agent (Subagent)
```

**Vorteile:**
- Parallele Ausführung möglich
- Spezialisierte Aufgaben
- Eigener Kontext = weniger Überlastung
- Können im Hintergrund laufen

## MCP (Model Context Protocol)

Verbindet Claude mit externen Tools/APIs:
- `ccxt-mcp` → Exchange-Anbindung
- `postgres-mcp` → Datenbank
- Custom MCP Server möglich

## Relevante Fähigkeiten für dieses Projekt

| Fähigkeit | Nutzung |
|-----------|---------|
| **Task Tool** | Spawnt Subagents für parallele Arbeit |
| **WebSearch** | Research Agent sucht Strategien |
| **Bash** | Code ausführen, Backtests starten |
| **Read/Write/Edit** | Code und Configs verwalten |
| **MCP** | Exchange-APIs, Datenbank |
| **Headless Mode** | Autonomer 24/7 Betrieb ohne UI |

## Headless Mode

Claude kann ohne Terminal-UI laufen:
```bash
claude -p "Führe Trading-Loop aus" --output-format json
```

Ermöglicht autonomen Dauerbetrieb.
