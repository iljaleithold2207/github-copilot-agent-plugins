---
name: pipeline-generator
description: "Generiert Multi-Stage Azure DevOps Pipeline-Konfigurationen mit integrierter DevSecOps-Toolchain für Java-Microservices"
command: /pipeline
---

# Pipeline Generator Skill

Generiert produktionsreife Azure DevOps CI/CD Pipeline-Konfigurationen mit Security Scanning, Quality Gates und Multi-Environment Deployment.

## Was macht diesen SKILL anders als einen Agent?

Dieser Skill ist ein **passives Template-System**:
- Er wird **nur aktiviert**, wenn der User `/pipeline` tippt oder explizit nach einer Pipeline fragt
- Er folgt einem **festen Template** und passt es an Parameter an
- Er **entscheidet nicht autonom** — er füllt Templates aus
- Er hat **keine Tools** — er kann nicht selbst Dateien scannen oder Befehle ausführen

Im Gegensatz dazu würde der `pipeline-architect` **Agent** autonom das Projekt analysieren, die passende Pipeline-Struktur wählen, generieren, validieren und korrigieren.

**Faustregel**: Skill = Rezept zum Nachkochen | Agent = Koch, der selbst entscheidet

## When to Activate

- User tippt `/pipeline` im Chat
- User fragt nach Azure DevOps Pipeline-YAML
- User will Security Scanning in eine bestehende Pipeline einbauen

## Azure DevOps Multi-Stage Pipeline Template (Java/Maven)

```yaml
trigger:
  branches:
    include: [main, develop, release/*]
  paths:
    exclude: [docs/*, README.md]

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: 'devsecops-vars'           # Variable Group mit Secrets
  - name: imageRepository
    value: 'myacr.azurecr.io/myapp'
  - name: MAVEN_CACHE_FOLDER
    value: $(Pipeline.Workspace)/.m2/repository

stages:
  # ──────────────────────────────────────────────
  # Stage 1: Build & Unit Test
  # ──────────────────────────────────────────────
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: BuildAndTest
        steps:
          - task: Cache@2
            displayName: 'Cache Maven Dependencies'
            inputs:
              key: 'maven | "$(Agent.OS)" | pom.xml'
              path: $(MAVEN_CACHE_FOLDER)

          - task: Maven@4
            displayName: 'Maven Build & Test'
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'clean verify'
              options: '-B -Dmaven.repo.local=$(MAVEN_CACHE_FOLDER)'
              publishJUnitResults: true
              testResultsFiles: '**/target/surefire-reports/TEST-*.xml'
              javaHomeOption: 'JDKVersion'
              jdkVersionOption: '1.21'

          - task: PublishCodeCoverageResults@2
            displayName: 'Publish JaCoCo Coverage'
            inputs:
              codeCoverageTool: 'JaCoCo'
              summaryFileLocation: '**/target/site/jacoco/jacoco.xml'
              reportDirectory: '**/target/site/jacoco'

  # ──────────────────────────────────────────────
  # Stage 2: Security Scanning
  # ──────────────────────────────────────────────
  - stage: SecurityScan
    displayName: 'Security Scanning'
    dependsOn: Build
    jobs:
      - job: SAST
        displayName: 'SonarQube SAST'
        steps:
          - task: SonarQubePrepare@6
            inputs:
              SonarQube: 'sonarqube-connection'   # Service Connection Name
              scannerMode: 'Other'
              extraProperties: |
                sonar.projectKey=$(Build.Repository.Name)
                sonar.coverage.jacoco.xmlReportPaths=**/target/site/jacoco/jacoco.xml

          - task: Maven@4
            displayName: 'SonarQube Analysis'
            inputs:
              mavenPomFile: 'pom.xml'
              goals: 'sonar:sonar'
              options: '-B'

          - task: SonarQubePublish@6
            displayName: 'Publish Quality Gate Result'
            inputs:
              pollingTimeoutSec: '300'

      - job: DependencyScan
        displayName: 'Trivy Dependency Scan'
        steps:
          - script: |
              curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
              trivy fs --severity HIGH,CRITICAL --exit-code 1 --format table .
            displayName: 'Trivy Filesystem Scan'

  # ──────────────────────────────────────────────
  # Stage 3: Container Build & Scan
  # ──────────────────────────────────────────────
  - stage: ContainerBuild
    displayName: 'Container Build & Scan'
    dependsOn: SecurityScan
    jobs:
      - job: BuildAndScanImage
        steps:
          - task: Docker@2
            displayName: 'Build Container Image'
            inputs:
              containerRegistry: 'acr-connection'   # Service Connection
              repository: '$(imageRepository)'
              command: 'build'
              Dockerfile: 'Dockerfile'
              tags: '$(Build.BuildId)'

          - script: |
              trivy image --severity HIGH,CRITICAL --exit-code 1 \
                $(imageRepository):$(Build.BuildId)
            displayName: 'Trivy Image Scan'

          - task: Docker@2
            displayName: 'Push to ACR'
            inputs:
              containerRegistry: 'acr-connection'
              repository: '$(imageRepository)'
              command: 'push'
              tags: '$(Build.BuildId)'

  # ──────────────────────────────────────────────
  # Stage 4: Deploy to Dev (automatisch)
  # ──────────────────────────────────────────────
  - stage: DeployDev
    displayName: 'Deploy → Dev'
    dependsOn: ContainerBuild
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: DeployToDev
        environment: 'dev'                     # Kein Approval Gate
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  displayName: 'Deploy to AKS Dev'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-dev-connection'
                    namespace: 'dev'
                    manifests: 'k8s/dev/*.yml'
                    containers: '$(imageRepository):$(Build.BuildId)'

  # ──────────────────────────────────────────────
  # Stage 5: Deploy to Prod (manuelles Approval)
  # ──────────────────────────────────────────────
  - stage: DeployProd
    displayName: 'Deploy → Production'
    dependsOn: ContainerBuild
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToProd
        environment: 'production'              # Hat Approval Gate konfiguriert
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  displayName: 'Deploy to AKS Production'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-prod-connection'
                    namespace: 'production'
                    manifests: 'k8s/prod/*.yml'
                    containers: '$(imageRepository):$(Build.BuildId)'
```

## Anpassungs-Regeln

1. Immer nach dem **Azure DevOps Projektnamen** und existierenden Service Connections fragen
2. Sprache wird aus Projektdateien erkannt: `pom.xml` → Java/Maven, `build.gradle` → Java/Gradle
3. **Caching** für Maven-Dependencies einbauen
4. **Quality Gates**: Pipeline failt bei Test-Fehlschlägen, Coverage < 80%, kritischen Vulnerabilities
5. **Environment Promotion**: Dev (auto) → Prod (manuelles Approval über Azure DevOps Environment)
6. Variable Groups für Secrets, nie Inline-Variablen mit sensiblen Werten
