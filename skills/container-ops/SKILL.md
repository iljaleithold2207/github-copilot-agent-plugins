---
name: container-ops
description: "Generiert produktionsreife Dockerfiles für Java-Microservices (Spring Boot, Quarkus) mit Security Best Practices"
command: /containers
---

# Container Operations Skill — Java Dockerfiles

Generiert sichere, optimierte Dockerfiles für Java-Microservices. Fokus auf Spring Boot und Quarkus mit Maven oder Gradle.

## Skill-Charakter (passiv, template-basiert)

Dieser Skill liefert **fertige Dockerfile-Templates** basierend auf dem Java-Framework und Build-Tool. Er analysiert nicht autonom das Projekt — er wird vom User aufgerufen und füllt das passende Template aus.

## When to Activate

- User tippt `/containers` im Chat
- User fragt nach einem Dockerfile für Java / Spring Boot / Quarkus
- User will ein bestehendes Java-Dockerfile optimieren

## Template: Spring Boot + Maven

```dockerfile
# ============================================
# Stage 1: Build (JDK — wird nicht ausgeliefert)
# ============================================
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build

# Dependency-Layer cachen (ändert sich selten)
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B

# Source kopieren und bauen
COPY src src
RUN ./mvnw package -DskipTests -B

# ============================================
# Stage 2: Runtime (nur JRE — minimale Angriffsfläche)
# ============================================
FROM eclipse-temurin:21-jre-alpine

# Non-root User erstellen
RUN addgroup -S app && adduser -S app -G app

WORKDIR /app

# Nur das JAR aus dem Builder kopieren
COPY --from=builder /build/target/*.jar app.jar

# Niemals als root laufen
USER app

EXPOSE 8080

# Health Check für Kubernetes readiness/liveness
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

# JVM-Flags für Container-Umgebung
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseG1GC", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

## Template: Spring Boot + Gradle

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build
COPY build.gradle.kts settings.gradle.kts gradlew ./
COPY gradle gradle
RUN ./gradlew dependencies --no-daemon
COPY src src
RUN ./gradlew bootJar --no-daemon -x test

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=builder /build/build/libs/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

## Template: Quarkus (Native Build)

```dockerfile
FROM ghcr.io/graalvm/native-image-community:21 AS builder
WORKDIR /build
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B
COPY src src
RUN ./mvnw package -Pnative -DskipTests -B

FROM alpine:3.19
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=builder /build/target/*-runner app
USER app
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=2s --start-period=5s --retries=3 \
  CMD wget -q --spider http://localhost:8080/q/health || exit 1
ENTRYPOINT ["./app"]
```

## Checkliste für jedes generierte Dockerfile

Jedes Dockerfile MUSS folgende Kriterien erfüllen:

1. **Multi-Stage Build** — JDK/Build-Tools nie im finalen Image
2. **Non-Root User** — `USER app` nach `adduser`
3. **Alpine Base** — minimale Image-Größe und Angriffsfläche
4. **Pinned Version** — nie `:latest`, immer konkrete Version
5. **HEALTHCHECK** — für K8s-Integration
6. **Dependency-Layer-Caching** — `COPY pom.xml` vor `COPY src`
7. **Container-aware JVM-Flags** — `UseContainerSupport`, `MaxRAMPercentage`
8. **Kein ADD** — immer `COPY` für lokale Dateien
9. **Kein Secrets** — keine `ARG`/`ENV` mit Credentials
