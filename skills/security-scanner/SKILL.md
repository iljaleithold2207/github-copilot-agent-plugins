---
name: security-scanner
description: "Analysiert Dockerfiles, Kubernetes-Manifeste und Azure DevOps Pipelines auf Security-Fehlkonfigurationen — passives Template-basiertes Scanning"
command: /seccheck
---

# Security Scanner Skill

Statische Sicherheitsanalyse für Projektartefakte nach festen Prüf-Checklisten. Kein autonomes Verhalten — wird nur bei explizitem Aufruf aktiv.

## Skill vs. Agent — Unterschied am Beispiel Security

| Aspekt | Dieser Skill (`/seccheck`) | Der Agent (`security-reviewer`) |
|--------|---------------------------|--------------------------------|
| **Aktivierung** | Nur wenn User `/seccheck` tippt | Autonom, auch durch Delegation |
| **Verhalten** | Wendet feste Checkliste an | Entscheidet selbst, was er prüft |
| **Tools** | Keine — arbeitet mit übergebenem Context | Hat `bash`, `view`, `codebase` |
| **Iteration** | Einmalig: Input → Output | Scannt, findet, prüft nach, korrigiert |
| **Scope** | Was der User zeigt | Erkundet selbst den Workspace |

**Wann Skill, wann Agent?**
- `/seccheck` wenn du eine schnelle Checklisten-Prüfung für eine spezifische Datei brauchst
- `security-reviewer` Agent wenn du eine umfassende, autonome Sicherheitsanalyse des Projekts willst

## When to Activate

- User tippt `/seccheck` im Chat
- User fragt nach Security-Check für eine spezifische Datei
- User will wissen, ob ein Dockerfile oder K8s-Manifest sicher ist

## Dockerfile Security-Checkliste

| Check | Severity | Pattern |
|-------|----------|---------|
| Root-User | HIGH | Kein `USER`-Directive oder `USER root` |
| Latest Tag | MEDIUM | `FROM image:latest` oder kein Tag |
| Secrets im Build | CRITICAL | `ARG`/`ENV` mit Passwörtern, Tokens, Keys |
| Package-Cache | LOW | Kein `rm -rf /var/cache` nach Install |
| Multi-Stage fehlt | MEDIUM | Einzelnes `FROM` mit Build-Tools im finalen Image |
| COPY vs ADD | LOW | `ADD` für lokale Dateien (COPY bevorzugen) |
| Healthcheck | LOW | Kein `HEALTHCHECK`-Directive |
| JDK statt JRE | MEDIUM | Finales Image nutzt JDK statt JRE |

### Remediation: Sicheres Java-Dockerfile

```dockerfile
# Build Stage — JDK nur hier
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B
COPY src src
RUN ./mvnw package -DskipTests -B

# Runtime Stage — nur JRE, non-root
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

## Kubernetes Manifest-Checkliste

| Check | Severity | Was prüfen |
|-------|----------|------------|
| Privilegierte Container | CRITICAL | `securityContext.privileged: true` |
| Keine Resource Limits | HIGH | Fehlende `resources.limits` |
| Host Network | HIGH | `hostNetwork: true` |
| Keine NetworkPolicy | MEDIUM | Namespace ohne NetworkPolicy |
| Default ServiceAccount | MEDIUM | Kein expliziter `serviceAccountName` |
| Keine Readiness Probe | MEDIUM | Fehlende `readinessProbe` |
| Mutable Image Tag | LOW | `imagePullPolicy` nicht `Always` bei Tags wie `latest` |

## Azure DevOps Pipeline-Checkliste

| Check | Severity | Was prüfen |
|-------|----------|------------|
| Inline-Secrets | CRITICAL | Passwörter/Tokens direkt in YAML |
| Keine Variable Groups | HIGH | Secrets nicht in Variable Groups |
| Kein Security-Scan-Stage | HIGH | Pipeline ohne SAST/SCA Stage |
| Offene Trigger | MEDIUM | `trigger: '*'` ohne Branch-Filter |
| Kein Approval Gate | MEDIUM | Prod-Deployment ohne Environment Approval |
| Fehlende Caching | LOW | Keine Cache-Tasks für Dependencies |

## Output Format

```markdown
## Security Scan: [Dateityp + Dateiname]

**Geprüfte Checks**: [n von n bestanden]
**Risk Level**: CRITICAL | HIGH | MEDIUM | LOW | PASS

### Befund 1: [Titel]
- **Severity**: CRITICAL
- **Location**: `Datei:Zeile`
- **Problem**: [Beschreibung]
- **Fix**: [konkreter Code/Config-Fix]
- **Referenz**: OWASP / CIS Benchmark / BSI Grundschutz

### Empfohlene Maßnahmen (priorisiert)
1. [Kritischster Fix zuerst]
```
