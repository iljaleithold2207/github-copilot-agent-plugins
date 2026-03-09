# 🛡️ DevSecOps Agent Plugin

> **Enterprise AI Agent Plugin für GitHub Copilot (VS Code & CLI)**
>
> Azure DevOps Fokus · Java Microservices · Alle Plugin-Komponenten demonstriert

---

## 🎯 Was ist ein Agent Plugin?

Agent Plugins sind **installierbare Pakete von KI-Anpassungen** für VS Code (ab v1.110) und GitHub Copilot CLI. Sie bündeln mehrere Komponenten in einem einzigen Paket, das Teams über ein Git-Repository teilen können.

---

## 🧠 Agent vs. Skill — Der wichtigste Unterschied

Das ist die Frage, die am häufigsten aufkommt, und dieses Plugin demonstriert den Unterschied praktisch:

### Agent = Autonomer Spezialist

```
User: "Mach mein Projekt sicher"
        ↓
   ┌─────────────┐
   │    AGENT     │ ← Hat eigene Tools (bash, view, codebase...)
   │              │ ← Entscheidet selbst, was er prüft
   │  1. Scannt   │ ← Erkundet autonom den Workspace
   │  2. Analysiert│ ← Priorisiert Findings nach Risiko
   │  3. Fixt     │ ← Kann Code ändern und validieren
   │  4. Prüft    │ ← Verifiziert seinen eigenen Fix
   │  5. Delegiert│ ← Kann andere Agents aufrufen (Subagent)
   └─────────────┘
```

**Eigenschaften eines Agents:**
- Startet autonom und iteriert bis zum Ergebnis
- Hat Zugriff auf Tools (`bash`, `edit`, `view`, `codebase`, `fetch`)
- Kann andere Agents delegieren (Subagent-Pattern)
- Trifft eigene Entscheidungen über Reihenfolge und Tiefe
- Kann vom Chat, von anderen Agents, oder per Hook getriggert werden

**In diesem Plugin:**
- `pipeline-architect` — Scannt das Projekt, wählt Pipeline-Architektur, generiert YAML, validiert, korrigiert
- `security-reviewer` — Findet selbstständig sicherheitsrelevante Dateien, prüft sie, reportet strukturiert

### Skill = Passives Template-System

```
User: "/pipeline"
        ↓
   ┌─────────────┐
   │    SKILL     │ ← Keine eigenen Tools
   │              │ ← Aktiviert nur bei explizitem Aufruf
   │  Template    │ ← Füllt ein festes Template aus
   │  ausfüllen   │ ← Kein autonomes Verhalten
   │  → Output    │ ← Einmaliger Input → Output Zyklus
   └─────────────┘
```

**Eigenschaften eines Skills:**
- Wird nur aktiviert, wenn User ihn explizit aufruft (Slash Command oder Keyword)
- Hat keine eigenen Tools — arbeitet mit dem übergebenen Context
- Folgt einem festen Template/Muster
- Kein Iterieren, kein Selbst-Korrigieren
- Portabel über VS Code, Copilot CLI und Copilot Coding Agent

**In diesem Plugin:**
- `/pipeline` — Gibt Azure DevOps Multi-Stage YAML nach festem Muster aus
- `/seccheck` — Wendet eine feste Checkliste auf übergebene Dateien an
- `/containers` — Liefert optimierte Java-Dockerfile-Templates

### Faustregel

| Frage | Agent | Skill |
|-------|-------|-------|
| Kann er selbst entscheiden, was er tut? | ✅ Ja | ❌ Nein |
| Hat er Tools (bash, edit, view)? | ✅ Ja | ❌ Nein |
| Kann er andere Agents aufrufen? | ✅ Ja (Subagent) | ❌ Nein |
| Iteriert er bis zum Ergebnis? | ✅ Ja | ❌ Einmal |
| Braucht er einen expliziten Aufruf? | ❌ Kann autonom starten | ✅ Immer |

---

## 📁 Plugin-Struktur

```
devsecops-plugin/
├── plugin.json                        # 📋 Plugin Manifest (required)
├── .mcp.json                          # 🔌 MCP Server Konfiguration
├── hooks.json                         # 🪝 Lifecycle Hooks
├── agents/                            # 🤖 Autonome Agents
│   ├── pipeline-architect.agent.md    #    → Autonomer Pipeline-Designer
│   └── security-reviewer.agent.md     #    → Autonomer Security-Prüfer (Subagent)
├── skills/                            # 📐 Passive Template-Skills
│   ├── pipeline-generator/
│   │   └── SKILL.md                   #    → /pipeline (Azure DevOps YAML)
│   ├── security-scanner/
│   │   └── SKILL.md                   #    → /seccheck (Checklisten-Scan)
│   └── container-ops/
│       └── SKILL.md                   #    → /containers (Java Dockerfiles)
└── .github/
    └── plugin/
        └── marketplace.json           # 🏪 Marketplace für Distribution
```

---

## 🪝 Hooks — Automatisierte Policy-Enforcement

Hooks feuern an bestimmten Punkten im Agent-Lifecycle, **ohne dass der User etwas tun muss**:

| Hook | Event | Was passiert |
|------|-------|-------------|
| Secret Scanner | `preToolUse` → `edit` | Scannt jede Code-Änderung auf hardcoded Secrets **bevor** sie angewendet wird. Blockiert bei Fund. |
| Audit Logger | `postToolUse` → `bash` | Loggt jeden Terminal-Befehl in `.devsecops-audit.log` für Compliance-Nachweise. |

Hooks sind vergleichbar mit Git Pre-Commit-Hooks, aber auf Agent-Ebene. Sie laufen automatisch — der Agent kann sie nicht umgehen.

---

## 🔌 MCP Server — Enterprise-sichere Auswahl

Alle MCP Server in diesem Plugin sind **offizielle Anthropic Reference Servers** und laufen lokal. Hier die Übersicht, was enterprise-tauglich ist:

### ✅ Im Plugin enthalten (keine Freigabe nötig)

| Server | Was er tut | Netzwerk? | Credentials? |
|--------|-----------|-----------|-------------|
| **filesystem** | Dateien lesen/schreiben im Workspace | ❌ Lokal | ❌ Keine |
| **git** | Git log, diff, blame, branches | ❌ Lokal | ❌ Keine |
| **memory** | Knowledge-Graph über Sessions hinweg | ❌ Lokal | ❌ Keine |
| **sequential-thinking** | Strukturiertes Problemlösen in Schritten | ❌ Lokal | ❌ Keine |
| **fetch** | Web-Inhalte holen (Doku-Lookup) | ⚠️ HTTP nach außen | ❌ Keine |

### 💡 Weitere empfehlenswerte MCP Server (nicht im Plugin, aber enterprise-tauglich)

| Server | Was er tut | Warum enterprise-safe |
|--------|-----------|----------------------|
| **@modelcontextprotocol/server-time** | Zeitzonen-Konvertierung | Komplett lokal, kein Netzwerk |
| **DuckDuckGo MCP** | Web-Suche ohne API-Key | Kein Account/Token nötig, aber Netzwerk |
| **Playwright MCP** | Browser-Automatisierung/Testing | Lokal, nützlich für E2E-Tests |

### ⚠️ MCP Server die Freigabe brauchen (nicht im Plugin)

| Server | Warum Freigabe nötig |
|--------|---------------------|
| GitHub MCP | Braucht Personal Access Token |
| Slack MCP | Braucht Slack Bot Token + Workspace-Zugriff |
| Azure DevOps MCP | Braucht PAT + Org-Zugriff |
| Notion/Jira/etc. | API-Keys + Datenzugriff |

**Empfehlung**: Starte mit den 5 lokalen Servern im Plugin. Wenn das Team den Mehrwert sieht, kann man GitHub/Azure DevOps MCP über die IT-Security freigeben lassen.

---

## 🚀 Installation

### Lokal testen (empfohlen zum Start)

```json
// VS Code Settings (settings.json)
{
  "chat.plugins.enabled": true,
  "chat.plugins.paths": [
    "C:/Pfad/zu/devsecops-plugin"
  ]
}
```

### Über Git-Repository verteilen

```json
// VS Code Settings
{
  "chat.plugins.enabled": true,
  "chat.plugins.marketplaces": [
    "iljaleithold2207/github-copilot-agent-plugins"
  ]
}
```

### Copilot CLI

```bash
# Direkt installieren
copilot plugin install iljaleithold2207/github-copilot-agent-plugins

# Oder über Marketplace
copilot plugin marketplace add iljaleithold2207/github-copilot-agent-plugins
copilot plugin install devsecops-toolkit@devsecops-marketplace
```

---

## 💡 Demo-Szenarien

### 1. Agent autonom arbeiten lassen

> "Analysiere mein Java-Projekt und erstelle eine sichere Azure DevOps Pipeline"

→ Der **pipeline-architect Agent** scannt autonom das Projekt, erkennt Spring Boot + Maven, generiert die Pipeline, delegiert Security-Check an den **security-reviewer Agent**.

### 2. Skill als schnelles Template

> `/pipeline`

→ Der **pipeline-generator Skill** gibt direkt das Azure DevOps YAML-Template aus — ohne autonome Analyse.

### 3. Hook live demonstrieren

> Schreibe `String password = "admin123";` in eine Java-Datei

→ Der **preToolUse Hook** blockiert die Änderung mit Warnung.

### 4. Agent vs. Skill direkt vergleichen

> Erst: `/seccheck` auf ein Dockerfile (Skill — fixe Checkliste)
> Dann: "Prüfe mein gesamtes Projekt auf Sicherheitsprobleme" (Agent — autonome Exploration)

→ Zeigt den Unterschied zwischen passivem Template und autonomem Verhalten.

---

## 📊 Architektur

```
┌──────────────────────────────────────────────────────────┐
│                VS Code / Copilot CLI                      │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Agent Plugin Runtime                    │ │
│  │                                                      │ │
│  │  🤖 AGENTS (autonom, mit Tools)                     │ │
│  │  ┌──────────────┐     ┌──────────────┐              │ │
│  │  │  Pipeline     │────▶│  Security    │              │ │
│  │  │  Architect    │     │  Reviewer    │              │ │
│  │  │  (orchestriert)│    │  (Subagent)  │              │ │
│  │  └──────┬───────┘     └──────────────┘              │ │
│  │         │ nutzt                                      │ │
│  │  📐 SKILLS (passiv, template-basiert)               │ │
│  │  ┌────────────┬────────────┬────────────┐           │ │
│  │  │ /pipeline  │ /seccheck  │ /containers│           │ │
│  │  │ AzDO YAML  │ Checkliste │ Java Docker│           │ │
│  │  └────────────┴────────────┴────────────┘           │ │
│  │                                                      │ │
│  │  🪝 HOOKS (automatisch, policy-enforcement)         │ │
│  │  ┌──────────────────┬──────────────────┐            │ │
│  │  │ pre: Secret Scan │ post: Audit Log  │            │ │
│  │  └──────────────────┴──────────────────┘            │ │
│  │                                                      │ │
│  │  🔌 MCP SERVER (externe Tool-Anbindung)             │ │
│  │  ┌────┬────┬────────┬───────┬───────────────┐      │ │
│  │  │FS  │Git │Memory  │Fetch  │Seq. Thinking  │      │ │
│  │  │lokal│lokal│lokal  │extern │lokal           │      │ │
│  │  └────┴────┴────────┴───────┴───────────────┘      │ │
│  └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## 🔗 Referenzen

- [VS Code Agent Plugins Doku](https://code.visualstudio.com/docs/copilot/customization/agent-plugins)
- [GitHub Copilot CLI Plugin-Erstellung](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating)
- [Agent Skills Spezifikation](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [MCP Reference Servers](https://github.com/modelcontextprotocol/servers)
- [Ken Muse: Creating Agent Plugins](https://www.kenmuse.com/blog/creating-agent-plugins-for-vs-code-and-copilot-cli/)

---

*Erstellt von Ilja Leithold · März 2026*
