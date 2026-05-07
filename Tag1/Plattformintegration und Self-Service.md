# Plattformintegration und Self-Service-Automatisierung

Vervollständige dein internes Developer Portal, indem du Backstage mit GitHub, Kubernetes, Monitoring-Tools und Dokumentationssystemen integrierst.

In dieser Lektion konfigurierst du GitHub OAuth für die Authentifizierung, implementierst TechDocs für Documentation as Code, bindest Monitoring-Dashboards ein und misst Verbesserungen der Developer Experience für die TaskFlow-Plattform.

---

## Lernmaterial

Du hast bereits das TaskFlow Dashboard Plugin erstellt und Software Templates für das Projekt-Scaffolding gebaut. Jetzt wird Backstage mit dem weiteren Plattform-Ökosystem verbunden: GitHub, Kubernetes, Monitoring-Tools und Dokumentationssysteme.

Diese Integrationen machen Backstage vom eigenständigen Portal zum zentralen Hub für Entwickler-Workflows. Entwickler können Pull Requests prüfen, Deployment-Status einsehen, Metrik-Dashboards öffnen und Dokumentation lesen, ohne Backstage zu verlassen.

Das reduziert Kontextwechsel und erhöht die Produktivität.

In dieser abschließenden Lektion lernst du:

* GitHub-Integration
* Kubernetes-Plugin-Konfiguration
* Anbindung von Monitoring-Tools
* TechDocs für Documentation as Code
* Authentifizierung und Autorisierung
* Messung des Einflusses deines internen Developer Portals auf die Developer Experience

---

## Backstage mit GitHub integrieren

Die GitHub-Integration ermöglicht:

* Repository-Informationen auf Catalog-Seiten anzeigen
* Aktuelle Commits und Pull Requests anzeigen
* GitHub Actions Workflows verlinken
* Repository-Contributors anzeigen
* Issues und Pull Requests direkt aus Backstage erstellen

### Konfiguration in `app-config.yaml`

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

---

## GitHub App erstellen

Für Produktionsumgebungen wird eine GitHub App empfohlen. Sie ist sicherer als Personal Access Tokens und besser für Teams verwaltbar.

### Vorgehen

1. Öffne GitHub Settings → Developer Settings → GitHub Apps.
2. Klicke auf **New GitHub App**.
3. Konfiguriere die App:

| Einstellung         | Wert                             |
| ------------------- | -------------------------------- |
| Name                | `TaskFlow Backstage Portal`      |
| Homepage URL        | `https://backstage.taskflow.dev` |
| Webhook             | Deaktiviert, vorerst             |
| Repository contents | Read                             |
| Pull requests       | Read & Write                     |
| Metadata            | Read                             |
| Workflows           | Read                             |

4. Generiere den Private Key und lade ihn herunter.
5. Installiere die App in deiner Organisation.

### `app-config.yaml` mit GitHub App aktualisieren

```yaml
integrations:
  github:
    - host: github.com
      apps:
        - appId: ${GITHUB_APP_ID}
          clientId: ${GITHUB_APP_CLIENT_ID}
          clientSecret: ${GITHUB_APP_CLIENT_SECRET}
          webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}
          privateKey: ${GITHUB_APP_PRIVATE_KEY}
```

Damit kann Backstage auf Repositories zugreifen, Pull Requests erstellen und Workflows im Namen der App auslösen.

---

## GitHub-Annotationen im Catalog

Components verwenden Annotationen, um eine Verbindung zu GitHub herzustellen.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    github.com/project-slug: tekanaid/taskflow-backend
    github.com/workflows: ci.yml,deploy.yml
spec:
  type: service
  owner: platform-team
```

Mit diesen Annotationen zeigt Backstage auf jeder Component-Seite relevante GitHub-Informationen an.

| Backstage-Anzeige                       | Quelle                  |
| --------------------------------------- | ----------------------- |
| Repository README                       | GitHub Repository Root  |
| Zeitachse der letzten Commits           | GitHub Commits API      |
| Offene und gemergte Pull Requests       | GitHub PR API           |
| GitHub Actions Workflow Runs und Status | GitHub Actions API      |
| Contributors und Code Owners            | GitHub Contributors API |

Entwickler können bei Bedarf direkt zu GitHub wechseln. Die Übersicht erhalten sie jedoch direkt in Backstage.

---

## Backstage mit Kubernetes integrieren

Das Kubernetes Plugin zeigt Cluster-Informationen direkt in Backstage an.

### Plugin installieren

```bash
cd packages/app
yarn add @backstage/plugin-kubernetes
```

### Konfiguration in `app-config.yaml`

```yaml
kubernetes:
  serviceLocatorMethod:
    type: 'multiTenant'
  clusterLocatorMethods:
    - type: 'config'
      clusters:
        - url: https://kubernetes.default.svc
          name: production-cluster
          authProvider: 'serviceAccount'
          skipTLSVerify: false
          serviceAccountToken: ${K8S_SERVICE_ACCOUNT_TOKEN}
```

### Component Page erweitern

```tsx
// packages/app/src/components/catalog/EntityPage.tsx
import { EntityKubernetesContent } from '@backstage/plugin-kubernetes';

const serviceEntityPage = (
  <EntityLayout>
    <EntityLayout.Route path="/" title="Overview">
      <Grid container spacing={3}>
        <Grid item md={6}>
          <EntityAboutCard />
        </Grid>
      </Grid>
    </EntityLayout.Route>

    <EntityLayout.Route path="/kubernetes" title="Kubernetes">
      <EntityKubernetesContent />
    </EntityLayout.Route>
  </EntityLayout>
);
```

### Components mit Kubernetes-Labels annotieren

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    backstage.io/kubernetes-id: taskflow-backend
    backstage.io/kubernetes-namespace: production
```

Die Component-Seite des TaskFlow Backends zeigt danach:

* Deployment-Status mit Replicas und Ready Pods
* Pod-Liste mit Status und Logs
* Service Endpoints
* Ingress Routes
* Ressourcennutzung für CPU und Memory

Entwickler können Deployment-Probleme analysieren, ohne zu `kubectl` oder zur AWS Console zu wechseln.

---

## Monitoring-Tools anbinden

Backstage kann mit Grafana, Prometheus, Datadog oder anderen Monitoring-Plattformen verbunden werden.

---

## Grafana Plugin

### Plugin installieren

```bash
yarn add @k-phoen/backstage-plugin-grafana
```

### Konfiguration in `app-config.yaml`

```yaml
grafana:
  domain: https://grafana.taskflow.dev
  unifiedAlerting: true
```

### Dashboard-Links zu Components hinzufügen

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    grafana/dashboard-selector: "taskflow-backend"
    grafana/alert-label-selector: "service=taskflow-backend"
```

Dadurch werden Grafana-Dashboards direkt in Component-Seiten eingebettet.

Entwickler sehen:

* Request Rate, Latenz und Error Rate
* Trends zur Ressourcennutzung
* Aktive Alerts
* Links zu vollständigen Grafana-Dashboards

---

## Prometheus-Metriken

Für die Prometheus-Integration werden Annotationen verwendet.

```yaml
annotations:
  prometheus.io/rule: 'sum(rate(http_requests_total{service="taskflow-backend"}[5m]))'
```

Damit werden Prometheus-Metriken direkt in Backstage angezeigt. Ein zusätzlicher Wechsel zu Grafana ist nicht nötig.

---

## TechDocs: Documentation as Code

TechDocs wandelt Markdown-Dokumentation aus Repositories in strukturierte, durchsuchbare Dokumentation in Backstage um.

### TechDocs in `app-config.yaml` aktivieren

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```

### Produktionskonfiguration mit Cloud Storage

```yaml
techdocs:
  builder: 'external'
  publisher:
    type: 'awsS3'
    awsS3:
      bucketName: 'taskflow-techdocs'
      region: 'us-east-1'
      credentials:
        accessKeyId: ${AWS_ACCESS_KEY_ID}
        secretAccessKey: ${AWS_SECRET_ACCESS_KEY}
```

---

## Dokumentation zu Components hinzufügen

Erstelle ein `docs/`-Verzeichnis im Repository.

```text
taskflow-backend/
├── src/
├── docs/
│   ├── index.md
│   ├── api-reference.md
│   ├── deployment.md
│   └── troubleshooting.md
└── mkdocs.yml
```

### `mkdocs.yml` konfigurieren

```yaml
site_name: 'TaskFlow Backend Documentation'
site_description: 'FastAPI backend for TaskFlow application'

nav:
  - Home: index.md
  - API Reference: api-reference.md
  - Deployment: deployment.md
  - Troubleshooting: troubleshooting.md

plugins:
  - techdocs-core
```

### Component für TechDocs annotieren

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    backstage.io/techdocs-ref: dir:.
```

`dir:.` bedeutet, dass die Dokumentation im selben Repository liegt. Alternativ kann `url:` für externe Repositories verwendet werden.

---

## Dokumentation schreiben

### `docs/index.md`

# TaskFlow Backend-Dokumentation

Willkommen in der Dokumentation des TaskFlow Backends. Dieser FastAPI-Service stellt die REST API für das Task Management bereit.

## Überblick

Das Backend übernimmt:

* CRUD-Operationen für Tasks
* Benutzerauthentifizierung
* Persistenz mit PostgreSQL
* Caching mit Redis

## Quick Start

Lokal starten:

```bash
docker-compose up
uvicorn src.main:app --reload
```

API-Dokumentation öffnen:

```text
http://localhost:8000/docs
```

## Architektur

Das Backend verwendet FastAPI mit einer PostgreSQL-Datenbank und Redis Cache.

> **Diagramm:** Das Architekturdiagramm wird hier eingefügt.

## API Endpoints

Weitere Details zu den Endpoints stehen in der API Reference.

TechDocs rendert diese Inhalte mit Navigation, Suche und Backstage Theme. Entwickler lesen die Dokumentation direkt in Backstage.

---

## Authentifizierung und Autorisierung

Konfiguriere die Authentifizierung für den Zugriff auf das Backstage Portal.

### GitHub OAuth

GitHub OAuth wird für diesen Anwendungsfall empfohlen.

```yaml
auth:
  environment: production
  providers:
    github:
      production:
        clientId: ${GITHUB_OAUTH_CLIENT_ID}
        clientSecret: ${GITHUB_OAUTH_CLIENT_SECRET}
```

### GitHub OAuth App erstellen

1. Öffne GitHub Settings → Developer Settings → OAuth Apps → New OAuth App.
2. Setze den Application Name auf `TaskFlow Backstage`.
3. Setze die Homepage URL auf `https://backstage.taskflow.dev`.
4. Setze die Authorization Callback URL auf `https://backstage.taskflow.dev/api/auth/github/handler/frame`.
5. Kopiere die Client ID und generiere ein Client Secret.
6. Setze die benötigten Umgebungsvariablen.

### Sign-in in `App.tsx` aktivieren

```tsx
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPage } from '@backstage/core-components';

const app = createApp({
  components: {
    SignInPage: props => (
      <SignInPage
        {...props}
        auto
        provider={{
          id: 'github-auth-provider',
          title: 'GitHub',
          message: 'Sign in using GitHub',
          apiRef: githubAuthApiRef,
        }}
      />
    ),
  },
});
```

Benutzer authentifizieren sich danach mit GitHub, bevor sie auf Backstage zugreifen.

---

## Autorisierung und Permissions

Implementiere rollenbasierte Zugriffskontrolle mit RBAC.

### Permissions in `app-config.yaml` definieren

```yaml
permission:
  enabled: true
  policy:
    - resource: 'catalog-entity'
      actions: ['read']
      allow:
        - kind: 'Group'
          name: 'everyone'
    - resource: 'catalog-entity'
      actions: ['create', 'update', 'delete']
      allow:
        - kind: 'Group'
          name: 'platform-team'
    - resource: 'scaffolder-template'
      actions: ['use']
      allow:
        - kind: 'Group'
          name: 'developers'
```

Diese Konfiguration gewährt:

* Alle Benutzer können Catalog Entities lesen.
* Nur das `platform-team` kann Entities erstellen, aktualisieren oder löschen.
* Entwickler können Software Templates verwenden.

### Custom Permission Policies in TypeScript

Für komplexere Regeln können eigene Permission Policies in TypeScript implementiert werden.

```ts
import { PermissionPolicy } from '@backstage/plugin-permission-node';

export class CustomPermissionPolicy implements PermissionPolicy {
  async handle(request, user) {
    if (request.permission.name === 'catalog.entity.delete') {
      // Only allow deletion for entities owned by user's team
      const entity = request.resourceRef;
      const ownerTeam = entity.metadata.annotations['owner'];
      return user.groups.includes(ownerTeam);
    }
    return { result: AuthorizeResult.ALLOW };
  }
}
```

---

## Vollständiges Developer Portal für TaskFlow aufbauen

> **Abbildung:** Backstage-Plattformintegrationen. Backstage dient als zentraler Hub und verbindet GitHub, Kubernetes, Grafana, TechDocs, Catalog, Templates, Authentifizierung und RBAC.

Kombiniere alle Integrationen zu einem vollständigen Portal.

| Component             | Nutzen                                                                            |
| --------------------- | --------------------------------------------------------------------------------- |
| Catalog               | Alle TaskFlow Components sind registriert: Frontend, Backend, Datenbank und Redis |
| TechDocs              | Dokumentation je Component: API Reference, Deployment Guides und Troubleshooting  |
| GitHub Integration    | Commits, Pull Requests und Workflow Runs je Component anzeigen                    |
| Kubernetes Plugin     | Pod-Status, Logs und Ressourcennutzung anzeigen                                   |
| Monitoring Dashboards | Grafana-Dashboards für Performance-Metriken einbetten                             |
| Custom Plugins        | TaskFlow Dashboard mit anwendungsspezifischen Metriken                            |
| Software Templates    | Template `New FastAPI Microservice` für Self-Service-Projekterstellung            |
| Authentication        | GitHub OAuth für sicheren Portalzugriff                                           |
| Permissions           | RBAC zur Steuerung von Catalog-Änderungen und Template-Nutzung                    |

---

## Developer Workflow

1. Mit GitHub anmelden.
2. Im Catalog das TaskFlow Backend finden.
3. TechDocs für die API-Dokumentation lesen.
4. Im Kubernetes-Tab den Deployment-Status prüfen.
5. Das Grafana-Dashboard für Performance-Metriken öffnen.
6. Im GitHub-Tab aktuelle Commits und Pull Requests anzeigen.
7. Einen neuen Microservice über ein Template erstellen.
8. Das TaskFlow Dashboard für den Anwendungszustand überwachen.

Alles, was Entwickler benötigen, ist an einem Ort verfügbar. Navigation und Suche bleiben konsistent.

---

## Best Practices für Developer Experience

| Bereich         | Best Practice                                                                                                                                                         |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Portal Adoption | Backstage in das Onboarding neuer Entwickler aufnehmen, Workshops durchführen, interne Workflow-Guides erstellen und Team Champions benennen                          |
| Content Quality | Catalog-Registrierung über CI/CD automatisieren, TechDocs wie Code behandeln, Alerts für Integrationsfehler einrichten und einen Deprecation-Prozess pflegen          |
| Performance     | Backend-Caching für Catalog Queries nutzen, Plugin-Code lazy laden und langsame Seiten messen und optimieren                                                          |
| Security        | Least-Privilege-Berechtigungen vergeben, Zugriff auf sensible Daten protokollieren, Tokens regelmäßig rotieren und Backstage-Abhängigkeiten auf Schwachstellen prüfen |

---

## Einfluss auf die Developer Experience messen

Verfolge Metriken, um den Nutzen des Portals messbar zu machen.

| Kategorie              | Zu messende Metriken                                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Developer Productivity | Time to First Commit für neue Projekte, Onboarding-Zeit, Kontextwechsel pro Tag, Self-Service Completion Rate                         |
| Portal Adoption        | Daily Active Users, Page Views pro Component, Template-Nutzungsrate, Suchanfragen und Click-through Rate                              |
| Developer Satisfaction | NPS und Zufriedenheitswerte aus Umfragen, Support-Ticket-Volumen, Auffindbarkeit der Dokumentation, Feedback aus Entwicklerinterviews |

### Beispielergebnisse nach dem Rollout des TaskFlow Portals

| Metrik                                              |   Vorher |    Nachher |
| --------------------------------------------------- | -------: | ---------: |
| Zeit zum Erstellen eines neuen Microservice         | 2–3 Tage | 15 Minuten |
| Entwickler, die Dokumentation über Backstage finden |     30 % |       80 % |
| Kontextwechsel pro Tag                              |      20+ |         ~5 |
| Developer NPS Score                                 |       40 |         75 |

Nutze diese Metriken, um weitere Investitionen in das Portal zu begründen und Verbesserungsbereiche zu identifizieren.

---

## Beispiele für Self-Service-Automatisierung

### Automatisierte Bereitstellung von Umgebungen

* Template erstellt Dev- und Staging-Umgebungen in AWS.
* RDS-Datenbank und ElastiCache Redis werden provisioniert.
* VPC und Security Groups werden eingerichtet.
* Deployment nach EKS erfolgt mit Terraform.

### Onboarding-Automatisierung

* Template erstellt einen Slack Channel.
* Entwickler wird zu GitHub Teams hinzugefügt.
* Jira-Projekt wird erstellt.
* Willkommens-E-Mail mit relevanten Links wird versendet.

### Compliance Checks

* Automatisierte Security Scans für neue Projekte
* Prüfung der Lizenzkonformität
* Checkliste für GDPR- und Privacy-Anforderungen
* Approval Workflows für Produktionszugriff

### Dokumentationsgenerierung

* API-Dokumentation automatisch aus OpenAPI Specs generieren
* Architekturdiagramme aus Code erstellen
* Changelog aus Commits generieren
* TechDocs bei jedem Merge nach `main` aktualisieren

Diese Automatisierungen reduzieren manuelle Arbeit, erzwingen Standards und geben Entwicklern mehr Zeit für Feature-Entwicklung.

---

## Ausblick

Glückwunsch. Du hast Woche 12 abgeschlossen: Backstage – Internal Developer Portal.

Du hast gelernt:

* Grundlagen interner Developer Portals und Developer Experience
* React Framework im Vergleich zu Vue.js
* Backstage Service Catalog und Component-Registrierung
* React Hooks und Component-Entwicklung
* Backstage Plugins bauen
* TaskFlow Dashboard Plugin mit Echtzeitmetriken erstellen
* Software Templates und Scaffolder für Projekt-Automatisierung
* Plattformintegrationen mit GitHub, Kubernetes, Monitoring und TechDocs
* Authentifizierung, Autorisierung und Security
* Einfluss auf die Developer Experience messen

Du hast ein vollständiges internes Developer Portal für TaskFlow aufgebaut mit:

* Service Catalog für alle Components
* Custom Dashboard Plugin für Metriken
* Software Template zur Erstellung neuer Microservices
* GitHub-, Kubernetes- und Monitoring-Integrationen
* Documentation as Code mit TechDocs
* Self-Service Workflows für Entwickler

Diese Platform-Engineering-Fähigkeit verbessert die Produktivität deutlich. Sie verkürzt die Onboarding-Zeit und schafft eine bessere Developer Experience für die Organisation.

In den kommenden Wochen geht es im Platform Engineering Bootcamp weiter mit Observability über Prometheus und Grafana, Service Mesh mit Istio, Secrets Management mit Vault und GitOps mit ArgoCD.

Jede Woche baut auf dieser Grundlage auf und erweitert dein Platform-Engineering-Skillset.

Du hast jetzt die Fähigkeiten, interne Developer Portals zu bauen, die Developer Experience und Plattformfähigkeiten in deiner Organisation skalieren.

---

## Kernaussagen

* Die GitHub-Integration zeigt Repository-Informationen, Commits, Pull Requests und Workflows auf Catalog-Seiten über die Annotation `github.com/project-slug`.
* Das Kubernetes Plugin zeigt Pod-Status, Logs und Ressourcennutzung über die Annotation `backstage.io/kubernetes-id` und die Cluster-Konfiguration.
* TechDocs rendert Markdown-Dokumentation aus Repositories mithilfe von `mkdocs.yml` und der Annotation `backstage.io/techdocs-ref`.
* GitHub OAuth stellt die Authentifizierung über `auth.providers.github` bereit. RBAC steuert Berechtigungen für Catalog und Templates.
* Der Einfluss auf die Developer Experience wird über Metriken gemessen: Time to First Commit, Onboarding-Zeit, Portal Adoption und Zufriedenheitswerte.

---

## Zusätzliche Ressourcen

| Ressource                              | Typ           |
| -------------------------------------- | ------------- |
| Backstage-Integrationen – Übersicht    | Dokumentation |
| Kubernetes Plugin Dokumentation        | Dokumentation |
| TechDocs Dokumentation                 | Dokumentation |
| Authentication and Authorization Guide | Dokumentation |

---

## Private Lesson Notes

Private Notizen während des Lernens erfassen.

---

## Study Group

### Backstage – Internes Developer Portal

Baue ein internes Developer Portal mit Backstage, um die Developer Experience zu verbessern. Lerne das React Framework durch die Entwicklung von Backstage Plugins. Die Teilnehmer kennen bereits Vue.js.

Erstelle Service Catalog, Dokumentation und Templates für TaskFlow.
# Interne Developer Portals und Backstage-Überblick

Entdecke, wie interne Developer Portals die Developer Experience und Platform-Engineering-Workflows verbessern. Lerne die Backstage-Architektur kennen, ihre Ursprünge bei Spotify und wie Backstage Self-Service-Plattformen für das TaskFlow-Anwendungsökosystem ermöglicht.

---

## Lernmaterial

Willkommen in Woche 12 des Platform Engineering Bootcamps.

In den vergangenen 11 Wochen hast du zentrale DevOps-Grundlagen erarbeitet: Linux, Git, Networking, Docker, Kubernetes, AWS, Terraform und Ansible.

Du hast außerdem die TaskFlow-Anwendung aufgebaut:

* Vue.js Frontend
* FastAPI Backend
* PostgreSQL Datenbank
* Redis Cache
* Betrieb auf Kubernetes
* Infrastruktur als Code

Diese Woche geht es um einen wichtigen Baustein moderner Plattformteams: das interne Developer Portal.

Wir verwenden dafür Backstage. Backstage ist eine Open-Source-Plattform, die von Spotify entwickelt wurde und sich als Branchenstandard für den Aufbau von Developer Portals etabliert hat.

Gleichzeitig lernst du React kennen. Dabei vergleichen wir React mit Vue.js, das du bereits kennst.

---

## Was ist ein internes Developer Portal?

Ein internes Developer Portal ist eine zentrale Plattform für Entwickler. Es stellt Self-Service-Werkzeuge, Dokumentation und Automatisierung bereit, um Produktivität und Developer Experience zu verbessern.

Du kannst es dir als zentrale Oberfläche vorstellen, über die Entwickler:

* alle Services, APIs und Ressourcen der Organisation entdecken,
* Dokumentation für jede Komponente abrufen,
* neue Projekte mit wenigen Klicks aus Templates erstellen,
* Deployment-Status und Application Health prüfen,
* Ownership-Informationen und zuständige Teams finden,
* über Infrastruktur und Services hinweg suchen.

Ohne Developer Portal verlieren Entwickler Zeit bei der Suche nach Informationen. Sie wechseln zwischen Confluence-Seiten, GitHub-Repositories, Slack Channels und informellem Teamwissen.

Mit einem Portal sind Informationen auffindbar, dokumentiert und zentral zugänglich.

---

## Warum Developer Experience wichtig ist

Developer Experience, kurz DevEx, ist ein zentraler Fokus moderner Platform-Engineering-Teams.

Wenn Entwickler weniger Zeit mit wiederkehrender manueller Arbeit verbringen, haben sie mehr Zeit für wertschöpfende Features. Dazu gehören Aufgaben wie:

* Dokumentation suchen
* Deployment-Prozesse verstehen
* Entwicklungsumgebungen einrichten
* Zuständigkeiten klären
* Standards manuell nachvollziehen

Eine schlechte Developer Experience führt häufig zu:

* langsamerer Time-to-Market für neue Features,
* höherer kognitiver Belastung,
* Developer Burnout,
* uneinheitlichen Praktiken zwischen Teams,
* doppelter Arbeit,
* wiederholtem Lösen derselben Probleme.

Ein gutes internes Developer Portal reduziert diese Probleme. Es macht den „Golden Path“ zum einfachsten Weg.

Entwickler können viele Aufgaben selbst erledigen, ohne Tickets zu erstellen, Freigaben abzuwarten oder Plattformteams direkt einzubinden.

---

## Backstage: Plattform für Developer Portals

Backstage ist ein Open-Source-Framework. Es wurde 2020 von Spotify entwickelt und an die Cloud Native Computing Foundation, kurz CNCF, übergeben.

Backstage wird heute von vielen Unternehmen eingesetzt, darunter Netflix, American Airlines, Expedia und IKEA.

Backstage bietet vier zentrale Funktionen.

| Funktion                        | Beschreibung                                                                                                                                                                                                                                    |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Software Catalog                | Zentrales Register aller Software-Systeme. Dazu gehören Services, Libraries, Websites, Datenbanken und ML-Modelle. Ownership, Dokumentation und Abhängigkeiten werden über YAML-Dateien gepflegt.                                               |
| Software Templates (Scaffolder) | Vorgefertigte Templates zur Automatisierung der Projekterstellung. Entwickler füllen ein Formular aus. Backstage erzeugt ein Repository mit Code, CI/CD-Pipelines, Dokumentation und Catalog-Registrierung nach den Standards der Organisation. |
| TechDocs                        | Documentation as Code. Dokumentation wird als Markdown direkt neben dem Code gepflegt. Backstage rendert sie im Portal mit Suche und Navigation.                                                                                                |
| Plugin Ecosystem                | Backstage ist über Plugins erweiterbar. GitHub, Kubernetes, Jenkins, PagerDuty, Datadog und viele weitere Tools können integriert werden. Eigene Plugins werden mit React gebaut und können beliebige Daten für Entwickler sichtbar machen.     |

---

## Backstage-Architektur

> **Abbildung:** Backstage-Drei-Schichten-Architektur mit Frontend, Backend, Datenbank und Plugin Ecosystem.

Backstage basiert auf drei Hauptschichten.

### Frontend

Das Frontend ist eine React-Anwendung. Für die Benutzeroberfläche wird Material-UI verwendet.

Plugins können neue Seiten, Komponenten und Funktionen im Frontend bereitstellen.

### Backend

Das Backend ist eine Node.js-Anwendung. Es stellt REST APIs für Catalog, Scaffolder, Search und Plugins bereit.

Backend-Plugins können externe Systeme und Datenbanken integrieren.

### Datenbank

PostgreSQL speichert Catalog Entities, Benutzereinstellungen und Plugin-Daten.

Der Catalog ist die zentrale Quelle für Softwareinformationen in der Organisation.

---

## Plugin-Architektur

Backstage verwendet eine Plugin-Architektur. Jede größere Funktion ist als Plugin umgesetzt.

Das Backstage-Core-Team pflegt offizielle Plugins wie:

* Catalog
* Scaffolder
* TechDocs

Die Community stellt zusätzlich viele weitere Plugins bereit.

In dieser Woche baust du ein eigenes Plugin, um TaskFlow-Metriken direkt in Backstage sichtbar zu machen.

---

## Wie Backstage TaskFlow-Workflows verbessert

Für die TaskFlow-Anwendung stellt Backstage mehrere zentrale Fähigkeiten bereit.

| Fähigkeit              | Nutzen                                                                                                                                                                      |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Service Discovery      | Alle TaskFlow-Komponenten werden im Catalog registriert: Frontend, Backend, Datenbank und Redis. Jede Komponente enthält Ownership, Repository-Links und Deployment-Status. |
| Documentation Hub      | TechDocs stellt API-Dokumentation, Deployment Guides, Troubleshooting Runbooks und Architekturdiagramme bereit. Alles ist über den Catalog auffindbar.                      |
| Project Templates      | Ein Template `New FastAPI Microservice` erzeugt einen neuen Service nach TaskFlow-Standards: FastAPI, Docker, Kubernetes, GitHub Actions und Monitoring.                    |
| Custom Dashboard       | Ein TaskFlow Dashboard Plugin zeigt Echtzeitmetriken: Gesamtzahl der Tasks, aktive Benutzer, Deployment-Status und Zustand der Infrastruktur.                               |
| Kubernetes Integration | TaskFlow Pods, Deployments und Services können direkt in Backstage angezeigt werden. Ein Wechsel zu `kubectl` oder zur AWS Console ist nicht nötig.                         |

---

## Backstage installieren

Backstage lässt sich mit Node.js ab Version 18 und Yarn installieren.

Der Befehl `create-app` erzeugt eine neue Backstage-Anwendung.

```bash
npx @backstage/create-app
```

Während der Erstellung wirst du nach einem Namen für die Anwendung gefragt, zum Beispiel `taskflow-portal`.

Die CLI erstellt ein neues Verzeichnis mit folgender Struktur:

| Datei oder Verzeichnis | Zweck                                                          |
| ---------------------- | -------------------------------------------------------------- |
| `app-config.yaml`      | Konfiguration für Integrationen, Plugins und Authentifizierung |
| `packages/app/`        | Frontend-Anwendung mit React                                   |
| `packages/backend/`    | Backend-Anwendung mit Node.js                                  |
| `catalog-info.yaml`    | Beispielhafte Catalog Entities                                 |

### Backstage lokal starten

```bash
cd taskflow-portal
yarn install
yarn dev
```

Damit werden Frontend und Backend im Entwicklungsmodus gestartet.

| Dienst   |   Port |
| -------- | -----: |
| Frontend | `3000` |
| Backend  | `7007` |

Danach ist die Backstage-Oberfläche mit Standard-Catalog, Startseite und Dokumentation verfügbar.

---

## Navigation in der Backstage-Oberfläche

Die Standardoberfläche von Backstage enthält mehrere wichtige Bereiche.

### Home

Die Startseite ist ein anpassbares Dashboard. Sie zeigt Favoriten, letzte Suchanfragen und personalisierte Inhalte.

### Catalog

Der Catalog ist die zentrale Ansicht für Software Entities.

Du kannst nach folgenden Kriterien filtern:

* Kind, zum Beispiel `Component`, `API` oder `Resource`
* Tags
* Ownership
* Lifecycle Stage

### APIs

Der API-Bereich ist für die Suche und Anzeige von APIs vorgesehen.

OpenAPI- und AsyncAPI-Definitionen werden als interaktive Dokumentation gerendert.

### Docs

Docs zeigt TechDocs für alle Services.

Du kannst eine Component im Catalog öffnen und über den Docs-Tab die gerenderte Markdown-Dokumentation anzeigen.

### Create

Create ist der Bereich für Software Templates und den Scaffolder.

Entwickler wählen ein Template aus, füllen ein Formular aus und generieren daraus ein neues Projekt.

### Search

Search ist die globale Suche über Catalog, Dokumentation und Plugins.

Hier finden Entwickler schnell die Informationen, die sie benötigen.

---

## Erweiterbare Navigation

Die Sidebar ist erweiterbar.

Plugins können eigene Navigationseinträge hinzufügen.

Beim Bau des eigenen Plugins fügst du später einen Link für das `TaskFlow Dashboard` hinzu.

---

## Grundlagen der Konfiguration

Die Backstage-Konfiguration liegt in `app-config.yaml`.

Diese Datei steuert Integrationen, Authentifizierung und Plugin-Einstellungen.

Wichtige Abschnitte sind:

```yaml
app:
  title: TaskFlow Developer Portal
  baseUrl: http://localhost:3000

backend:
  baseUrl: http://localhost:7007
  database:
    client: pg
    connection:
      host: localhost
      port: 5432
      user: backstage
      password: backstage

catalog:
  rules:
    - allow: [Component, API, Resource, System, Domain, Location]

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

Umgebungsvariablen wie `${GITHUB_TOKEN}` werden zur Laufzeit ersetzt.

Später konfigurierst du die GitHub-Integration, um TaskFlow-Repositories in den Catalog zu importieren.

---

## Ausblick

In dieser Lektion hast du interne Developer Portals, die Bedeutung von Developer Experience und Backstage als Plattform für Developer Portals kennengelernt.

Du hast außerdem die Backstage-Architektur verstanden:

* Catalog
* Scaffolder
* TechDocs
* Plugins

Außerdem hast du gesehen, wie Backstage Workflows für die TaskFlow-Anwendung verbessert.

Als Nächstes geht es um React-Grundlagen und den Vergleich mit Vue.js.

Da Backstage-Plugins mit React gebaut werden, brauchst du ein gutes Verständnis für:

* React Component Model
* State Management mit Hooks
* JSX-Syntax

Wir ziehen direkte Vergleiche zur Composition API von Vue.js. So kannst du vorhandenes Wissen gezielt übertragen.

Danach richtest du den Backstage Service Catalog für TaskFlow ein. Dabei registrierst du alle Components mit Metadaten, Ownership und Dokumentationslinks.

Am Ende dieser Woche hast du ein funktionsfähiges internes Developer Portal mit eigenen Plugins und Self-Service-Automatisierung für die TaskFlow-Plattform.

---

## Kernaussagen

* Interne Developer Portals stellen Self-Service-Werkzeuge und zentrale Dokumentation bereit, um die Developer Experience zu verbessern.
* Backstage ist ein Open-Source-Projekt der CNCF und wurde von Spotify entwickelt.
* Backstage bietet vier Kernfunktionen: Software Catalog, Scaffolder, TechDocs und Plugins.
* Die Backstage-Architektur besteht aus React Frontend, Node.js Backend und PostgreSQL Datenbank.
* Das Plugin-System macht Backstage flexibel erweiterbar.
* Für TaskFlow ermöglicht Backstage Service Discovery, Documentation Hub, Project Templates, Custom Dashboards und Kubernetes Integration.
* Die Installation von Backstage erfordert Node.js 18 oder höher.
* Neue Anwendungen werden mit `npx @backstage/create-app` erstellt.

---

## Zusätzliche Ressourcen

| Ressource                                     | Typ           |
| --------------------------------------------- | ------------- |
| Offizielle Backstage-Dokumentation            | Dokumentation |
| Backstage GitHub Repository                   | GitHub        |
| CNCF Backstage-Projekt                        | Dokumentation |
| Spotify Engineering Blog zum Backstage Launch | Artikel       |

---

## Private Lesson Notes

Private Notizen während des Lernens erfassen.

---

# React vs. Vue.js – Framework-Vergleich

Lerne React, indem du es mit Vue.js vergleichst, das du bereits kennst. Du verstehst die Unterschiede in der Template-Syntax, im State Management, bei Lifecycle Hooks und beim Einsatz beider Frameworks im Platform Engineering.

---

## Lernmaterial

In Woche 6 hast du das TaskFlow Frontend mit Vue.js und der Composition API gebaut. Du hast reaktiven State mit `ref()`, Lifecycle Hooks wie `onMounted()` und Template-Syntax mit Direktiven wie `v-if` und `v-for` kennengelernt.

Dieses Wissen ist eine starke Grundlage, um React zu lernen.

React und Vue.js lösen dieselben Probleme: Beide Frameworks helfen dabei, interaktive Benutzeroberflächen mit wiederverwendbaren Komponenten zu bauen. Der Unterschied liegt im Ansatz.

Wenn du diese Unterschiede verstehst, kannst du für jedes Projekt das passende Werkzeug wählen. Gleichzeitig bereitest du dich darauf vor, diese Woche Backstage Plugins zu entwickeln.

---

## Warum React nach Vue.js lernen?

Es gibt mehrere gute Gründe, React zusätzlich zu Vue.js zu lernen.

### Hohe Nachfrage in der Industrie

React ist das am weitesten verbreitete Frontend-Framework. Der Arbeitsmarkt ist größer als bei Vue.js. Viele Enterprise-Unternehmen und Open-Source-Projekte setzen auf React, weil das Framework ausgereift ist und ein großes Ökosystem bietet.

### Backstage-Ökosystem

Backstage basiert auf React. Wenn du eigene Plugins für dein internes Developer Portal bauen willst, musst du React Components, Hooks und JSX verstehen.

### Ergänzende Fähigkeiten

Vue.js und React sind keine Gegensätze. Sie ergänzen sich.

Einige Projekte profitieren von der Einfachheit und sanften Lernkurve von Vue.js. Andere Projekte benötigen das React-Ökosystem und die Unterstützung durch Meta.

### Besseres Verständnis von Frontend-Architektur

Wenn du beide Frameworks lernst, verstehst du Frontend-Konzepte tiefer. Du erkennst Muster, die unabhängig von einem bestimmten Framework funktionieren.

Für TaskFlow verwenden wir Vue.js für das Haupt-Frontend. Es ist einsteigerfreundlich und produktiv. Für Backstage Plugins verwenden wir React, weil Backstage es voraussetzt.

Beide Frameworks sind wertvolle Werkzeuge in deinem Platform-Engineering-Werkzeugkasten.

---

## Template-Syntax: Vue Templates vs. JSX

Vue.js verwendet eine HTML-basierte Template-Syntax mit speziellen Direktiven.

```vue
<template>
  <div v-if="isLoading">Loading...</div>
  <ul v-else>
    <li v-for="task in tasks" :key="task.id">
      {{ task.title }}
    </li>
  </ul>
</template>
```

React verwendet JSX. Dabei schreibst du HTML-ähnliche Syntax direkt in JavaScript.

```jsx
function TaskList({ tasks, isLoading }) {
  if (isLoading) {
    return <div>Loading...</div>;
  }

  return (
    <ul>
      {tasks.map(task => (
        <li key={task.id}>{task.title}</li>
      ))}
    </ul>
  );
}
```

### Zentrale Unterschiede

| Thema                | Vue.js                                                                               | React                                               |
| -------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------- |
| Conditions und Loops | Direktiven: `v-if`, `v-for`, `v-bind`                                                | JavaScript-Ausdrücke: `{...}`, `.map()`             |
| Dateistruktur        | Single File Components mit getrennten Bereichen für `template`, `script` und `style` | JSX kombiniert JavaScript und Markup in einer Datei |
| Interpolation        | Doppelte geschweifte Klammern: `{{ task.title }}`                                    | Einfache geschweifte Klammern: `{task.title}`       |
| List Keys            | `v-for` mit `v-bind:key`                                                             | `.map()` mit `key` Prop                             |

Beide Ansätze sind gültig.

Vue Templates fühlen sich für Entwickler vertraut an, die aus HTML kommen. JSX fühlt sich natürlicher an, wenn du dich in JavaScript mit Funktionen und Ausdrücken wohlfühlst.

---

## State Management: `ref()` vs. `useState`

In der Vue.js Composition API erzeugst du reaktiven State mit `ref()`.

```js
import { ref } from 'vue';

const count = ref(0);
const increment = () => {
  count.value++;
};
```

In React verwendest du dafür den `useState` Hook.

```js
import { useState } from 'react';

const [count, setCount] = useState(0);
const increment = () => {
  setCount(count + 1);
};
```

### Zentrale Unterschiede

| Thema               | Vue.js `ref`                      | React `useState`                                             |
| ------------------- | --------------------------------- | ------------------------------------------------------------ |
| Rückgabewert        | Objekt mit `.value` Property      | Array `[value, setter]`, das per Destructuring entpackt wird |
| State aktualisieren | Direkte Mutation: `count.value++` | Setter aufrufen: `setCount(count + 1)`                       |
| Reaktivität         | Automatisch                       | Erfordert den Aufruf der Setter-Funktion                     |
| Immutability        | Mutationen sind erlaubt           | State darf nie direkt mutiert werden                         |

Das React-Muster kann sich zunächst ungewohnt anfühlen, wenn du Vue.js gewohnt bist. Dafür ist es sehr explizit. Du erkennst State-Änderungen sofort an Aufrufen wie `setCount()`.

---

## Component Lifecycle: `onMounted` vs. `useEffect`

Vue.js stellt Lifecycle Hooks wie `onMounted()` für Side Effects bereit.

```js
import { ref, onMounted } from 'vue';

const tasks = ref([]);

onMounted(async () => {
  const response = await fetch('/api/tasks');
  tasks.value = await response.json();
});
```

React verwendet den `useEffect` Hook für alle Side Effects.

```js
import { useState, useEffect } from 'react';

const [tasks, setTasks] = useState([]);

useEffect(() => {
  async function fetchTasks() {
    const response = await fetch('/api/tasks');
    const data = await response.json();
    setTasks(data);
  }
  fetchTasks();
}, []); // Empty array means run once on mount
```

### Zentrale Unterschiede

| Thema                      | Vue.js                                         | React                                |
| -------------------------- | ---------------------------------------------- | ------------------------------------ |
| Beim Mount ausführen       | `onMounted(() => {...})`                       | `useEffect(() => {...}, [])`         |
| Wert beobachten            | `watch(count, ...)`                            | `useEffect(() => {...}, [count])`    |
| Bei jedem Render ausführen | Nicht direkt, dafür `watchEffect`              | `useEffect(() => {...})` ohne Array  |
| Cleanup beim Unmount       | `onUnmounted(() => {...})`                     | Funktion aus `useEffect` zurückgeben |
| Spezifität                 | Eigene Hooks für einzelne Lifecycle-Zeitpunkte | Ein Hook mit Dependency Arrays       |

`useEffect` ist sehr flexibel, hat aber eine steilere Lernkurve. Du musst Dependency Arrays verstehen, um Endlosschleifen und veraltete Closures zu vermeiden.

---

## Props und Events: Vue vs. React

Vue.js Components empfangen Props und senden Events.

```vue
<!-- Parent component -->
<TaskItem
  :task="task"
  @complete="handleComplete"
/>

<!-- Child component -->
<script setup>
import { defineProps, defineEmits } from 'vue';

const props = defineProps(['task']);
const emit = defineEmits(['complete']);

const markComplete = () => {
  emit('complete', props.task.id);
};
</script>
```

React Components empfangen Props. Dazu gehören auch Callback-Funktionen.

```jsx
// Parent component
<TaskItem
  task={task}
  onComplete={handleComplete}
/>

// Child component
function TaskItem({ task, onComplete }) {
  const markComplete = () => {
    onComplete(task.id);
  };

  return <button onClick={markComplete}>Complete</button>;
}
```

### Zentrale Unterschiede

| Thema                  | Vue.js                                   | React                                              |
| ---------------------- | ---------------------------------------- | -------------------------------------------------- |
| Daten empfangen        | `defineProps`                            | Funktionsparameter als Props-Objekt                |
| Daten nach oben senden | `defineEmits` + `emit('event', payload)` | Callback wird als Prop übergeben                   |
| Event-Listener-Syntax  | `@click`, `@complete`                    | `onClick`, `onComplete`                            |
| Konzeptuelles Modell   | Props und Events sind getrennt           | Alles, auch Callbacks, fließt als Props nach unten |

Der React-Ansatz ist konzeptionell einfach: Alles wird als Props weitergegeben.

Vue.js trennt Props und Events klarer. Das macht die Kommunikation zwischen Parent und Child oft semantisch deutlicher.

---

## Component-Definitionen

Vue.js Single File Components haben getrennte Bereiche.

```vue
<template>
  <div class="task-card">
    <h3>{{ task.title }}</h3>
  </div>
</template>

<script setup>
import { defineProps } from 'vue';
const props = defineProps(['task']);
</script>

<style scoped>
.task-card {
  border: 1px solid #ccc;
  padding: 1rem;
}
</style>
```

React Components sind JavaScript-Funktionen, die JSX zurückgeben.

```jsx
import './TaskCard.css';

function TaskCard({ task }) {
  return (
    <div className="task-card">
      <h3>{task.title}</h3>
    </div>
  );
}

export default TaskCard;
```

### Zentrale Unterschiede

| Thema               | Vue.js                                          | React                                                                    |
| ------------------- | ----------------------------------------------- | ------------------------------------------------------------------------ |
| Dateistruktur       | Eine Datei mit `template`, `script` und `style` | JavaScript-first; JSX ist syntaktischer Zucker für `React.createElement` |
| Scoped Styles       | Integriert mit `<style scoped>`                 | Benötigt CSS Modules oder CSS-in-JS                                      |
| CSS-Klassenattribut | `class`                                         | `className`, da `class` ein reserviertes JavaScript-Schlüsselwort ist    |

---

## Wann Vue.js und wann React verwenden?

Beide Frameworks sind sehr gute Optionen. Im Platform Engineering hängt die Wahl stark vom Projektkontext ab.

| Szenario                  | Vue.js wählen                                                      | React wählen                                          |
| ------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------- |
| Team Tools und Dashboards | Interne Tools, schnelle Prototypen, schnelle MVPs                  | Enterprise-Anwendungen mit großen Teams               |
| Lernkurve                 | Sanfter Einstieg, Template-Syntax wirkt vertraut                   | Steilerer Einstieg, JSX und Hooks brauchen etwas Zeit |
| Ökosystem                 | Integriertes State Management mit Pinia und Routing mit Vue Router | Sehr großes Third-Party-Ökosystem                     |
| Mobile                    | —                                                                  | React Native für plattformübergreifende mobile Apps   |
| Bestehende Codebases      | —                                                                  | Erforderlich für Backstage und andere React-Projekte  |
| Bundle Size               | Standardmäßig kleiner                                              | Größer, aber tree-shakeable                           |
| Corporate Backing         | Community-getrieben                                                | Meta mit langfristiger Unterstützung                  |

Für TaskFlow wurde Vue.js gewählt, weil es einsteigerfreundlich und produktiv für Platform-Engineering-Dashboards ist.

Für Backstage Plugins ist React erforderlich, weil Backstage auf React basiert.

In der Praxis wirst du beide Frameworks professionell einsetzen.

---

## Codebeispiel: Task List in beiden Frameworks

Im folgenden Beispiel wird dieselbe Task List einmal mit Vue.js und einmal mit React umgesetzt.

---

## Vue.js-Version

```vue
<template>
  <div>
    <h2>Tasks ({{ tasks.length }})</h2>
    <div v-if="loading">Loading tasks...</div>
    <ul v-else>
      <li v-for="task in tasks" :key="task.id">
        {{ task.title }}
        <button @click="completeTask(task.id)">Complete</button>
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const tasks = ref([]);
const loading = ref(true);

onMounted(async () => {
  const res = await fetch('/api/tasks');
  tasks.value = await res.json();
  loading.value = false;
});

const completeTask = async (id) => {
  await fetch(`/api/tasks/${id}/complete`, { method: 'POST' });
  tasks.value = tasks.value.filter(t => t.id !== id);
};
</script>
```

---

## React-Version

```jsx
import { useState, useEffect } from 'react';

function TaskList() {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function fetchTasks() {
      const res = await fetch('/api/tasks');
      const data = await res.json();
      setTasks(data);
      setLoading(false);
    }
    fetchTasks();
  }, []);

  const completeTask = async (id) => {
    await fetch(`/api/tasks/${id}/complete`, { method: 'POST' });
    setTasks(tasks.filter(t => t.id !== id));
  };

  return (
    <div>
      <h2>Tasks ({tasks.length})</h2>
      {loading ? (
        <div>Loading tasks...</div>
      ) : (
        <ul>
          {tasks.map(task => (
            <li key={task.id}>
              {task.title}
              <button onClick={() => completeTask(task.id)}>
                Complete
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

export default TaskList;
```

### Gemeinsamkeiten

Beide Varianten:

* laden Daten beim Mount der Component,
* verwalten `loading` und `tasks` als State,
* reagieren auf Button-Klicks,
* aktualisieren den State nach einer Aktion,
* zeigen bedingt einen Loading-Zustand an.

Die Syntax ist unterschiedlich. Die Logik ist jedoch fast identisch.

Wenn du Vue.js verstehst, kennst du bereits einen großen Teil der React-Konzepte.

---

## Ausblick

In dieser Lektion hast du React durch den Vergleich mit Vue.js kennengelernt.

Du hast die wichtigsten Unterschiede gesehen:

* Template-Syntax: JSX vs. Vue Templates
* State Management: `useState` vs. `ref`
* Lifecycle Hooks: `useEffect` vs. `onMounted`
* Component Patterns
* Props und Events

Beide Frameworks lösen dieselben Probleme. Sie verwenden jedoch unterschiedliche Syntax und unterschiedliche Denkmodelle.

Als Nächstes wendest du dieses React-Wissen an, um den Backstage Service Catalog für TaskFlow einzurichten.

Du erstellst `catalog-info.yaml`-Dateien für:

* Frontend
* Backend
* Datenbank
* Redis Components

Anschließend registrierst du diese Components in Backstage und untersuchst, wie der Catalog Metadaten, Ownership und Abhängigkeiten organisiert.

Danach steigst du tiefer in React Hooks ein und baust dein erstes Backstage Plugin. Dieses Plugin zeigt TaskFlow-Metriken in einem eigenen Dashboard an.

Am Ende dieses Abschnitts bist du mit Vue.js und React vertraut. Du kannst dann bewusst entscheiden, welches Framework zu welchem Platform-Engineering-Szenario passt.

---

## Kernaussagen

* React verwendet JSX, also HTML-ähnliche Syntax in JavaScript. Vue.js verwendet Template-Direktiven wie `v-if` und `v-for`.
* Der React Hook `useState` gibt `[value, setter]` zurück. Vue `ref()` gibt ein Objekt mit `.value` Property zurück.
* React `useEffect` behandelt Side Effects über Dependency Arrays. Vue.js bietet spezifische Hooks wie `onMounted`, `onUpdated` und `onUnmounted`.
* React behandelt Events als Callback Props. Vue.js verwendet `defineEmits` für Custom Events mit `@event`-Syntax.
* Vue.js eignet sich besonders für schnelle Entwicklung und interne Tools.
* React ist stark für Enterprise-Anwendungen und erforderlich für die Entwicklung von Backstage Plugins.

---

## Zusätzliche Ressourcen

| Ressource                            | Typ           |
| ------------------------------------ | ------------- |
| Offizielle React-Dokumentation       | Dokumentation |
| Vergleichsleitfaden Vue.js vs. React | Artikel       |
| React Hooks Reference                | Dokumentation |
| Thinking in React Guide              | Tutorial      |

---

## Private Lesson Notes

Private Notizen während des Lernens erfassen.

---

## Study Group

### Backstage – Internes Developer Portal






# Backstage Service Catalog einrichten

Beherrsche den Backstage Service Catalog, indem du die TaskFlow-Komponenten registrierst. Du lernst die Struktur von `catalog-info.yaml`, Component Types wie `service`, `library`, `website` und `resource`, Metadaten, Ownership, Abhängigkeiten sowie die Verknüpfung von Repositories, APIs und Dokumentation.

---

## Lernmaterial

Der Software Catalog ist das Herzstück von Backstage. Er ist ein zentrales Register für alle Software-Komponenten deiner Organisation.

Dazu gehören:

* Services
* Libraries
* Websites
* Datenbanken
* APIs
* weitere technische Ressourcen

Jede Komponente wird über eine `catalog-info.yaml` beschrieben. Diese Datei liegt direkt im Repository der jeweiligen Komponente.

Für TaskFlow registrieren wir vier Hauptkomponenten:

* Vue.js Frontend
* FastAPI Backend
* PostgreSQL Datenbank
* Redis Cache

Am Ende dieser Lektion verstehst du die Catalog-Struktur, Component Types, Metadaten, Ownership und die Art, wie du deine gesamte Anwendung über Backstage auffindbar machst.

---

## Service Catalog verstehen

Der Backstage Catalog löst ein zentrales Problem moderner Organisationen:

> Welche Systeme existieren, und wer ist dafür verantwortlich?

Wenn Systeme auf Hunderte oder Tausende Services wachsen, wird Nachverfolgung ohne Catalog kaum beherrschbar.

Der Catalog bietet folgende Funktionen.

| Funktion      | Beschreibung                                                                                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discovery     | Alle Software-Komponenten der Organisation können an einem Ort gesucht und durchsucht werden. Kein Suchen mehr in GitHub-Repositories oder Confluence-Seiten. |
| Ownership     | Jede Komponente hat eine klare Verantwortlichkeit. Bei Fragen oder Incidents ist sofort ersichtlich, welches Team zuständig ist.                              |
| Dependencies  | Beziehungen zwischen Komponenten werden sichtbar. Du erkennst, welche Systeme von deinem Service abhängen und welche Abhängigkeiten dein Service selbst hat.  |
| Metadata      | Standardisierte Informationen wie Lifecycle Stage, Systemzuordnung und Tags erleichtern die Kategorisierung.                                                  |
| Documentation | Direkte Links zu TechDocs, API-Spezifikationen, Runbooks und Architekturdiagrammen.                                                                           |

Der Catalog ist die zentrale Quelle für die Frage, welche Software in deiner Plattform existiert.

Alles, was Entwickler über einen Service wissen müssen, beginnt mit seinem Catalog-Eintrag.

---

## Struktur von `catalog-info.yaml`

Jede Komponente wird in einer YAML-Datei beschrieben. Diese Datei folgt dem Entity-Format von Backstage.

Beispiel für das TaskFlow Backend:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  title: TaskFlow API Backend
  description: FastAPI backend providing REST API for task management with PostgreSQL persistence
  annotations:
    github.com/project-slug: tekanaid/taskflow-backend
    backstage.io/techdocs-ref: dir:.
  tags:
    - python
    - fastapi
    - api
    - postgresql
  links:
    - url: https://api.taskflow.dev
      title: Production API
      icon: web
    - url: https://github.com/tekanaid/taskflow-backend
      title: Source Code
      icon: github
spec:
  type: service
  lifecycle: production
  owner: platform-team
  system: taskflow
  dependsOn:
    - resource:taskflow-database
    - resource:taskflow-redis
  providesApis:
    - taskflow-api
```

---

## Aufbau der Datei

### `apiVersion`

Die Version des Backstage Entity Schemas.

Aktuell wird verwendet:

* `backstage.io/v1alpha1`

### `kind`

Der Entity Type.

`Component` ist der häufigste Typ.

Weitere mögliche `kind`-Werte sind:

* `API`
* `Resource`
* `System`
* `Domain`
* `Group`
* `User`

### `metadata`

Enthält die zentralen Informationen zur Komponente.

| Feld          | Beschreibung                                                                         |
| ------------- | ------------------------------------------------------------------------------------ |
| `name`        | Eindeutiger Identifier. Kleinschreibung und Bindestriche werden empfohlen.           |
| `title`       | Lesbarer Anzeigename für Benutzer.                                                   |
| `description` | Kurze Beschreibung der Funktion der Komponente.                                      |
| `annotations` | Integrationsspezifische Metadaten, zum Beispiel GitHub-Projekt oder TechDocs-Quelle. |
| `tags`        | Suchbare Labels für Technologie, Zweck oder Team.                                    |
| `links`       | Externe URLs, zum Beispiel Produktionssysteme, Dashboards oder Dokumentation.        |

### `spec`

Enthält typspezifische Details.

| Feld           | Beschreibung                                                                                  |
| -------------- | --------------------------------------------------------------------------------------------- |
| `type`         | Kategorie der Component, zum Beispiel `service`, `library` oder `website`.                    |
| `lifecycle`    | Status der Komponente, zum Beispiel `experimental`, `production` oder `deprecated`.           |
| `owner`        | Verantwortliches Team oder verantwortlicher Benutzer. Referenziert meist eine `Group` Entity. |
| `system`       | Übergeordnetes System, zu dem die Komponente gehört.                                          |
| `dependsOn`    | Andere Components oder Resources, von denen diese Komponente abhängt.                         |
| `providesApis` | APIs, die diese Komponente bereitstellt.                                                      |

---

## Component Types

Backstage verwendet verschiedene Component Types, um Software zu kategorisieren.

| Type       | Beschreibung                                                                                       | TaskFlow-Beispiel                              |
| ---------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `service`  | Backend-Service oder Microservice, der eine API bereitstellt und dauerhaft läuft.                  | TaskFlow Backend mit FastAPI                   |
| `website`  | Frontend-Anwendung, die HTML, CSS und JavaScript für Benutzer bereitstellt.                        | TaskFlow Vue.js Frontend                       |
| `library`  | Wiederverwendbarer Code, der von mehreren Services genutzt wird, zum Beispiel ein gemeinsames SDK. | Gemeinsamer Python API Client, falls vorhanden |
| `resource` | Infrastruktur wie Datenbanken, Message Queues, Caches oder Object Storage.                         | PostgreSQL Datenbank, Redis Cache              |

Die Wahl des richtigen Typs beeinflusst, wie Backstage die Komponente darstellt.

Services zeigen zum Beispiel Deployment-Informationen und API Endpoints. Websites zeigen URLs und Hosting-Plattformen. Resources zeigen technische Informationen wie Connection Strings oder Kapazitätsmetriken.

---

## TaskFlow-Komponenten registrieren

> **Abbildung:** TaskFlow Catalog Entity Hierarchy — Domain, System, Components, Resources und API-Beziehungen.

Jetzt registrieren wir alle vier TaskFlow-Komponenten im Catalog.

---

## TaskFlow Frontend

Das TaskFlow Frontend wird als `website` registriert.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-frontend
  title: TaskFlow Web UI
  description: Vue.js 3 frontend application for task management with real-time updates
  annotations:
    github.com/project-slug: tekanaid/taskflow-frontend
  tags:
    - vuejs
    - frontend
    - javascript
spec:
  type: website
  lifecycle: production
  owner: platform-team
  system: taskflow
  consumesApis:
    - taskflow-api
```

---

## TaskFlow Backend

Das TaskFlow Backend wird als `service` registriert.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  title: TaskFlow API Backend
  description: FastAPI backend providing REST API for task management
  annotations:
    github.com/project-slug: tekanaid/taskflow-backend
  tags:
    - python
    - fastapi
    - api
spec:
  type: service
  lifecycle: production
  owner: platform-team
  system: taskflow
  dependsOn:
    - resource:taskflow-database
    - resource:taskflow-redis
  providesApis:
    - taskflow-api
```

---

## TaskFlow Datenbank

Die PostgreSQL-Datenbank wird als `Resource` registriert.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: taskflow-database
  title: TaskFlow PostgreSQL Database
  description: PostgreSQL 14 database storing tasks and user data
  tags:
    - postgresql
    - database
spec:
  type: database
  owner: platform-team
  system: taskflow
```

---

## TaskFlow Redis

Der Redis Cache wird ebenfalls als `Resource` registriert.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: taskflow-redis
  title: TaskFlow Redis Cache
  description: Redis cache for session storage and rate limiting
  tags:
    - redis
    - cache
spec:
  type: cache
  owner: platform-team
  system: taskflow
```

---

## Abhängigkeiten im Catalog

Die Abhängigkeiten werden direkt in den Entities beschrieben.

Das Backend verwendet:

* `dependsOn` für die Abhängigkeit zur Datenbank
* `dependsOn` für die Abhängigkeit zu Redis
* `providesApis` für die bereitgestellte API

Das Frontend verwendet:

* `consumesApis` für die Nutzung der Backend API

Backstage nutzt diese Informationen, um einen Dependency Graph aufzubauen. Dadurch werden technische Beziehungen zwischen Components, APIs und Resources sichtbar.

---

## GitHub-Repositories verknüpfen

Die Annotation `github.com/project-slug` verbindet Catalog Components mit GitHub-Repositories.

```yaml
annotations:
  github.com/project-slug: tekanaid/taskflow-backend
```

Wenn diese Annotation gesetzt ist und die GitHub-Integration in `app-config.yaml` konfiguriert wurde, kann Backstage zusätzliche Repository-Informationen anzeigen.

Dazu gehören:

* aktuelle Commits und Pull Requests auf der Component-Seite
* Repository README
* Links zu GitHub Actions Workflows
* Repository Contributors
* Shortcuts zum Erstellen von Issues und Pull Requests

### GitHub-Integration aktivieren

Füge die GitHub-Integration in `app-config.yaml` hinzu.

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

Setze die Umgebungsvariable `GITHUB_TOKEN` auf ein Personal Access Token mit `repo` Scope.

Backstage verwendet dieses Token, um Repository-Daten von GitHub abzurufen.

---

## API-Definitionen und Abhängigkeiten

APIs sind eigenständige Entities in Backstage.

Eine API wird separat von Components definiert.

```yaml
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: taskflow-api
  title: TaskFlow REST API
  description: RESTful API for task CRUD operations
  annotations:
    github.com/project-slug: tekanaid/taskflow-backend
spec:
  type: openapi
  lifecycle: production
  owner: platform-team
  system: taskflow
  definition: |
    openapi: 3.0.0
    info:
      title: TaskFlow API
      version: 1.0.0
    paths:
      /api/tasks:
        get:
          summary: List all tasks
          responses:
            '200':
              description: Array of tasks
```

Components können diese API anschließend referenzieren.

| Component | Beziehung zur API              |
| --------- | ------------------------------ |
| Backend   | `providesApis: [taskflow-api]` |
| Frontend  | `consumesApis: [taskflow-api]` |

Backstage rendert OpenAPI-Spezifikationen als interaktive Dokumentation mit Try-it-out-Funktionalität.

So werden APIs auffindbar und selbstdokumentierend.

---

## Ownership und Team-Organisation

Das Feld `owner` referenziert eine `Group` Entity. Diese Gruppe repräsentiert ein Team.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: platform-team
  title: Platform Engineering Team
  description: Team responsible for TaskFlow and developer tooling
spec:
  type: team
  children: []
```

Wenn diese Entity im Catalog registriert ist, verlinken alle Components mit `owner: platform-team` auf diese Gruppe.

Team-Seiten zeigen alle Components, APIs und Resources, die diesem Team gehören.

---

## Benutzer als Catalog Entities

Auch Benutzer können als Entities im Catalog geführt werden.

```yaml
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: jane-doe
  title: Jane Doe
spec:
  memberOf:
    - platform-team
```

Dadurch entsteht in Backstage eine Organisationsstruktur, die reale Teams und Zuständigkeiten abbildet.

---

## Organisation über Systems und Domains

Systems gruppieren zusammengehörige Components.

TaskFlow ist ein System, das Frontend, Backend, Datenbank und Redis umfasst.

```yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: taskflow
  title: TaskFlow Application
  description: Task management platform with Vue.js frontend and FastAPI backend
spec:
  owner: platform-team
  domain: developer-productivity
```

Domains repräsentieren Geschäftsbereiche oder Produktkategorien.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Domain
metadata:
  name: developer-productivity
  title: Developer Productivity
  description: Tools and platforms that improve developer workflows
spec:
  owner: platform-team
```

Damit entsteht folgende Hierarchie:

```text
Domain → System → Components/APIs/Resources
```

Backstage visualisiert diese Struktur über Navigation, Filter und Beziehungsansichten.

---

## Components in Backstage registrieren

Catalog Entities können auf mehrere Arten registriert werden.

| Methode               | Funktionsweise                                                                                                            | Geeignet für                                 |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| Direkter YAML Import  | In Backstage zu Create → Register Existing Component navigieren und die GitHub URL zu einer `catalog-info.yaml` einfügen. | Einstieg und einzelne Registrierungen        |
| Location Entity       | Eine `kind: Location` YAML verweist auf mehrere Entity-Dateien. Backstage lädt alle Targets.                              | Mehrere Components gleichzeitig registrieren |
| GitHub Auto-Discovery | Backstage durchsucht automatisch alle Repositories einer GitHub-Organisation nach `catalog-info.yaml`-Dateien.            | Große Organisationen mit vielen Repositories |
| Static Configuration  | `catalog.locations` wird direkt in `app-config.yaml` eingetragen.                                                         | Dauerhafte und versionierte Registrierungen  |

Für TaskFlow verwenden wir den direkten Import für jede Komponente.

Dazu wird `catalog-info.yaml` im Root-Verzeichnis jedes Repositories committed. Anschließend werden die URLs in Backstage registriert.

---

## Ausblick

In dieser Lektion hast du die Struktur des Backstage Service Catalog mit `catalog-info.yaml` kennengelernt.

Du hast TaskFlow-Komponenten als Catalog Entities registriert:

* Frontend
* Backend
* Datenbank
* Redis

Du hast außerdem Metadaten, Ownership, Abhängigkeiten und API-Definitionen modelliert.

Als Nächstes geht es in die React Plugin-Entwicklung. Du lernst React Hooks genauer kennen:

* `useState`
* `useEffect`
* `useContext`

Dabei vergleichen wir diese Konzepte mit bekannten Vue.js-Patterns.

Anschließend erstellst du dein erstes Backstage Plugin, um TaskFlow-Metriken in einem eigenen Dashboard anzuzeigen.

Der Catalog bildet dafür die Grundlage. Plugins fragen häufig Catalog-Daten ab, um Component-Informationen, Deployments oder Metriken anzuzeigen.

Da TaskFlow jetzt sauber im Catalog modelliert ist, kannst du darauf aufbauend neue Developer-Portal-Funktionen entwickeln.

---

## Kernaussagen

* Der Backstage Catalog bietet Discovery, Ownership Tracking, Dependency Visualization und Metadaten für alle Software-Komponenten einer Organisation.
* Die Struktur von `catalog-info.yaml` besteht aus `apiVersion`, `kind`, `metadata` und `spec`.
* `metadata` enthält Informationen wie `name`, `description` und `tags`.
* `spec` beschreibt Details wie `type`, `lifecycle` und `owner`.
* Component Types umfassen `service`, `website`, `library` und `resource`.
* Abhängigkeiten werden mit `dependsOn`, `consumesApis` und `providesApis` beschrieben.
* Diese Beziehungen erzeugen navigierbare Relationship Graphs.
* Die GitHub-Integration über Annotationen ermöglicht die Anzeige von Commits, Pull Requests, Workflows und Contributors auf Component-Seiten.

---

## Zusätzliche Ressourcen

| Ressource                        | Typ           |
| -------------------------------- | ------------- |
| Backstage Catalog Documentation  | Dokumentation |
| Catalog Entity Descriptor Format | Dokumentation |
| Well-known Annotations Reference | Dokumentation |
| GitHub Integration Plugin        | Dokumentation |

---

## Private Lesson Notes

Private Notizen während des Lernens erfassen.

---

## Study Group

### Backstage – Internes Developer Portal

# Backstage Setup, React-Einführung und Service Catalog

> **Beta:** Dieses Lab befindet sich in der Beta-Phase. Inhalte können aktualisiert werden, während das Material weiter verbessert wird.

Richte ein internes Developer Portal mit Backstage ein und lerne React-Grundlagen durch den Vergleich mit Vue.js.

---

## Lab-Übersicht

| Eigenschaft        | Wert                             |
| ------------------ | -------------------------------- |
| Schwierigkeitsgrad | Fortgeschritten                  |
| Dauer              | 45 Minuten                       |
| Kategorie          | Developer Experience / Backstage |
| Typ                | Lab                              |

---

## Tags

* `backstage`
* `react`
* `vuejs`
* `service-catalog`
* `developer-portal`
* `platform-engineering`

---

## Voraussetzungen

Für dieses Lab solltest du mit folgenden Themen vertraut sein:

* `taskflow-frontend-development`
* `vuejs-fundamentals`
* `kubernetes-basics`
* `docker-fundamentals`
* `nodejs-basics`

---

## Über dieses Lab

In diesem Lab richtest du ein internes Developer Portal mit Backstage ein und lernst zentrale React-Grundlagen kennen. Der Einstieg erfolgt über den Vergleich mit Vue.js.

Du installierst Backstage, verstehst die grundlegende Architektur und vergleichst wichtige React-Patterns mit der Vue.js Composition API.

Dazu gehören:

* JSX
* Hooks
* `useState`
* `useEffect`

Außerdem erstellst du einen vollständigen Service Catalog für die TaskFlow-Komponenten. Dabei pflegst du korrekte Metadaten, Beziehungen und Integrationen.

Du registrierst alle TaskFlow-Microservices im Backstage Catalog:

* Frontend
* Backend
* Datenbank
* Redis

Zusätzlich dokumentierst du APIs und Abhängigkeiten, verknüpfst GitHub-Repositories und richtest Catalog Relationships ein.

So lernst du React direkt im Kontext praktischer Platform-Engineering-Aufgaben.

---

## Was du lernst

Dieses Lab besteht aus 2 Aufgaben.

| Nr. | Aufgabe                                                 | Ziel                                                                                                    |
| --: | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
|   1 | Backstage installieren                                  | Internes Developer Portal mit Backstage installieren                                                    |
|   2 | Backstage erkunden und TaskFlow im Catalog registrieren | Backstage Catalog UI kennenlernen und YAML-Catalog-Einträge für TaskFlow Backend und Frontend erstellen |

---

## Systemanforderungen

Für dieses Lab benötigst du:

* einen modernen Webbrowser,
* eine stabile Internetverbindung,
* ungefähr 45 Minuten konzentrierte Arbeitszeit,
* Kenntnisse in `taskflow-frontend-development`, `vuejs-fundamentals`, `kubernetes-basics`, `docker-fundamentals` und `nodejs-basics`.


# React Hooks und Component-Entwicklung

Beherrsche React Hooks wie `useState`, `useEffect` und `useContext`, um Backstage Plugins zu entwickeln. Du lernst State Management, Side Effects, Event Handling, Conditional Rendering, TypeScript Interfaces und Material-UI Components. Dabei vergleichen wir die Konzepte direkt mit bekannten Vue.js-Patterns.

---

## Lernmaterial

Nachdem du die grundlegenden Unterschiede zwischen React und Vue.js kennengelernt hast, geht es jetzt tiefer in React Hooks.

Hooks sind die zentralen Bausteine moderner React Components. Sie wurden mit React 16.8 eingeführt. Seitdem können Functional Components State und Side Effects verwenden, ohne Class Components schreiben zu müssen.

Du lernst:

* `useState` für State Management
* `useEffect` für Side Effects
* `useContext` für gemeinsam genutzte Daten über Components hinweg

Wir vergleichen diese Konzepte direkt mit der Vue.js Composition API. So kannst du vorhandenes Wissen gezielt übertragen.

Am Ende dieser Lektion bist du bereit, Backstage Plugin Components zu entwickeln.

---

## Warum React Hooks?

Vor Hooks gab es in React zwei Hauptarten von Components:

| Component-Typ         | Eigenschaften                                                               |
| --------------------- | --------------------------------------------------------------------------- |
| Class Components      | Unterstützten State und Lifecycle Methods, waren aber komplexer             |
| Functional Components | Einfach und gut für reine Darstellung, aber ursprünglich ohne eigenen State |

Das führte zu unnötiger Komplexität. Entwickler mussten zwischen Logik in Klassen und einfacher Darstellung in Funktionen wählen.

Hooks lösen dieses Problem. Functional Components können damit State, Side Effects und gemeinsame Daten verwenden.

Die drei wichtigsten Hooks sind:

| Hook         | Zweck                                                                            |
| ------------ | -------------------------------------------------------------------------------- |
| `useState`   | Fügt Functional Components lokalen State hinzu                                   |
| `useEffect`  | Behandelt Side Effects wie Data Fetching, Subscriptions und DOM-Manipulation     |
| `useContext` | Greift auf gemeinsamen Context zu, ohne Props durch viele Ebenen weiterzureichen |

Diese Hooks fühlen sich ähnlich an wie die Vue.js Composition API. Wenn du `ref()`, `onMounted()` und `provide/inject` in Vue verstehst, wirst du React Hooks schnell einordnen können.

---

## `useState`: Component State verwalten

`useState` fügt Functional Components reaktiven State hinzu.

```js
import { useState } from 'react';

function TaskCounter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Total tasks: {count}</p>
      <button onClick={() => setCount(count + 1)}>Add Task</button>
    </div>
  );
}
```

`useState` gibt ein Array mit zwei Elementen zurück:

| Element    | Bedeutung                                    |
| ---------- | -------------------------------------------- |
| `count`    | Aktueller State-Wert                         |
| `setCount` | Setter-Funktion zum Aktualisieren des States |

---

## Vergleich mit Vue.js

```js
// Vue.js Composition API
import { ref } from 'vue';

const count = ref(0);
count.value++; // Direct mutation

// React hooks
import { useState } from 'react';

const [count, setCount] = useState(0);
setCount(count + 1); // Setter function
```

| Thema               | Vue.js `ref`                      | React `useState`                                             |
| ------------------- | --------------------------------- | ------------------------------------------------------------ |
| Rückgabewert        | Objekt mit `.value` Property      | Array `[value, setter]`, das per Destructuring entpackt wird |
| State aktualisieren | Direkte Mutation: `count.value++` | Setter aufrufen: `setCount(count + 1)`                       |
| Immutability        | Mutationen sind erlaubt           | State darf nie direkt verändert werden                       |

Der React-Ansatz ist expliziter. Jede State-Änderung ist an einem Setter-Aufruf wie `setCount()` erkennbar.

---

## Mehrere State-Variablen

Components benötigen oft mehrere State-Werte. Dafür kann `useState` mehrfach aufgerufen werden.

```js
function TaskManager() {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  return (
    <div>
      {loading && <p>Loading...</p>}
      {error && <p>Error: {error}</p>}
      {tasks.map(task => <li key={task.id}>{task.title}</li>)}
    </div>
  );
}
```

### Vergleich mit Vue.js

```js
// Vue.js
const tasks = ref([]);
const loading = ref(true);
const error = ref(null);

// React
const [tasks, setTasks] = useState([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);
```

Beide Frameworks erlauben beliebig viele State-Variablen.

React ist durch Array Destructuring etwas ausführlicher als Vue mit `ref()`. Die Struktur bleibt aber klar.

---

## `useState` mit Objects und Arrays

Wenn der State ein Objekt oder Array ist, musst du neue Objekte oder Arrays erzeugen. React erkennt direkte Mutationen nicht zuverlässig.

```js
function TaskForm() {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    priority: 'medium'
  });

  const updateField = (field, value) => {
    setFormData({
      ...formData,  // Spread existing fields
      [field]: value  // Override specific field
    });
  };

  return (
    <input
      value={formData.title}
      onChange={(e) => updateField('title', e.target.value)}
    />
  );
}
```

Für Arrays verwendest du immutable Array-Methoden.

```js
const [tasks, setTasks] = useState([]);

// Add item
setTasks([...tasks, newTask]);

// Remove item
setTasks(tasks.filter(t => t.id !== taskId));

// Update item
setTasks(tasks.map(t =>
  t.id === taskId ? { ...t, completed: true } : t
));
```

### Vergleich mit Vue.js

```js
// Vue.js allows mutations
formData.value.title = 'New title';
tasks.value.push(newTask);

// React requires immutability
setFormData({ ...formData, title: 'New title' });
setTasks([...tasks, newTask]);
```

React verlangt Immutability. Das wirkt am Anfang strenger, verhindert aber viele Fehler und ermöglicht Performance-Optimierungen.

---

## `useEffect`: Side Effects behandeln

`useEffect` behandelt Side Effects in Functional Components.

Dazu gehören:

* Data Fetching
* Subscriptions
* DOM-Manipulation
* Logging
* Timer
* Event Listener

`useEffect` ersetzt klassische Lifecycle Methods wie `componentDidMount`, `componentDidUpdate` und `componentWillUnmount`.

### Grundsyntax

```js
import { useState, useEffect } from 'react';

function TaskList() {
  const [tasks, setTasks] = useState([]);

  useEffect(() => {
    // This runs after component renders
    async function fetchTasks() {
      const response = await fetch('/api/tasks');
      const data = await response.json();
      setTasks(data);
    }
    fetchTasks();
  }, []); // Empty array = run once on mount

  return (
    <ul>
      {tasks.map(task => <li key={task.id}>{task.title}</li>)}
    </ul>
  );
}
```

---

## Vergleich mit Vue.js

```js
// Vue.js
import { ref, onMounted } from 'vue';

const tasks = ref([]);

onMounted(async () => {
  const response = await fetch('/api/tasks');
  tasks.value = await response.json();
});

// React
const [tasks, setTasks] = useState([]);

useEffect(() => {
  async function fetchTasks() {
    const response = await fetch('/api/tasks');
    setTasks(await response.json());
  }
  fetchTasks();
}, []);
```

Beide Varianten führen Code nach dem ersten Render aus.

Vue.js `onMounted` ist direkter. React `useEffect` ist flexibler, weil Dependency Arrays steuern, wann der Effect erneut läuft.

---

## Dependency Arrays in `useEffect`

Das Dependency Array ist der zweite Parameter von `useEffect`. Es bestimmt, wann der Effect ausgeführt wird.

| Dependency Array | Wann der Effect läuft                           |
| ---------------- | ----------------------------------------------- |
| `[]`             | Einmal beim Mount                               |
| Kein Array       | Nach jedem Render                               |
| `[count]`        | Wenn sich eine der angegebenen Variablen ändert |

```js
useEffect(() => { console.log('Component mounted'); }, []);        // once on mount
useEffect(() => { console.log('Component rendered'); });           // every render
useEffect(() => { console.log(`Count: ${count}`); }, [count]);    // when count changes
```

### Vergleich mit Vue.js

```js
// Vue.js watch
watch(count, (newValue) => {
  console.log(`Count changed to ${newValue}`);
});

// React useEffect
useEffect(() => {
  console.log(`Count changed to ${count}`);
}, [count]);
```

`useEffect` mit Dependencies ähnelt Vue.js `watch`.

Der Unterschied: In React musst du alle Dependencies manuell angeben. Fehlende Dependencies sind eine häufige Fehlerquelle. Ein ESLint Plugin hilft, solche Probleme früh zu erkennen.

---

## Cleanup Functions in `useEffect`

Effects können eine Cleanup Function zurückgeben.

Das ist wichtig für:

* Subscriptions
* Timer
* Event Listener
* offene Verbindungen

```js
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);

    return () => {
      clearInterval(interval); // Cleanup on unmount
    };
  }, []);

  return <p>Elapsed: {seconds}s</p>;
}
```

### Vergleich mit Vue.js

```js
// Vue.js
import { ref, onMounted, onUnmounted } from 'vue';

const seconds = ref(0);
let interval;

onMounted(() => {
  interval = setInterval(() => {
    seconds.value++;
  }, 1000);
});

onUnmounted(() => {
  clearInterval(interval);
});

// React
useEffect(() => {
  const interval = setInterval(() => {
    setSeconds(s => s + 1);
  }, 1000);

  return () => clearInterval(interval);
}, []);
```

Vue.js verwendet getrennte Hooks wie `onMounted` und `onUnmounted`.

React bündelt Setup und Cleanup in einem `useEffect`. Die Cleanup-Logik wird als Rückgabefunktion definiert.

---

## Component Props und TypeScript Interfaces

React Components erhalten Props als Funktionsparameter. TypeScript Interfaces definieren die Typen dieser Props.

```tsx
interface TaskCardProps {
  task: {
    id: string;
    title: string;
    completed: boolean;
  };
  onComplete: (id: string) => void;
}

function TaskCard({ task, onComplete }: TaskCardProps) {
  return (
    <div>
      <h3>{task.title}</h3>
      <button onClick={() => onComplete(task.id)}>
        {task.completed ? 'Undo' : 'Complete'}
      </button>
    </div>
  );
}
```

### Vergleich mit Vue.js

```ts
// Vue.js
interface Props {
  task: {
    id: string;
    title: string;
    completed: boolean;
  };
}

const props = defineProps<Props>();
const emit = defineEmits<{
  complete: [id: string]
}>();

// React
interface TaskCardProps {
  task: { id: string; title: string; completed: boolean };
  onComplete: (id: string) => void;
}

function TaskCard({ task, onComplete }: TaskCardProps) {
  // ...
}
```

Vue.js verwendet `defineProps` und `defineEmits`.

React verwendet Funktionsparameter. Events werden als Callback Props modelliert.

---

## Event Handling in React

React Event Handler verwenden CamelCase-Namen wie `onClick`, `onChange` und `onSubmit`.

```tsx
function TaskForm() {
  const [title, setTitle] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log('Submitted:', title);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Task title"
      />
      <button type="submit">Add Task</button>
    </form>
  );
}
```

Wichtige Punkte:

* Verwende CamelCase: `onClick`, nicht `onclick`.
* Übergib Funktionsreferenzen: `onClick={handleSubmit}`, nicht `onClick={handleSubmit()}`.
* Das Event Object ist synthetisch. React kapselt native Browser Events.
* Verwende `e.preventDefault()`, um Standardverhalten zu verhindern.

### Vergleich mit Vue.js

```vue
<!-- Vue.js -->
<form @submit.prevent="handleSubmit">
  <input v-model="title" placeholder="Task title" />
  <button type="submit">Add Task</button>
</form>

<!-- React JSX -->
<form onSubmit={handleSubmit}>
  <input value={title} onChange={(e) => setTitle(e.target.value)} />
  <button type="submit">Add Task</button>
</form>
```

Vue.js verwendet die `@click`-Syntax und Modifier wie `.prevent`.

React verwendet `onClick` und ruft `e.preventDefault()` explizit auf.

---

## Conditional Rendering und Listen

Conditional Rendering in React basiert auf JavaScript-Ausdrücken.

```tsx
function TaskList({ tasks, loading, error }) {
  if (loading) return <p>Loading tasks...</p>;
  if (error) return <p>Error: {error}</p>;
  if (tasks.length === 0) return <p>No tasks yet</p>;

  return (
    <ul>
      {tasks.map(task => (
        <li key={task.id}>
          {task.title}
          {task.completed && <span>✓</span>}
        </li>
      ))}
    </ul>
  );
}
```

Typische Patterns:

| Pattern          | Verwendung                                     |
| ---------------- | ---------------------------------------------- |
| Early Return     | Für Loading-, Error- oder Empty States         |
| `&&`             | Inline Rendering, wenn eine Bedingung wahr ist |
| Ternary Operator | Rendering mit zwei möglichen Varianten         |
| `.map()`         | Listen rendern, immer mit `key` Prop           |

### Vergleich mit Vue.js

```vue
<template>
  <p v-if="loading">Loading tasks...</p>
  <p v-else-if="error">Error: {{ error }}</p>
  <p v-else-if="tasks.length === 0">No tasks yet</p>
  <ul v-else>
    <li v-for="task in tasks" :key="task.id">
      {{ task.title }}
      <span v-if="task.completed">✓</span>
    </li>
  </ul>
</template>
```

Vue.js verwendet Direktiven wie `v-if`, `v-else-if` und `v-for`.

React verwendet normales JavaScript: `if`, `&&`, `? :` und `.map()`.

---

## Material-UI Component Library

Backstage verwendet Material-UI, kurz MUI, für UI Components.

MUI stellt vorgefertigte React Components bereit.

```tsx
import { Card, CardContent, Button, Typography } from '@material-ui/core';

function TaskCard({ task }) {
  return (
    <Card>
      <CardContent>
        <Typography variant="h5">{task.title}</Typography>
        <Typography variant="body2">{task.description}</Typography>
        <Button variant="contained" color="primary">
          Complete
        </Button>
      </CardContent>
    </Card>
  );
}
```

| MUI Component                        | Zweck                                                        |
| ------------------------------------ | ------------------------------------------------------------ |
| `Card`, `CardContent`, `CardActions` | Container Layouts                                            |
| `Typography`                         | Text mit Material Design Styles                              |
| `Button`                             | Buttons mit Varianten wie `text`, `outlined` und `contained` |
| `Grid`                               | Responsive Layouts                                           |
| `TextField`                          | Form Inputs                                                  |
| `Dialog`                             | Modal Dialogs                                                |

MUI ist vergleichbar mit Vuetify im Vue-Ökosystem. Beide liefern vorgefertigte Components nach Material Design.

---

## React Development Workflow und Debugging

Für React-Entwicklung sind folgende Werkzeuge besonders hilfreich.

### React DevTools

React DevTools ist eine Browser-Erweiterung. Sie zeigt Component Tree, Props und State.

Sie ist essenziell für das Debugging von React-Anwendungen.

### Console Logging

`console.log` kann in Render-Funktionen und Effects genutzt werden.

```js
useEffect(() => {
  console.log('Tasks updated:', tasks);
}, [tasks]);
```

### Error Boundaries

Error Boundaries fangen Rendering-Fehler ab.

```tsx
class ErrorBoundary extends React.Component {
  componentDidCatch(error) {
    console.error('Component error:', error);
  }
}
```

### Strict Mode

Strict Mode hilft, potenzielle Probleme früh zu erkennen.

```tsx
<React.StrictMode>
  <App />
</React.StrictMode>
```

---

## Häufige Probleme und Debugging-Hinweise

| Symptom                                  | Was du prüfen solltest                                                                     |
| ---------------------------------------- | ------------------------------------------------------------------------------------------ |
| Component rendert nicht neu              | Stelle sicher, dass die `useState` Setter-Funktion aufgerufen wird                         |
| Endlosschleife in `useEffect`            | Prüfe das Dependency Array. Meist fehlt eine Dependency oder es ist eine falsche enthalten |
| Props werden nicht aktualisiert          | Prüfe, ob die Parent Component neu rendert, wenn sich der Wert ändert                      |
| State Updates sind nicht sofort sichtbar | State Updates sind asynchron. Logge den State nach dem nächsten Render                     |

---

## Ausblick

In dieser Lektion hast du React Hooks vertieft.

Du hast gelernt:

* `useState` für State Management
* `useEffect` für Side Effects
* Cleanup Functions in Effects
* TypeScript Interfaces für Props
* Event Handling in React
* Conditional Rendering
* Listen mit `.map()`
* Material-UI Components
* Debugging-Techniken

Außerdem hast du gesehen, wie diese Konzepte zu Vue.js Patterns passen.

Als Nächstes wendest du diese React-Kenntnisse auf die Backstage Plugin-Entwicklung an.

Du lernst:

* Backstage Plugin Architecture
* Erstellen eines neuen Plugins mit der CLI
* Plugin Routing und Navigation
* Integration von Plugins in die Backstage App

Diese Grundlage bereitet dich darauf vor, das TaskFlow Dashboard Plugin zu bauen.

Danach implementierst du ein Dashboard mit Echtzeitmetriken, Task-Statistiken und Deployment-Status. Dafür nutzt du React Components, Material-UI und die TaskFlow API.

Am Ende dieses Abschnitts hast du ein vollständiges Backstage Plugin mit React Hooks gebaut.

---

## Kernaussagen

* `useState` gibt `[value, setter]` zurück und verlangt immutable Updates. Für Objekte und Arrays werden häufig Spread Operator, `.filter()` und `.map()` verwendet.
* `useEffect` führt Side Effects nach dem Render aus. Dependency Arrays steuern, wann Effects erneut ausgeführt werden.
* `[]` führt einen Effect einmal beim Mount aus. `[var]` führt ihn erneut aus, wenn sich `var` ändert.
* Cleanup Functions werden aus `useEffect` zurückgegeben und ersetzen Patterns wie `onUnmounted` aus Vue.js.
* React Event Handler verwenden CamelCase wie `onClick` und `onChange`.
* Event Handler erhalten Funktionsreferenzen und synthetische Event Objects.
* Material-UI stellt vorgefertigte React Components wie `Card`, `Button` und `Typography` für die Entwicklung von Backstage Plugin UIs bereit.

---

## Zusätzliche Ressourcen

| Ressource                      | Typ           |
| ------------------------------ | ------------- |
| React Hooks API Reference      | Dokumentation |
| `useState` Hook Documentation  | Dokumentation |
| `useEffect` Hook Documentation | Dokumentation |
| Material-UI Component Library  | Dokumentation |

---

## Private Lesson Notes

Private Notizen während des Lernens erfassen.

---

## Study Group

### Backstage – Internes Developer Portal

# Plattformintegration und Self-Service-Automatisierung

Vervollständige dein internes Developer Portal, indem du Backstage mit GitHub, Kubernetes, Monitoring-Tools und Dokumentationssystemen integrierst.

In dieser Lektion konfigurierst du die Authentifizierung mit GitHub OAuth, implementierst TechDocs für Documentation as Code, bindest Monitoring-Dashboards ein und misst Verbesserungen der Developer Experience für die TaskFlow-Plattform.

---

## Lernmaterial

Du hast bereits das TaskFlow Dashboard Plugin gebaut und Software Templates für das Projekt-Scaffolding erstellt. Jetzt wird Backstage mit dem erweiterten Plattform-Ökosystem verbunden:

* GitHub
* Kubernetes
* Monitoring-Tools
* Dokumentationssysteme

Diese Integrationen machen Backstage von einem eigenständigen Portal zum zentralen Hub für Entwickler-Workflows.

Entwickler können Pull Requests prüfen, Deployment-Status einsehen, Metrik-Dashboards öffnen und Dokumentation lesen, ohne Backstage zu verlassen.

Das reduziert Kontextwechsel und verbessert die Produktivität.

In dieser abschließenden Lektion lernst du:

* GitHub-Integration
* Kubernetes-Plugin-Konfiguration
* Anbindung von Monitoring-Tools
* TechDocs für Documentation as Code
* Authentifizierung und Autorisierung
* Messung des Einflusses deines internen Developer Portals auf die Developer Experience

---
===========================================

# Backstage Plugins entwickeln

Beherrsche die Backstage Plugin-Architektur und den Entwicklungsworkflow. Du lernst Plugin-Typen wie `frontend`, `backend` und `common`, die Verzeichnisstruktur, Plugin-Erstellung mit der Backstage CLI, Routing, Navigation, Konfiguration und API-Integration zur Erweiterung von Developer Portals.

---

## Lernmaterial

Plugins sind der zentrale Erweiterungsmechanismus in Backstage.

Jede Funktion, die du in Backstage siehst, ist als Plugin umgesetzt:

* Catalog
* Scaffolder
* Search
* Kubernetes Integration
* GitHub Integration

Diese modulare Architektur macht Backstage sehr flexibel. Du kannst das Portal exakt an die Anforderungen deiner Organisation anpassen.

In dieser Lektion lernst du:

* Backstage Plugin-Architektur
* Unterschied zwischen Frontend- und Backend-Plugins
* Plugin-Erstellung mit der Backstage CLI
* Plugin-Struktur
* Routing
* Navigation
* API-Integration

Dieses Wissen bereitet dich darauf vor, in der nächsten Lektion das TaskFlow Dashboard Plugin zu bauen.

---

## Backstage Plugin-Architektur

> **Abbildung:** Backstage Plugin Architecture — Frontend-, Backend- und Common-Library-Plugins mit Kommunikationskanälen.

Backstage Plugins folgen einer modularen Architektur.

Jedes Plugin ist ein eigenständiges Package mit eigener Funktionalität, eigenen Routes und eigenen Abhängigkeiten.

Plugins können unterschiedliche Typen haben.

| Plugin-Typ     | Runtime                      | Zweck                                                                                  | Beispiele                      |
| -------------- | ---------------------------- | -------------------------------------------------------------------------------------- | ------------------------------ |
| Frontend       | React im Browser             | UI-Funktionen hinzufügen: neue Seiten, Dashboard Cards, Catalog Tabs                   | `kubernetes`, `github-actions` |
| Backend        | Node.js auf dem Server       | REST APIs bereitstellen, externe Systeme integrieren, Daten verarbeiten                | `auth`, `scaffolder-backend`   |
| Common Library | Shared, Frontend und Backend | TypeScript Types, Utilities und API-Client-Code, der von mehreren Plugins genutzt wird | Gemeinsame Datenmodelle        |

---

## Kommunikation zwischen Plugins

Plugins kommunizieren über mehrere Kanäle.

| Kanal   | Funktionsweise                                                                             |
| ------- | ------------------------------------------------------------------------------------------ |
| APIs    | Backend-Plugins stellen REST Endpoints bereit. Frontend-Plugins rufen diese Endpoints auf. |
| Context | Die React Context API teilt Daten zwischen Frontend Components.                            |
| Events  | Ein Pub/Sub Event Bus ermöglicht asynchrone Kommunikation.                                 |
| Catalog | Frontend-Plugins fragen den Catalog ab, um Entity-Daten zu erhalten.                       |

---

## Plugin-Ökosystem

Das Plugin-Ökosystem besteht aus mehreren Kategorien.

| Kategorie        | Maintainer         | Beispiele                                             |
| ---------------- | ------------------ | ----------------------------------------------------- |
| Core Plugins     | Backstage Team     | `catalog`, `scaffolder`, `techdocs`                   |
| Contrib Plugins  | Community          | GitHub, Kubernetes, Jenkins, PagerDuty                |
| Internal Plugins | Deine Organisation | Eigene Dashboards, organisationsspezifische Workflows |

Für TaskFlow baust du ein internes Frontend Plugin. Es zeigt Metriken und Deployment-Status an.

---

## Plugin-Typen im Detail

| Plugin-Typ     | Verantwortlichkeiten                                                                                                                            | Beispiele                                                                    |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Frontend       | Seiten und Navigation hinzufügen, React Dashboard Components bereitstellen, Catalog Entity Informationen rendern, Backend APIs integrieren      | `kubernetes` zeigt Pods, `github-actions` zeigt Workflows                    |
| Backend        | REST API Endpoints bereitstellen, externe Systeme wie Datenbanken oder Cloud Provider integrieren, Daten verarbeiten, Webhooks behandeln        | `auth` für Authentifizierung, `scaffolder-backend` für Template-Verarbeitung |
| Common Library | TypeScript Types und Interfaces definieren, Utility-Funktionen teilen, API-Client-Code bereitstellen, der von Frontend und Backend genutzt wird | Gemeinsame Datenmodelle für konsistente Typisierung                          |

---

## Plugin-Verzeichnisstruktur

Ein Frontend Plugin hat typischerweise folgende Struktur:

```text
plugins/taskflow-dashboard/
├── package.json              # Plugin metadata and dependencies
├── src/
│   ├── plugin.ts            # Plugin entry point and routing
│   ├── components/
│   │   ├── TaskFlowDashboard/
│   │   │   ├── TaskFlowDashboard.tsx    # Main component
│   │   │   └── index.ts
│   │   └── index.ts
│   ├── api/
│   │   ├── TaskFlowClient.ts           # API client
│   │   └── index.ts
│   └── index.ts             # Public exports
└── README.md
```

---

## Wichtige Dateien

| Datei oder Verzeichnis | Zweck                                                                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `package.json`         | Definiert Plugin-Name, Version, Dependencies und Peer Dependencies wie React, Material-UI und Backstage Core Packages.                |
| `src/plugin.ts`        | Registriert das Plugin in der Backstage App. Definiert Routes, Navigation und Component Mounting.                                     |
| `src/components/`      | Enthält React Components für die Plugin UI. Jede Component liegt in einem eigenen Verzeichnis mit Component-Datei, Tests und `index`. |
| `src/api/`             | Enthält API-Client-Code für Backend Services. Behandelt Authentifizierung, Fehlerbehandlung und Data Fetching.                        |
| `src/index.ts`         | Öffentliche Exports. Exportiert nur das, was andere Plugins verwenden sollen, zum Beispiel Components, Types und API Factories.       |

---

## Plugin mit der Backstage CLI erstellen

Die Backstage CLI automatisiert das Plugin-Scaffolding.

```bash
cd backstage-app
yarn backstage-cli create-plugin
```

Du wirst nach folgenden Informationen gefragt:

| Eingabe     | Beschreibung                                              |
| ----------- | --------------------------------------------------------- |
| Plugin ID   | Eindeutiger Identifier, zum Beispiel `taskflow-dashboard` |
| Owner       | GitHub Username oder Organisation                         |
| Description | Beschreibung, was das Plugin macht                        |

Die CLI generiert:

* Plugin-Verzeichnis unter `plugins/`
* `package.json` mit Dependencies
* Beispiel-Component und Route
* Plugin-Registrierungscode

Für das TaskFlow Dashboard verwendest du:

```bash
yarn backstage-cli create-plugin
```

Dadurch wird `plugins/taskflow-dashboard/` mit Boilerplate-Code erstellt, den du anschließend anpasst.

---

## Plugin-Registrierung und Routing

Nach dem Erstellen eines Plugins muss es in der Backstage App registriert werden.

Bearbeite dazu `packages/app/src/App.tsx`.

```tsx
import { TaskFlowDashboardPage } from '@internal/plugin-taskflow-dashboard';

const routes = (
  <FlatRoutes>
    {/* Existing routes */}
    <Route path="/catalog" element={<CatalogIndexPage />} />
    <Route path="/docs" element={<TechDocsIndexPage />} />

    {/* Add TaskFlow Dashboard route */}
    <Route path="/taskflow" element={<TaskFlowDashboardPage />} />
  </FlatRoutes>
);
```

Dadurch entsteht die Route `/taskflow`.

Wenn Benutzer `http://localhost:3000/taskflow` öffnen, sehen sie deine Plugin-Seite.

---

## `plugin.ts`

Die Datei `plugin.ts` exportiert die Page Component.

```ts
import { createPlugin, createRoutableExtension } from '@backstage/core-plugin-api';

export const taskflowDashboardPlugin = createPlugin({
  id: 'taskflow-dashboard',
  routes: {
    root: rootRouteRef,
  },
});

export const TaskFlowDashboardPage = taskflowDashboardPlugin.provide(
  createRoutableExtension({
    name: 'TaskFlowDashboardPage',
    component: () =>
      import('./components/TaskFlowDashboard').then(m => m.TaskFlowDashboard),
    mountPoint: rootRouteRef,
  }),
);
```

| Konzept                   | Zweck                                                                                                         |
| ------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `createPlugin`            | Registriert das Plugin mit eindeutiger ID und Route-Definitionen.                                             |
| `createRoutableExtension` | Erstellt eine routbare Component, also eine vollständige Seite, die Backstage unter einer URL einhängen kann. |
| `import()`                | Lädt die Component erst, wenn die Route geöffnet wird. Das verbessert die initiale Ladezeit.                  |
| `mountPoint`              | Verbindet die Component mit der passenden Route Reference.                                                    |

---

## Plugin zur Sidebar-Navigation hinzufügen

Um einen Sidebar-Link hinzuzufügen, bearbeitest du `packages/app/src/components/Root/Root.tsx`.

```tsx
import DashboardIcon from '@material-ui/icons/Dashboard';

export const Root = ({ children }: PropsWithChildren<{}>) => (
  <SidebarPage>
    <Sidebar>
      {/* Existing items */}
      <SidebarItem icon={HomeIcon} to="/" text="Home" />
      <SidebarItem icon={CategoryIcon} to="catalog" text="Catalog" />

      {/* Add TaskFlow Dashboard */}
      <SidebarItem icon={DashboardIcon} to="taskflow" text="TaskFlow" />
    </Sidebar>
    {children}
  </SidebarPage>
);
```

Danach sehen Benutzer den Eintrag `TaskFlow` in der Sidebar.

Ein Klick führt zur Route `/taskflow` und zeigt die Plugin-Seite.

---

## Struktur einer Plugin Component

Die Haupt-Component verwendet React Hooks und Material-UI.

```tsx
import React from 'react';
import { Header, Page, Content } from '@backstage/core-components';
import { Grid } from '@material-ui/core';

export const TaskFlowDashboard = () => {
  return (
    <Page themeId="tool">
      <Header title="TaskFlow Dashboard" subtitle="Metrics and deployment status" />
      <Content>
        <Grid container spacing={3}>
          <Grid item xs={12} md={6}>
            {/* Task metrics card */}
          </Grid>
          <Grid item xs={12} md={6}>
            {/* Deployment status card */}
          </Grid>
        </Grid>
      </Content>
    </Page>
  );
};
```

| Backstage Component | Zweck                                       |
| ------------------- | ------------------------------------------- |
| `Page`              | Top-Level Container mit Theme-Unterstützung |
| `Header`            | Seitentitel und Untertitel                  |
| `Content`           | Hauptbereich mit Padding                    |
| `Grid`              | Responsives Grid-System von Material-UI     |

Diese Struktur entspricht anderen Backstage-Seiten und sorgt für eine konsistente Benutzeroberfläche.

---

## Plugin-Konfiguration

Plugins können Werte aus `app-config.yaml` lesen.

```yaml
taskflow:
  apiUrl: https://api.taskflow.dev
  refreshInterval: 30000  # 30 seconds
```

Diese Konfiguration wird im Plugin-Code gelesen.

```tsx
import { useApi, configApiRef } from '@backstage/core-plugin-api';

export const TaskFlowDashboard = () => {
  const config = useApi(configApiRef);
  const apiUrl = config.getString('taskflow.apiUrl');
  const refreshInterval = config.getNumber('taskflow.refreshInterval');

  // Use apiUrl and refreshInterval
};
```

| Practice                                                                    | Warum es wichtig ist                                                                               |
| --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Konfiguration für URLs, Intervalle und umgebungsspezifische Werte verwenden | Vermeidet hartcodierte Werte, die sich zwischen Development, Staging und Production unterscheiden. |
| Defaults im Plugin-Code bereitstellen                                       | Reduziert notwendige Konfiguration für einfache Nutzung.                                           |
| Erforderliche Konfiguration im Plugin README dokumentieren                  | Hilft anderen Entwicklern bei Installation und Einrichtung des Plugins.                            |
| `optional()` für optionale Config Keys verwenden                            | Verhindert Abstürze, wenn ein Key fehlt.                                                           |

---

## API-Integration in Plugins

Plugins rufen Backend APIs über `fetch` oder API Clients auf.

Erstelle dafür einen API Client.

### `src/api/TaskFlowClient.ts`

```ts
// src/api/TaskFlowClient.ts
export class TaskFlowClient {
  constructor(private baseUrl: string) {}

  async getMetrics(): Promise<TaskMetrics> {
    const response = await fetch(`${this.baseUrl}/api/metrics`);
    if (!response.ok) {
      throw new Error('Failed to fetch metrics');
    }
    return response.json();
  }

  async getDeploymentStatus(): Promise<DeploymentStatus> {
    const response = await fetch(`${this.baseUrl}/api/deployment-status`);
    if (!response.ok) {
      throw new Error('Failed to fetch deployment status');
    }
    return response.json();
  }
}
```

### Client in Components verwenden

```tsx
import { useState, useEffect } from 'react';
import { TaskFlowClient } from '../../api/TaskFlowClient';

export const TaskFlowDashboard = () => {
  const [metrics, setMetrics] = useState(null);
  const client = new TaskFlowClient('https://api.taskflow.dev');

  useEffect(() => {
    async function fetchMetrics() {
      const data = await client.getMetrics();
      setMetrics(data);
    }
    fetchMetrics();
  }, []);

  return <div>Total Tasks: {metrics?.totalTasks}</div>;
};
```

| Practice                                      | Zweck                                                                                       |
| --------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Client Classes erstellen                      | Hält API-Call-Logik organisiert und wiederverwendbar.                                       |
| TypeScript Interfaces für Responses verwenden | Erkennt Abweichungen zwischen Frontend-Annahmen und tatsächlichen API Responses frühzeitig. |
| Fehler mit `try/catch` behandeln              | Verhindert unhandled rejections, die das Plugin beschädigen könnten.                        |
| Loading States während des Fetchings anzeigen | Gibt Benutzern sichtbares Feedback, während Daten geladen werden.                           |
| Retry Logic implementieren                    | Behandelt kurzzeitige Netzwerkfehler robuster.                                              |

---

## Plugin Dependencies und Peer Dependencies

Die Datei `package.json` des Plugins deklariert Abhängigkeiten.

```json
{
  "name": "@internal/plugin-taskflow-dashboard",
  "version": "0.1.0",
  "dependencies": {
    "@backstage/core-components": "^0.13.0",
    "@backstage/core-plugin-api": "^1.5.0",
    "@material-ui/core": "^4.12.4",
    "@material-ui/icons": "^4.11.3",
    "react": "^18.0.0"
  },
  "peerDependencies": {
    "react": "^18.0.0"
  }
}
```

| Dependency Field   | Zweck                                                                                                                             |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| `dependencies`     | Packages, die dein Plugin zur Laufzeit benötigt.                                                                                  |
| `peerDependencies` | Packages, die von der Backstage App bereitgestellt werden, zum Beispiel React oder Material-UI. Das verhindert Versionskonflikte. |
| `devDependencies`  | Test- und Build-Werkzeuge, die nicht mit dem Plugin ausgeliefert werden.                                                          |

Verwende für React und Material-UI immer Peer Dependencies, um Versionskonflikte zu vermeiden.

---

## Plugin Development Workflow

Ein typischer Workflow für Plugin-Entwicklung sieht so aus:

```bash
yarn backstage-cli create-plugin
yarn install
yarn dev
yarn build
yarn test
yarn lint
```

| Schritt                            | Zweck                                           |
| ---------------------------------- | ----------------------------------------------- |
| `yarn backstage-cli create-plugin` | Erstellt ein neues Plugin.                      |
| `yarn install`                     | Installiert Dependencies.                       |
| `yarn dev`                         | Startet den Entwicklungsserver.                 |
| `yarn build`                       | Baut das Plugin.                                |
| `yarn test`                        | Führt Tests aus.                                |
| `yarn lint`                        | Prüft den Code auf Stil- und Qualitätsprobleme. |

Der Dev Server mit `yarn dev` startet Frontend und Backend mit Hot Reload.

Änderungen am Plugin-Code aktualisieren automatisch den Browser.

Für schnellere Iteration kannst du nur das Frontend starten.

```bash
yarn start
```

Das überspringt den Backend-Start, wenn du nur UI Components änderst.

---

## Plugin Testing

Plugin Components können mit React Testing Library getestet werden.

```tsx
import { renderInTestApp } from '@backstage/test-utils';
import { TaskFlowDashboard } from './TaskFlowDashboard';

describe('TaskFlowDashboard', () => {
  it('renders dashboard title', async () => {
    const rendered = await renderInTestApp(<TaskFlowDashboard />);
    expect(rendered.getByText('TaskFlow Dashboard')).toBeInTheDocument();
  });

  it('fetches and displays metrics', async () => {
    // Mock API response
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({ totalTasks: 42 }),
      })
    );

    const rendered = await renderInTestApp(<TaskFlowDashboard />);
    expect(await rendered.findByText('Total Tasks: 42')).toBeInTheDocument();
  });
});
```

### Best Practices für Tests

* Verwende `renderInTestApp`, damit Backstage Context wie Routing und Theme verfügbar ist.
* Mocke API Calls mit `jest.fn()`.
* Teste Loading States und Fehlerbehandlung.
* Verwende `data-testid` für zuverlässige Elementauswahl.

---

## Ausblick

In dieser Lektion hast du die Backstage Plugin-Architektur kennengelernt.

Du verstehst jetzt:

* Plugin-Typen: Frontend, Backend und Common
* Verzeichnisstruktur eines Plugins
* Plugin-Erstellung mit der CLI
* Routing
* Navigation
* Konfiguration
* API-Integration

Du weißt, wie Plugins Backstage erweitern und wie Plugin-Code sauber strukturiert wird.

Als Nächstes setzt du dieses Wissen um und implementierst das TaskFlow Dashboard Plugin.

Dabei erstellst du React Components für:

* Metrics Cards
* Deployment Status
* Task Statistics

Du rufst Daten aus der TaskFlow API ab, behandelst Loading- und Error-States, verwendest Material-UI für das Layout und integrierst das Plugin in die Backstage Sidebar.

Durch den Aufbau eines vollständigen Plugins festigst du deine React- und Backstage-Kenntnisse.

Das TaskFlow Dashboard zeigt, wie interne Developer Portals Echtzeitdaten aus Anwendungen sichtbar machen und dadurch Entwickler-Workflows verbessern.

---

## Kernaussagen

* Backstage hat drei Plugin-Typen: Frontend für React UI, Backend für Node.js APIs und Common für gemeinsamen TypeScript-Code.
* Plugins werden mit `yarn backstage-cli create-plugin` erstellt. Die CLI erzeugt Verzeichnisstruktur, `package.json` und Beispielcode.
* Plugin Routes werden in `App.tsx` mit `createRoutableExtension` registriert.
* Sidebar Navigation wird in `Root.tsx` ergänzt.
* Plugin Components verwenden Backstage Core Components wie `Page`, `Header` und `Content`.
* Material-UI sorgt für eine konsistente Benutzeroberfläche.
* API-Integration erfolgt über Client Classes mit `fetch`.
* Plugins sollten Fehler behandeln und Loading States anzeigen.

---

## Zusätzliche Ressourcen

| Ressource                                  | Typ           |
| ------------------------------------------ | ------------- |
| Backstage Plugin Development Documentation | Dokumentation |
| Creating a Backstage Plugin Tutorial       | Tutorial      |
| Plugin Architecture Overview               | Dokumentation |
| Backstage Plugin Best Practices            | Artikel       |

---
============================================
# TaskFlow Dashboard Plugin implementieren

Baue ein vollständiges Backstage Plugin für TaskFlow-Metriken und Deployment-Status. Du erstellst React Components mit Material-UI Cards, rufst Daten über `useEffect` und `fetch` aus der TaskFlow API ab, behandelst Loading- und Error-States, zeigst Echtzeitstatistiken zu Tasks an und integrierst das Plugin in die Backstage-Navigation.

---

## Lernmaterial

Du hast React Hooks, die Backstage Plugin-Architektur und die Plugin-Erstellung mit der CLI kennengelernt. Jetzt setzt du alles praktisch um und baust ein echtes Plugin: das TaskFlow Dashboard.

Dieses Plugin zeigt Echtzeitmetriken zur Nutzung der TaskFlow-Anwendung, Task-Statistiken und den Deployment-Status.

Am Ende dieser Lektion hast du ein vollständig funktionsfähiges Backstage Plugin erstellt mit:

* mehreren React Components,
* API-Integration,
* Material-UI Layout,
* Loading States,
* Error Handling,
* Sidebar Navigation.

Diese praktische Umsetzung festigt deine React- und Backstage-Kenntnisse für Platform Engineering.

---

## Anforderungen an das TaskFlow Dashboard

Das TaskFlow Dashboard soll folgende Informationen bereitstellen.

### Metrics Overview

* Gesamtzahl der Tasks im System
* Aktive Benutzer heute
* In dieser Woche abgeschlossene Tasks
* Durchschnittliche Bearbeitungszeit pro Task

### Deployment Status

* Deployment-Status des Frontends: `healthy`, `degraded`, `down`
* Deployment-Status des Backends
* Verbindungsstatus der Datenbank
* Status des Redis Cache

### Task Statistics

* Tasks nach Priorität: `high`, `medium`, `low`
* Tasks nach Status: `open`, `in progress`, `completed`
* Timeline der letzten Task-Aktivitäten

Damit erhalten Platform Engineers und Entwickler einen schnellen Überblick über Zustand und Nutzung von TaskFlow, ohne mehrere Monitoring-Tools öffnen zu müssen.

---

## Plugin erstellen

Erstelle das Plugin mit der Backstage CLI.

```bash
cd backstage-app
yarn backstage-cli create-plugin
```

Dadurch wird `plugins/taskflow-dashboard/` mit einer initialen Struktur generiert.

```text
plugins/taskflow-dashboard/
├── package.json
├── src/
│   ├── plugin.ts
│   ├── components/
│   │   └── ExampleComponent/
│   └── index.ts
```

Installiere anschließend die Dependencies.

```bash
cd plugins/taskflow-dashboard
yarn install
```

---

## Dashboard Layout entwerfen

Das Dashboard verwendet Material-UI Grid für ein responsives Layout mit drei Bereichen.

```tsx
// src/components/TaskFlowDashboard/TaskFlowDashboard.tsx
import React from 'react';
import { Header, Page, Content } from '@backstage/core-components';
import { Grid } from '@material-ui/core';
import { MetricsCards } from './MetricsCards';
import { DeploymentStatus } from './DeploymentStatus';
import { TaskStatistics } from './TaskStatistics';

export const TaskFlowDashboard = () => {
  return (
    <Page themeId="tool">
      <Header
        title="TaskFlow Dashboard"
        subtitle="Application metrics and deployment status"
      />
      <Content>
        <Grid container spacing={3}>
          {/* Top row - Metrics cards */}
          <Grid item xs={12}>
            <MetricsCards />
          </Grid>

          {/* Middle row - Deployment status and statistics */}
          <Grid item xs={12} md={6}>
            <DeploymentStatus />
          </Grid>
          <Grid item xs={12} md={6}>
            <TaskStatistics />
          </Grid>
        </Grid>
      </Content>
    </Page>
  );
};
```

Dieses Layout erzeugt drei Bereiche:

* Metrics Cards über die volle Breite im oberen Bereich
* Deployment Status links
* Task Statistics rechts

Auf Desktop-Bildschirmen teilen sich Deployment Status und Task Statistics die Breite jeweils zur Hälfte. Auf mobilen Geräten werden sie untereinander über die volle Breite dargestellt.

Material-UI Grid behandelt responsive Breakpoints automatisch über `xs`, `sm`, `md`, `lg` und `xl`.

---

## `MetricsCards` Component bauen

Die Metrics Cards zeigen zentrale TaskFlow-Kennzahlen in einer Reihe von Cards an.

```tsx
// src/components/TaskFlowDashboard/MetricsCards.tsx
import React, { useState, useEffect } from 'react';
import { Grid, Card, CardContent, Typography } from '@material-ui/core';
import { makeStyles } from '@material-ui/core/styles';

const useStyles = makeStyles(theme => ({
  card: {
    height: '100%',
  },
  metricValue: {
    fontSize: '2rem',
    fontWeight: 'bold',
    color: theme.palette.primary.main,
  },
}));

interface Metrics {
  totalTasks: number;
  activeUsers: number;
  completedThisWeek: number;
  avgCompletionTime: number;
}

export const MetricsCards = () => {
  const classes = useStyles();
  const [metrics, setMetrics] = useState<Metrics | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchMetrics() {
      try {
        const response = await fetch('https://api.taskflow.dev/api/metrics');
        if (!response.ok) {
          throw new Error('Failed to fetch metrics');
        }
        const data = await response.json();
        setMetrics(data);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setLoading(false);
      }
    }
    fetchMetrics();

    // Refresh metrics every 30 seconds
    const interval = setInterval(fetchMetrics, 30000);
    return () => clearInterval(interval);
  }, []);

  if (loading) {
    return <Typography>Loading metrics...</Typography>;
  }

  if (error) {
    return <Typography color="error">Error: {error}</Typography>;
  }

  if (!metrics) {
    return <Typography>No metrics available</Typography>;
  }

  return (
    <Grid container spacing={2}>
      <Grid item xs={12} sm={6} md={3}>
        <Card className={classes.card}>
          <CardContent>
            <Typography variant="body2" color="textSecondary">
              Total Tasks
            </Typography>
            <Typography className={classes.metricValue}>
              {metrics.totalTasks}
            </Typography>
          </CardContent>
        </Card>
      </Grid>

      <Grid item xs={12} sm={6} md={3}>
        <Card className={classes.card}>
          <CardContent>
            <Typography variant="body2" color="textSecondary">
              Active Users
            </Typography>
            <Typography className={classes.metricValue}>
              {metrics.activeUsers}
            </Typography>
          </CardContent>
        </Card>
      </Grid>

      <Grid item xs={12} sm={6} md={3}>
        <Card className={classes.card}>
          <CardContent>
            <Typography variant="body2" color="textSecondary">
              Completed This Week
            </Typography>
            <Typography className={classes.metricValue}>
              {metrics.completedThisWeek}
            </Typography>
          </CardContent>
        </Card>
      </Grid>

      <Grid item xs={12} sm={6} md={3}>
        <Card className={classes.card}>
          <CardContent>
            <Typography variant="body2" color="textSecondary">
              Avg Completion Time
            </Typography>
            <Typography className={classes.metricValue}>
              {metrics.avgCompletionTime}h
            </Typography>
          </CardContent>
        </Card>
      </Grid>
    </Grid>
  );
};
```

### Wichtige Implementierungsdetails

| Detail              | Umsetzung                                                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| State Management    | Drei `useState` Variablen: Metrikdaten, Loading Flag und Fehlermeldung                                                        |
| Data Fetching       | `useEffect` mit leerem Array `[]` läuft beim Mount und lädt die Metriken                                                      |
| Error Handling      | `try/catch` fängt Fetch-Fehler ab, setzt den Error State und rendert eine Fehlermeldung                                       |
| Loading States      | Zeigt `Loading metrics...` während des Ladens, einen Fehler bei Fehlschlag und Daten bei Erfolg                               |
| Auto-Refresh        | `setInterval` aktualisiert alle 30 Sekunden; die Cleanup Function entfernt das Intervall beim Unmount                         |
| Material-UI Styling | `makeStyles` erzeugt CSS-Klassen; `classes.metricValue` formatiert die Metrikwerte                                            |
| Responsives Grid    | `xs={12} sm={6} md={3}` bedeutet: volle Breite auf Mobile, zwei Cards pro Reihe auf Tablets, vier Cards pro Reihe auf Desktop |

---

## `DeploymentStatus` Component bauen

Die Deployment Status Component zeigt den Zustand jeder TaskFlow-Komponente an.

```tsx
// src/components/TaskFlowDashboard/DeploymentStatus.tsx
import React, { useState, useEffect } from 'react';
import {
  Card,
  CardContent,
  CardHeader,
  List,
  ListItem,
  ListItemText,
  Chip,
} from '@material-ui/core';

interface ComponentStatus {
  name: string;
  status: 'healthy' | 'degraded' | 'down';
  version: string;
  lastDeployed: string;
}

interface DeploymentData {
  components: ComponentStatus[];
}

export const DeploymentStatus = () => {
  const [deployment, setDeployment] = useState<DeploymentData | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function fetchDeployment() {
      try {
        const response = await fetch('https://api.taskflow.dev/api/deployment-status');
        if (!response.ok) {
          throw new Error('Failed to fetch deployment status');
        }
        const data = await response.json();
        setDeployment(data);
      } catch (err) {
        console.error('Deployment fetch error:', err);
      } finally {
        setLoading(false);
      }
    }
    fetchDeployment();

    const interval = setInterval(fetchDeployment, 60000); // Refresh every minute
    return () => clearInterval(interval);
  }, []);

  const getStatusColor = (status: string) => {
    switch (status) {
      case 'healthy':
        return 'primary';
      case 'degraded':
        return 'default';
      case 'down':
        return 'secondary';
      default:
        return 'default';
    }
  };

  if (loading) {
    return <Card><CardContent>Loading deployment status...</CardContent></Card>;
  }

  return (
    <Card>
      <CardHeader title="Deployment Status" />
      <CardContent>
        <List>
          {deployment?.components.map(component => (
            <ListItem key={component.name}>
              <ListItemText
                primary={component.name}
                secondary={`Version ${component.version} • Deployed ${component.lastDeployed}`}
              />
              <Chip
                label={component.status}
                color={getStatusColor(component.status)}
                size="small"
              />
            </ListItem>
          ))}
        </List>
      </CardContent>
    </Card>
  );
};
```

### Implementierungsschwerpunkte

| Element          | Beschreibung                                                                |
| ---------------- | --------------------------------------------------------------------------- |
| Status Indicator | `Chip` zeigt den Status farblich markiert an: `healthy`, `degraded`, `down` |
| Helper Function  | `getStatusColor` ordnet Statuswerte Material-UI Farben zu                   |
| List Layout      | `List` und `ListItem` sorgen für klare Zeilen je Component                  |
| Metadata Display | Version und Deployment-Zeitpunkt werden pro Component angezeigt             |
| Refresh Interval | Aktualisierung jede Minute mit `60000` Millisekunden                        |

---

## `TaskStatistics` Component bauen

Die Task Statistics Component zeigt Aufschlüsselungen nach Priorität und Status.

```tsx
// src/components/TaskFlowDashboard/TaskStatistics.tsx
import React, { useState, useEffect } from 'react';
import {
  Card,
  CardContent,
  CardHeader,
  Typography,
  Box,
} from '@material-ui/core';
import { makeStyles } from '@material-ui/core/styles';

const useStyles = makeStyles(theme => ({
  statRow: {
    display: 'flex',
    justifyContent: 'space-between',
    marginBottom: theme.spacing(2),
  },
  statLabel: {
    fontWeight: 'bold',
  },
}));

interface Statistics {
  byPriority: {
    high: number;
    medium: number;
    low: number;
  };
  byStatus: {
    open: number;
    inProgress: number;
    completed: number;
  };
}

export const TaskStatistics = () => {
  const classes = useStyles();
  const [stats, setStats] = useState<Statistics | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function fetchStats() {
      try {
        const response = await fetch('https://api.taskflow.dev/api/task-statistics');
        if (!response.ok) {
          throw new Error('Failed to fetch statistics');
        }
        const data = await response.json();
        setStats(data);
      } catch (err) {
        console.error('Statistics fetch error:', err);
      } finally {
        setLoading(false);
      }
    }
    fetchStats();

    const interval = setInterval(fetchStats, 30000);
    return () => clearInterval(interval);
  }, []);

  if (loading) {
    return <Card><CardContent>Loading statistics...</CardContent></Card>;
  }

  if (!stats) {
    return <Card><CardContent>No statistics available</CardContent></Card>;
  }

  return (
    <Card>
      <CardHeader title="Task Statistics" />
      <CardContent>
        <Typography variant="h6" gutterBottom>By Priority</Typography>
        <Box className={classes.statRow}>
          <Typography className={classes.statLabel}>High Priority:</Typography>
          <Typography>{stats.byPriority.high}</Typography>
        </Box>
        <Box className={classes.statRow}>
          <Typography className={classes.statLabel}>Medium Priority:</Typography>
          <Typography>{stats.byPriority.medium}</Typography>
        </Box>
        <Box className={classes.statRow}>
          <Typography className={classes.statLabel}>Low Priority:</Typography>
          <Typography>{stats.byPriority.low}</Typography>
        </Box>

        <Typography variant="h6" gutterBottom style={{ marginTop: 16 }}>
          By Status
        </Typography>
        <Box className={classes.statRow}>
          <Typography className={classes.statLabel}>Open:</Typography>
          <Typography>{stats.byStatus.open}</Typography>
        </Box>
        <Box className={classes.statRow}>
          <Typography className={classes.statLabel}>In Progress:</Typography>
          <Typography>{stats.byStatus.inProgress}</Typography>
        </Box>
        <Box className={classes.statRow}>
          <Typography className={classes.statLabel}>Completed:</Typography>
          <Typography>{stats.byStatus.completed}</Typography>
        </Box>
      </CardContent>
    </Card>
  );
};
```

### Hinweise zur Implementierung

| Element                | Beschreibung                                                               |
| ---------------------- | -------------------------------------------------------------------------- |
| Gruppierte Statistiken | Zwei Abschnitte: nach Priorität und nach Status                            |
| Custom Layout          | `Box` mit Flexbox für Label-Wert-Zeilen                                    |
| Typography Variants    | `h6` für Abschnittsüberschriften, normale Typography für Werte             |
| Spacing                | Material-UI Spacing über `theme.spacing(2)` sorgt für konsistente Abstände |

---

## Plugin in Backstage integrieren

Registriere die Plugin Route in `packages/app/src/App.tsx`.

```tsx
import { TaskFlowDashboardPage } from '@internal/plugin-taskflow-dashboard';

const routes = (
  <FlatRoutes>
    <Route path="/catalog" element={<CatalogIndexPage />} />
    <Route path="/docs" element={<TechDocsIndexPage />} />
    <Route path="/create" element={<ScaffolderPage />} />

    {/* TaskFlow Dashboard */}
    <Route path="/taskflow" element={<TaskFlowDashboardPage />} />
  </FlatRoutes>
);
```

Füge die Sidebar Navigation in `packages/app/src/components/Root/Root.tsx` hinzu.

```tsx
import DashboardIcon from '@material-ui/icons/Dashboard';

export const Root = ({ children }: PropsWithChildren<{}>) => (
  <SidebarPage>
    <Sidebar>
      <SidebarLogo />
      <SidebarDivider />
      <SidebarItem icon={HomeIcon} to="/" text="Home" />
      <SidebarItem icon={CategoryIcon} to="catalog" text="Catalog" />
      <SidebarItem icon={CreateComponentIcon} to="create" text="Create" />
      <SidebarItem icon={DocsIcon} to="docs" text="Docs" />

      {/* TaskFlow Dashboard */}
      <SidebarDivider />
      <SidebarItem icon={DashboardIcon} to="taskflow" text="TaskFlow" />
    </Sidebar>
    {children}
  </SidebarPage>
);
```

Danach erscheint `TaskFlow` in der Sidebar.

Ein Klick öffnet das Dashboard mit allen drei Komponenten.

---

## Testen und Debugging

Teste das Plugin lokal.

```bash
cd backstage-app
yarn dev
```

Öffne anschließend:

```text
http://localhost:3000/taskflow
```

### Häufige Probleme

| Problem                        | Was du prüfen solltest                                                                                                  |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `404` Fehler bei API Calls     | Prüfe die API URL in der Konfiguration. Ergänze CORS Header im TaskFlow Backend für lokale Entwicklung.                 |
| Component rendert nicht        | Prüfe die Browser Console auf React-Fehler. Stelle sicher, dass alle Imports korrekt sind.                              |
| Styling-Probleme               | Nutze DevTools. Prüfe, ob `makeStyles` Klassennamen angewendet werden und das Material-UI Theme korrekt geladen ist.    |
| Daten aktualisieren sich nicht | Prüfe `useEffect` Dependency Arrays und stelle sicher, dass Cleanup Functions für Intervalle korrekt ausgeführt werden. |
| TypeScript-Fehler              | Stelle sicher, dass TypeScript Interfaces zur tatsächlichen API Response passen. Ergänze bei Bedarf Type Assertions.    |

Nutze die React DevTools Browser Extension, um Component State, Props und Render-Zyklen zu untersuchen.

---

## Plugin erweitern

Sobald die Grundfunktionalität läuft, kannst du das Plugin weiter verbessern.

### Error Boundaries

Component-Fehler können sauber abgefangen werden.

```tsx
import { ErrorBoundary } from '@backstage/core-components';

<ErrorBoundary>
  <TaskFlowDashboard />
</ErrorBoundary>
```

### Skeleton Loaders

Während des Ladens kann eine Platzhalter-UI angezeigt werden.

```tsx
import { Skeleton } from '@material-ui/lab';

{loading && <Skeleton variant="rect" height={100} />}
```

### Charts und Graphen

Für Visualisierungen kannst du Recharts verwenden.

```tsx
import { LineChart, Line, XAxis, YAxis } from 'recharts';

<LineChart data={taskHistory}>
  <Line type="monotone" dataKey="tasks" stroke="#8884d8" />
  <XAxis dataKey="date" />
  <YAxis />
</LineChart>
```

### Echtzeit-Updates

Nutze WebSockets für Live-Daten statt Polling.

### Filter und Suche

Ergänze Controls, um Metriken nach Zeitraum oder Component zu filtern.

---

## Ausblick

In dieser Lektion hast du ein vollständiges Backstage Plugin von Grund auf gebaut.

Du hast:

* React Components mit Hooks erstellt,
* `useState` und `useEffect` eingesetzt,
* Daten aus APIs geladen,
* Loading States umgesetzt,
* Error States behandelt,
* Material-UI für Layout und Styling verwendet,
* das Plugin in die Backstage Navigation integriert.

Das TaskFlow Dashboard zeigt, wie interne Developer Portals Metriken und Deployment-Status aus Anwendungen sichtbar machen. Dadurch verbessert sich die Transparenz über den Systemzustand, und Entwickler verbringen weniger Zeit damit, Informationen über mehrere Tools hinweg zu suchen.

Als Nächstes geht es um Plattformautomatisierung mit Backstage Software Templates und dem Scaffolder.

Du lernst, wie Templates:

* Projekterstellung automatisieren,
* Code generieren,
* GitHub Repositories erstellen,
* CI/CD-Pipelines einrichten,
* neue Components im Catalog registrieren.

Danach integrierst du Backstage mit GitHub, Kubernetes und Monitoring-Tools, um ein vollständiges Developer Portal für TaskFlow aufzubauen.

Am Ende dieser Woche hast du ein funktionsfähiges internes Developer Portal mit Custom Plugins und Self-Service-Automatisierung.

---

## Kernaussagen

* Das Dashboard Layout verwendet Material-UI Grid mit responsiven Breakpoints wie `xs`, `sm` und `md` für Mobile-First Design.
* Die `MetricsCards` Component lädt Daten mit `useEffect`, verwaltet State mit `useState`, behandelt Fehler mit `try/catch` und aktualisiert Daten automatisch mit `setInterval`.
* Die `DeploymentStatus` Component verwendet Material-UI `List` und `Chip` für Statusanzeigen mit farblich markierter Health-Darstellung.
* Plugin Routes werden in `App.tsx` mit `FlatRoutes` registriert.
* Sidebar Navigation wird in `Root.tsx` mit `SidebarItem` ergänzt.
* Plugins werden lokal mit `yarn dev` getestet.
* Debugging erfolgt mit React DevTools, API-Fehlerbehandlung, Error Boundaries und Skeleton Loaders.

---

## Zusätzliche Ressourcen

| Ressource                      | Typ           |
| ------------------------------ | ------------- |
| Material-UI Card Component     | Dokumentation |
| Material-UI Grid Layout        | Dokumentation |
| React useEffect Complete Guide | Artikel       |
| Backstage Plugin Tutorial      | Tutorial      |

---

## Private Lesson Notes

Private Notizen während des Lernens erfassen.

---

=================================================

# Backstage Plugin mit React bauen
=================================================


Baue dein erstes React Plugin für Backstage. Erstelle ein TaskFlow Dashboard, das Live-Metriken mit React Hooks, Material-UI Components und API-Integration anzeigt.

Du vertiefst `useState`, `useEffect` und TypeScript, während du produktionsnahen React-Code entwickelst.

Lerne React durch praktische Umsetzung.

---

## Lab-Übersicht

| Eigenschaft        | Wert            |
| ------------------ | --------------- |
| Schwierigkeitsgrad | Fortgeschritten |
| Dauer              | 60 Minuten      |
| Typ                | Lab             |

---

## Voraussetzungen

Für dieses Lab benötigst du:

* `Week 12 Lab 1: Backstage Setup & Service Catalog completed`
* Abgeschlossene Lektion zum Vergleich von React und Vue.js
* Laufende TaskFlow-Anwendung mit API Endpoints
* TypeScript-Grundlagen
* Einführung in die Material-UI Component Library

---

## Über dieses Lab

In diesem Lab baust du ein eigenes Backstage Plugin mit React. Das Plugin zeigt TaskFlow-Metriken direkt im Developer Portal an.

Du vertiefst dabei zentrale React- und Backstage-Konzepte:

* React Hooks wie `useState` und `useEffect`
* Component-Entwicklung mit TypeScript
* Material-UI Components
* API-Integrationsmuster
* Loading States
* Error Handling
* Routing und Navigation in Backstage

Im Mittelpunkt steht das Plugin `TaskFlow Dashboard`. Es zeigt Live-Metriken wie Task-Anzahlen, aktive Benutzer und Deployment-Status mit sauberem, produktionsnahem React-Code.

Du verwendest die Backstage CLI, um ein Plugin zu generieren. Danach schreibst du React Components mit Hooks, rufst Daten aus der TaskFlow API ab, behandelst Lade- und Fehlerzustände, gestaltest die Oberfläche mit Material-UI Cards und Grids und integrierst das Plugin in das Navigation- und Routing-System von Backstage.

---

## Was du lernst

Dieses Lab besteht aus 5 Aufgaben.

| Nr. | Aufgabe                                              | Ziel                                                                                                                                                          |
| --: | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|   1 | Backstage erkunden und dein erstes Plugin scaffolden | Die laufende Backstage-Instanz erkunden, mit der Backstage CLI dein erstes Plugin erzeugen und es in das App Routing einbinden                                |
|   2 | React Component mit `useState` Hook erstellen        | Eine TypeScript React Component für TaskFlow-Metriken erstellen und mit `useState` Metriken, Loading State und Error State verwalten                          |
|   3 | API-Daten mit `useEffect` Hook abrufen               | Mit `useEffect` beim Mount der Component Live-Metriken aus der TaskFlow API abrufen und Loading- sowie Error-States behandeln                                 |
|   4 | Metriken mit Material-UI Components anzeigen         | Das TaskFlow Dashboard mit Material-UI Cards, Grid Layouts, `CircularProgress` Spinnern und `Alert` Components als produktionsnahe UI gestalten               |
|   5 | Plugin integrieren und testen                        | Die `TaskFlowMetrics` Component exportieren, einen Sidebar-Navigationslink hinzufügen, den Development Server starten und das Live Plugin in Backstage testen |

---

## Systemanforderungen

Für dieses Lab benötigst du:

* einen modernen Webbrowser,
* eine stabile Internetverbindung,
* ungefähr 60 Minuten konzentrierte Arbeitszeit,
* Kenntnisse aus `Week 12 Lab 1: Backstage Setup & Service Catalog completed`,
* abgeschlossene React-vs.-Vue.js-Vergleichslektion,
* eine laufende TaskFlow-Anwendung.



==========================================
# 3 Platform Self-Service & Integration
==========================================


# Backstage Software Templates und Scaffolder

Beherrsche Backstage Software Templates, um die Erstellung neuer Projekte zu automatisieren. Du lernst die Template-Struktur mit `template.yaml`, `skeleton/` und Parametern, die wichtigsten Template Actions wie `fetch:template`, `publish:github` und `catalog:register`, und erstellst ein Template `New FastAPI Microservice` für das TaskFlow-Ökosystem.

Damit ermöglichst du echte Self-Service-Workflows für Entwickler.

---

## Lernmaterial

Der Software Scaffolder ist eine der leistungsstärksten Backstage-Funktionen für Platform Engineering.

Er automatisiert die Projekterstellung durch:

* Codegenerierung
* Repository-Erstellung
* Einrichtung von CI/CD-Pipelines
* Registrierung neuer Components im Catalog

Alles wird über eine formularbasierte Oberfläche ausgelöst.

Statt Boilerplate-Code zu kopieren, Templates in Confluence zu suchen oder bestehende Projekte manuell nachzubauen, füllen Entwickler ein Formular aus. Danach erhalten sie ein produktionsnahes Projekt, das den Standards der Organisation entspricht.

Das reduziert die Time to First Commit erheblich und sorgt für Konsistenz über Teams hinweg.

In dieser Lektion lernst du:

* Template-Struktur
* Parameter
* Actions
* Aufbau eines Templates `New FastAPI Microservice`
* Self-Service-Workflows für das TaskFlow-Ökosystem

Am Ende verstehst du, wie du Workflows baust, die Entwicklerproduktivität skalierbar verbessern.

---

## Den Scaffolder verstehen

> **Abbildung:** Backstage Software Scaffolder Workflow — Ein Entwickler füllt ein Formular aus. Der Scaffolder verarbeitet das Template und führt Actions aus, um ein GitHub Repository und einen Catalog-Eintrag zu erstellen.

Der Backstage Scaffolder ist eine Template Engine mit mehreren Aufgaben.

| Funktion              | Beschreibung                                                                                                                           |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Formulare anzeigen    | Entwickler geben Projektname, Beschreibung, Owner und Technologieauswahl ein.                                                          |
| Templates verarbeiten | Template-Dateien werden mit Variablen ersetzt, zum Beispiel `${{ parameters.projectName }}`.                                           |
| Actions ausführen     | Vordefinierte Schritte werden ausgeführt, zum Beispiel Dateien laden, nach GitHub veröffentlichen oder Kubernetes-Manifeste erstellen. |
| Entities registrieren | Neue Components werden automatisch im Backstage Catalog registriert.                                                                   |

---

## Scaffolder Workflow

Der typische Ablauf sieht so aus:

1. Der Entwickler öffnet den Bereich **Create** in Backstage.
2. Er wählt ein Template, zum Beispiel `New FastAPI Microservice`.
3. Er füllt das Formular aus: Projektname, Beschreibung, Datenbankauswahl.
4. Er klickt auf **Create**.
5. Der Scaffolder generiert Code, erstellt ein GitHub Repository und richtet CI/CD ein.
6. Die neue Component erscheint mit vollständigen Metadaten im Catalog.

Dieser Self-Service-Ansatz reduziert die Frage „Wie erstelle ich einen neuen Service?“ von wochenlanger Einrichtung auf wenige Minuten.

---

## Struktur eines Software Templates

Templates sind Catalog Entities vom Typ `Template`.

```text
templates/fastapi-microservice/
├── template.yaml          # Template definition
└── skeleton/              # Template files
    ├── src/
    │   └── main.py
    ├── Dockerfile
    ├── kubernetes/
    │   └── deployment.yaml
    ├── .github/
    │   └── workflows/
    │       └── ci.yml
    └── catalog-info.yaml
```

### `template.yaml`

Diese Datei definiert:

* Parameter als Formularfelder
* Steps als auszuführende Actions
* Output-Links nach erfolgreicher Ausführung

### `skeleton/`

Dieses Verzeichnis enthält Template-Dateien mit Variablenersetzung.

Die Dateien werden verarbeitet und anschließend in das neue Repository kopiert.

---

## Template Definition: `template.yaml`

Das folgende Beispiel zeigt ein vollständiges Template zur Erstellung von FastAPI-Microservices.

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: fastapi-microservice
  title: New FastAPI Microservice
  description: Create a new FastAPI microservice with PostgreSQL, Docker, Kubernetes, and GitHub Actions
  tags:
    - python
    - fastapi
    - microservice
spec:
  owner: platform-team
  type: service

  parameters:
    - title: Service Information
      required:
        - name
        - description
      properties:
        name:
          title: Service Name
          type: string
          description: Unique name for the service (lowercase, hyphens)
          pattern: '^[a-z0-9-]+$'
          ui:autofocus: true
        description:
          title: Description
          type: string
          description: What does this service do?
        owner:
          title: Owner
          type: string
          description: Team responsible for this service
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group

    - title: Technology Choices
      required:
        - database
      properties:
        database:
          title: Database
          type: string
          enum:
            - postgresql
            - mysql
            - none
          default: postgresql
        cache:
          title: Enable Redis Cache
          type: boolean
          default: false

    - title: Repository Configuration
      required:
        - repoUrl
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com

  steps:
    - id: fetch-base
      name: Fetch Base Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          database: ${{ parameters.database }}
          cache: ${{ parameters.cache }}

    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts: ['github.com']
        description: ${{ parameters.description }}
        repoUrl: ${{ parameters.repoUrl }}
        repoVisibility: private
        defaultBranch: main

    - id: register
      name: Register Component
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'

  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in Catalog
        icon: catalog
        entityRef: ${{ steps.register.output.entityRef }}
```

---

## Template Parameters

Parameter definieren die Formularfelder, die Entwickler ausfüllen.

| Konzept                | Details                                                                                               |
| ---------------------- | ----------------------------------------------------------------------------------------------------- |
| Mehrere Formularseiten | Jeder Eintrag im Array `parameters` ist eine eigene Formularseite. `title` trennt die Seiten visuell. |
| Feldtypen              | Unterstützt werden unter anderem `string`, `boolean`, `number`, `array` und `object`.                 |
| Validierung            | Pflichtfelder, Regex-Patterns und `enum`-Werte für Dropdowns.                                         |
| UI-Hinweise            | `ui:autofocus` und `ui:field` für spezielle Widgets wie `OwnerPicker` oder `RepoUrlPicker`.           |

### Beispiel für ein Formularfeld

```yaml
name:
  title: Service Name
  type: string
  description: Unique name for the service (lowercase, hyphens)
  pattern: '^[a-z0-9-]+$'
  ui:autofocus: true
```

Dieses Feld erzeugt ein Texteingabefeld mit Regex-Validierung und Beschreibung.

---

## Spezielle UI-Felder

### `OwnerPicker`

Der `OwnerPicker` zeigt Teams oder Benutzer aus dem Catalog an.

```yaml
owner:
  title: Owner
  type: string
  ui:field: OwnerPicker
  ui:options:
    allowedKinds:
      - Group
```

### `RepoUrlPicker`

Der `RepoUrlPicker` hilft bei der Auswahl eines GitHub Repository-Ziels.

```yaml
repoUrl:
  title: Repository Location
  type: string
  ui:field: RepoUrlPicker
  ui:options:
    allowedHosts:
      - github.com
```

| UI Field            | Zweck                                            |
| ------------------- | ------------------------------------------------ |
| `OwnerPicker`       | Dropdown für Teams oder Benutzer aus dem Catalog |
| `RepoUrlPicker`     | GitHub Repository Picker mit erlaubten Hosts     |
| `EntityPicker`      | Auswahl einer einzelnen Catalog Entity           |
| `MultiEntityPicker` | Auswahl mehrerer Catalog Entities                |

Diese Picker nutzen Live-Daten aus Backstage und verbessern dadurch die Benutzererfahrung.

---

## Template Actions

Actions sind Schritte, die während der Projekterstellung ausgeführt werden.

### `fetch:template`

Kopiert Template-Dateien und ersetzt Variablen.

```yaml
- id: fetch-base
  name: Fetch Base Template
  action: fetch:template
  input:
    url: ./skeleton
    values:
      name: ${{ parameters.name }}
      description: ${{ parameters.description }}
```

### `publish:github`

Erstellt ein GitHub Repository und pusht den generierten Code.

```yaml
- id: publish
  name: Publish to GitHub
  action: publish:github
  input:
    allowedHosts: ['github.com']
    description: ${{ parameters.description }}
    repoUrl: ${{ parameters.repoUrl }}
    repoVisibility: private
    defaultBranch: main
```

### `catalog:register`

Registriert die neue Component im Backstage Catalog.

```yaml
- id: register
  name: Register Component
  action: catalog:register
  input:
    repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
    catalogInfoPath: '/catalog-info.yaml'
```

### Weitere nützliche Actions

| Action          | Zweck                                       |
| --------------- | ------------------------------------------- |
| `fetch:plain`   | Dateien ohne Variablenersetzung kopieren    |
| `fs:rename`     | Dateien umbenennen                          |
| `fs:delete`     | Dateien löschen                             |
| `debug:log`     | Nachrichten für Template-Debugging ausgeben |
| `catalog:write` | Eine Catalog Entity Datei schreiben         |

Actions werden nacheinander ausgeführt.

Spätere Actions können Outputs früherer Actions referenzieren, zum Beispiel mit `${{ steps.stepId.output.fieldName }}`.

---

## Variablenersetzung in Template-Dateien

Dateien im Verzeichnis `skeleton/` verwenden Nunjucks-Syntax für Variablenersetzung.

### `skeleton/catalog-info.yaml`

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.name }}
  title: ${{ values.name | capitalize }}
  description: ${{ values.description }}
  annotations:
    github.com/project-slug: ${{ values.repoUrl | parseRepoUrl | pick('owner') }}/${{ values.name }}
  tags:
    - python
    - fastapi
    {% if values.database != 'none' %}
    - ${{ values.database }}
    {% endif %}
spec:
  type: service
  lifecycle: experimental
  owner: ${{ values.owner }}
  {% if values.database != 'none' %}
  dependsOn:
    - resource:${{ values.name }}-database
  {% endif %}
```

### Häufige Substitution Patterns

| Pattern                                            | Bedeutung                                                       |
| -------------------------------------------------- | --------------------------------------------------------------- |
| `${{ values.name }}`                               | Wird durch den Parameterwert ersetzt                            |
| `${{ values.name \| capitalize }}`                 | Wendet einen Filter an, hier Großschreibung des ersten Zeichens |
| `{% if values.database != 'none' %}...{% endif %}` | Bedingter Block                                                 |
| `{% for item in values.items %}...{% endfor %}`    | Schleife                                                        |

### `skeleton/src/main.py`

```python
from fastapi import FastAPI

app = FastAPI(
    title="${{ values.name }}",
    description="${{ values.description }}",
)

@app.get("/")
def read_root():
    return {"service": "${{ values.name }}", "status": "healthy"}

{% if values.database == 'postgresql' %}
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/dbname"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()
{% endif %}
```

Dadurch wird je nach Datenbankauswahl unterschiedlicher Code generiert.

---

## Template `New FastAPI Microservice` erstellen

Jetzt bauen wir ein vollständiges Template für TaskFlow-kompatible Microservices.

### `skeleton/Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### `skeleton/kubernetes/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${{ values.name }}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${{ values.name }}
  template:
    metadata:
      labels:
        app: ${{ values.name }}
    spec:
      containers:
      - name: ${{ values.name }}
        image: ${{ values.name }}:latest
        ports:
        - containerPort: 8000
        env:
        {% if values.database == 'postgresql' %}
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: ${{ values.name }}-db-secret
              key: url
        {% endif %}
```

### `skeleton/.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: docker build -t ${{ values.name }}:latest .
```

Diese Skeleton-Dateien erzeugen einen produktionsnahen Microservice mit:

* FastAPI
* Docker
* Kubernetes
* GitHub Actions CI/CD

---

## Template registrieren

Das Template wird über einen Catalog-Eintrag in Backstage registriert.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Location
metadata:
  name: fastapi-templates
  description: FastAPI microservice templates
spec:
  type: url
  targets:
    - https://github.com/tekanaid/backstage-templates/blob/main/fastapi-microservice/template.yaml
```

Alternativ kann die Registrierung direkt in `app-config.yaml` erfolgen.

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/tekanaid/backstage-templates/blob/main/fastapi-microservice/template.yaml
```

Danach erscheint das Template im Bereich **Create** unter `New FastAPI Microservice`.

---

## Templates testen und validieren

Templates sollten gründlich getestet werden, bevor sie für Entwickler freigegeben werden.

### 1. Dry Run

Template-Verarbeitung testen, ohne ein Repository zu veröffentlichen.

```bash
yarn backstage-cli repo test-templates --template-dir ./templates/fastapi-microservice
```

### 2. Lokaler Test

Scaffolder in der Backstage-Entwicklungsumgebung ausführen.

```bash
yarn dev
```

### 3. Validierungs-Checkliste

Prüfe vor dem Rollout:

* Alle Parameter haben Beschreibungen und Validierung.
* Variablenersetzung funktioniert korrekt.
* Generierter Code besteht Linting und Tests.
* Kubernetes-Manifeste sind gültig.
* CI/CD-Pipelines laufen erfolgreich.
* `catalog-info.yaml` enthält korrekte Metadaten.

### Häufige Probleme

| Problem                      | Ursache                                                  |
| ---------------------------- | -------------------------------------------------------- |
| Fehlende Required Parameters | Template-Ausführung bricht ab                            |
| Tippfehler in Variablennamen | Literal wie `${{ values.typo }}` bleibt im Output stehen |
| Ungültige YAML-Syntax        | Template Parsing schlägt fehl                            |
| GitHub Permission Errors     | Repository kann nicht erstellt werden                    |

Nutze Backstage Logs und Browser DevTools, um Template-Probleme zu debuggen.

---

## Erweiterte Template-Funktionen

### Bedingte Steps

Actions können abhängig von Parametern ausgeführt werden.

```yaml
steps:
  - id: create-database
    name: Create Database Resource
    if: ${{ parameters.database != 'none' }}
    action: catalog:write
    input:
      entity:
        apiVersion: backstage.io/v1alpha1
        kind: Resource
        metadata:
          name: ${{ parameters.name }}-database
```

### Dynamische Werte

Werte können aus anderen Parametern abgeleitet werden.

```yaml
properties:
  serviceName:
    title: Service Name
    type: string
  repoName:
    title: Repository Name
    type: string
    default: ${{ parameters.serviceName }}-service
```

### Template Composition

Templates können gemeinsame Dateien aus anderen Skeletons einbinden.

```yaml
- id: fetch-common
  name: Fetch Common Files
  action: fetch:template
  input:
    url: ../../common/skeleton
```

### Custom Actions

Für organisationsspezifische Logik kannst du eigene Actions in TypeScript schreiben.

Beispiele:

* Jira Tickets erstellen
* Confluence aktualisieren
* AWS-Ressourcen provisionieren
* interne Freigabeprozesse auslösen

---

## Best Practices für Templates

| Practice                  | Warum es wichtig ist                                                                                |
| ------------------------- | --------------------------------------------------------------------------------------------------- |
| Skeleton minimal halten   | Nur essenzielle Dateien einfügen. Zu viele Optionen machen Templates schwer wartbar.                |
| Sinnvolle Defaults setzen | Häufige Entscheidungen vorbefüllen, zum Beispiel PostgreSQL als Standarddatenbank.                  |
| Beispiele bereitstellen   | Generierte Projekte sollten Beispielcode und Dokumentation enthalten. Das erleichtert den Einstieg. |
| Eingaben validieren       | Regex Patterns, Enums und Pflichtfelder verhindern ungültige Projektnamen oder Konfigurationen.     |
| Parameter dokumentieren   | Klare Feldbeschreibungen helfen Entwicklern zu verstehen, was jede Eingabe bewirkt.                 |
| Templates versionieren    | Git Tags ermöglichen stabile Template-Versionen und sichere Weiterentwicklung.                      |
| Gründlich testen          | Jede Template-Änderung sollte Ende-zu-Ende getestet werden, bevor sie in Produktion geht.           |

---

## Ausblick

In dieser Lektion hast du Backstage Software Templates und den Scaffolder zur Automatisierung der Projekterstellung kennengelernt.

Du verstehst jetzt:

* die Template-Struktur mit `template.yaml` und `skeleton/`,
* Parameter für Formularfelder,
* Actions zur Ausführung von Schritten,
* `fetch:template`,
* `publish:github`,
* `catalog:register`,
* den Aufbau eines Templates `New FastAPI Microservice` für das TaskFlow-Ökosystem.

Als Nächstes geht es um Plattformintegration.

Du verbindest Backstage mit:

* GitHub
* Kubernetes
* Monitoring-Tools
* Dokumentationssystemen

Außerdem lernst du:

* Authentifizierung und Autorisierung
* TechDocs für Documentation as Code
* Aufbau eines vollständigen Developer Portals für TaskFlow
* Messung von Verbesserungen der Developer Experience

Wenn du Custom Plugins, Software Templates und externe Integrationen kombinierst, entsteht ein vollständiges internes Developer Portal. Es verbessert Entwicklerproduktivität und Platform-Engineering-Workflows deutlich.

---

## Kernaussagen

* Der Software Scaffolder zeigt Formulare an, verarbeitet Templates mit Variablenersetzung, führt Actions aus und registriert Catalog Entities automatisch.
* `template.yaml` definiert Parameter, Steps und Output-Links.
* Parameter unterstützen Validierung und UI-Felder wie `OwnerPicker`.
* Häufige Actions sind `fetch:template`, `publish:github` und `catalog:register`.
* Skeleton-Dateien verwenden Nunjucks-Syntax für Variablen wie `${{ values.name }}`, Bedingungen mit `{% if %}` und Filter wie `| capitalize`.
* Templates sollten mit `yarn backstage-cli repo test-templates` getestet werden.
* Generierter Code, Kubernetes-Manifeste und CI/CD-Pipelines müssen vor dem Rollout validiert werden.

---

## Zusätzliche Ressourcen

| Ressource                                  | Typ           |
| ------------------------------------------ | ------------- |
| Backstage Software Templates Documentation | Dokumentation |
| Creating Software Templates Tutorial       | Tutorial      |
| Built-in Template Actions Reference        | Dokumentation |
| Nunjucks Templating Documentation          | Dokumentation |

---



## Backstage mit GitHub integrieren

Die GitHub-Integration ermöglicht:

* Repository-Informationen auf Catalog-Seiten anzeigen
* Aktuelle Commits und Pull Requests anzeigen
* GitHub Actions Workflows verlinken
* Repository-Contributors anzeigen
* Issues und Pull Requests aus Backstage erstellen

### Konfiguration in `app-config.yaml`

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

---

## GitHub App erstellen

Für Produktionsumgebungen wird eine GitHub App empfohlen.

Statt Personal Access Tokens verwendest du eine GitHub App. Das verbessert Sicherheit, Berechtigungsmanagement und Team-Verwaltung.

### Vorgehen

1. Öffne GitHub Settings → Developer Settings → GitHub Apps.
2. Klicke auf **New GitHub App**.
3. Konfiguriere die App:

| Einstellung         | Wert                             |
| ------------------- | -------------------------------- |
| Name                | `TaskFlow Backstage Portal`      |
| Homepage URL        | `https://backstage.taskflow.dev` |
| Webhook             | Deaktiviert, vorerst             |
| Repository contents | Read                             |
| Pull requests       | Read & Write                     |
| Metadata            | Read                             |
| Workflows           | Read                             |

4. Generiere den Private Key und lade ihn herunter.
5. Installiere die App in deiner Organisation.

### `app-config.yaml` mit GitHub App aktualisieren

```yaml
integrations:
  github:
    - host: github.com
      apps:
        - appId: ${GITHUB_APP_ID}
          clientId: ${GITHUB_APP_CLIENT_ID}
          clientSecret: ${GITHUB_APP_CLIENT_SECRET}
          webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}
          privateKey: ${GITHUB_APP_PRIVATE_KEY}
```

Damit kann Backstage auf Repositories zugreifen, Pull Requests erstellen und Workflows im Namen der App auslösen.

---

## GitHub-Annotationen im Catalog

Components verwenden Annotationen, um eine Verbindung zu GitHub herzustellen.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    github.com/project-slug: tekanaid/taskflow-backend
    github.com/workflows: ci.yml,deploy.yml
spec:
  type: service
  owner: platform-team
```

Mit diesen Annotationen zeigt Backstage auf jeder Component-Seite relevante GitHub-Informationen an.

| Backstage-Anzeige                       | Quelle                  |
| --------------------------------------- | ----------------------- |
| Repository README                       | GitHub repo root        |
| Zeitachse der letzten Commits           | GitHub Commits API      |
| Offene und gemergte Pull Requests       | GitHub PR API           |
| GitHub Actions Workflow Runs und Status | GitHub Actions API      |
| Contributors und Code Owners            | GitHub Contributors API |

Entwickler wechseln nur dann zu GitHub, wenn sie Details benötigen. Die Übersicht erhalten sie direkt in Backstage.

---

## Backstage mit Kubernetes integrieren

Das Kubernetes Plugin zeigt Cluster-Informationen direkt in Backstage an.

### Plugin installieren

```bash
cd packages/app
yarn add @backstage/plugin-kubernetes
```

### Konfiguration in `app-config.yaml`

```yaml
kubernetes:
  serviceLocatorMethod:
    type: 'multiTenant'
  clusterLocatorMethods:
    - type: 'config'
      clusters:
        - url: https://kubernetes.default.svc
          name: production-cluster
          authProvider: 'serviceAccount'
          skipTLSVerify: false
          serviceAccountToken: ${K8S_SERVICE_ACCOUNT_TOKEN}
```

### Component Page erweitern

```tsx
// packages/app/src/components/catalog/EntityPage.tsx
import { EntityKubernetesContent } from '@backstage/plugin-kubernetes';

const serviceEntityPage = (
  <EntityLayout>
    <EntityLayout.Route path="/" title="Overview">
      <Grid container spacing={3}>
        <Grid item md={6}>
          <EntityAboutCard />
        </Grid>
      </Grid>
    </EntityLayout.Route>

    <EntityLayout.Route path="/kubernetes" title="Kubernetes">
      <EntityKubernetesContent />
    </EntityLayout.Route>
  </EntityLayout>
);
```

### Components mit Kubernetes-Labels annotieren

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    backstage.io/kubernetes-id: taskflow-backend
    backstage.io/kubernetes-namespace: production
```

Die Component-Seite des TaskFlow Backends zeigt danach:

* Deployment-Status mit Replicas und Ready Pods
* Pod-Liste mit Status und Logs
* Service Endpoints
* Ingress Routes
* Ressourcennutzung für CPU und Memory

Entwickler können Deployment-Probleme analysieren, ohne zu `kubectl` oder zur AWS Console zu wechseln.

---

## Monitoring-Tools anbinden

Backstage kann mit Grafana, Prometheus, Datadog oder anderen Monitoring-Plattformen verbunden werden.

---

## Grafana Plugin

### Plugin installieren

```bash
yarn add @k-phoen/backstage-plugin-grafana
```

### Konfiguration in `app-config.yaml`

```yaml
grafana:
  domain: https://grafana.taskflow.dev
  unifiedAlerting: true
```

### Dashboard-Links zu Components hinzufügen

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    grafana/dashboard-selector: "taskflow-backend"
    grafana/alert-label-selector: "service=taskflow-backend"
```

Dadurch werden Grafana-Dashboards direkt in Component-Seiten eingebettet.

Entwickler sehen:

* Request Rate, Latenz und Error Rate
* Trends zur Ressourcennutzung
* Aktive Alerts
* Links zu vollständigen Grafana-Dashboards

---

## Prometheus-Metriken

Für die Prometheus-Integration werden Annotationen verwendet.

```yaml
annotations:
  prometheus.io/rule: 'sum(rate(http_requests_total{service="taskflow-backend"}[5m]))'
```

Damit werden Prometheus-Metriken direkt in Backstage angezeigt. Ein zusätzlicher Wechsel zu Grafana ist nicht nötig.

---

## TechDocs: Documentation as Code

TechDocs wandelt Markdown-Dokumentation aus Repositories in strukturierte, durchsuchbare Dokumentation in Backstage um.

### TechDocs in `app-config.yaml` aktivieren

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```

### Produktionskonfiguration mit Cloud Storage

```yaml
techdocs:
  builder: 'external'
  publisher:
    type: 'awsS3'
    awsS3:
      bucketName: 'taskflow-techdocs'
      region: 'us-east-1'
      credentials:
        accessKeyId: ${AWS_ACCESS_KEY_ID}
        secretAccessKey: ${AWS_SECRET_ACCESS_KEY}
```

---

## Dokumentation zu Components hinzufügen

Erstelle ein `docs/`-Verzeichnis im Repository.

```text
taskflow-backend/
├── src/
├── docs/
│   ├── index.md
│   ├── api-reference.md
│   ├── deployment.md
│   └── troubleshooting.md
└── mkdocs.yml
```

### `mkdocs.yml` konfigurieren

```yaml
site_name: 'TaskFlow Backend Documentation'
site_description: 'FastAPI backend for TaskFlow application'

nav:
  - Home: index.md
  - API Reference: api-reference.md
  - Deployment: deployment.md
  - Troubleshooting: troubleshooting.md

plugins:
  - techdocs-core
```

### Component für TechDocs annotieren

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    backstage.io/techdocs-ref: dir:.
```

`dir:.` bedeutet, dass die Dokumentation im selben Repository liegt. Alternativ kann `url:` für externe Repositories verwendet werden.

---

## Dokumentation schreiben

### `docs/index.md`

Willkommen in der TaskFlow Backend-Dokumentation. Dieser FastAPI-Service stellt die REST API für das Task Management bereit.

## Überblick

Das Backend übernimmt:

* CRUD-Operationen für Tasks
* Benutzerauthentifizierung
* Persistenz mit PostgreSQL
* Caching mit Redis

## Quick Start

Lokal starten:

```bash
docker-compose up
uvicorn src.main:app --reload
```

API-Dokumentation öffnen:

```text
http://localhost:8000/docs
```

## Architektur

Das Backend verwendet FastAPI mit einer PostgreSQL-Datenbank und Redis Cache.

> **Diagramm:** Das Architekturdiagramm wird hier eingefügt.

## API Endpoints

Weitere Details zu den Endpoints stehen in der API Reference.

TechDocs rendert diese Inhalte mit Navigation, Suche und Backstage Theme. Entwickler lesen die Dokumentation direkt in Backstage.

---

## Authentifizierung und Autorisierung

Konfiguriere die Authentifizierung für den Zugriff auf das Backstage Portal.

### GitHub OAuth

GitHub OAuth wird für diesen Anwendungsfall empfohlen.

```yaml
auth:
  environment: production
  providers:
    github:
      production:
        clientId: ${GITHUB_OAUTH_CLIENT_ID}
        clientSecret: ${GITHUB_OAUTH_CLIENT_SECRET}
```

### GitHub OAuth App erstellen

1. Öffne GitHub Settings → Developer Settings → OAuth Apps → New OAuth App.
2. Setze den Application Name auf `TaskFlow Backstage`.
3. Setze die Homepage URL auf `https://backstage.taskflow.dev`.
4. Setze die Authorization Callback URL auf `https://backstage.taskflow.dev/api/auth/github/handler/frame`.
5. Kopiere die Client ID und generiere ein Client Secret.
6. Setze die benötigten Umgebungsvariablen.

### Sign-in in `App.tsx` aktivieren

```tsx
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPage } from '@backstage/core-components';

const app = createApp({
  components: {
    SignInPage: props => (
      <SignInPage
        {...props}
        auto
        provider={{
          id: 'github-auth-provider',
          title: 'GitHub',
          message: 'Sign in using GitHub',
          apiRef: githubAuthApiRef,
        }}
      />
    ),
  },
});
```

Benutzer authentifizieren sich danach mit GitHub, bevor sie auf Backstage zugreifen.

---

## Autorisierung und Permissions

Implementiere rollenbasierte Zugriffskontrolle mit RBAC.

### Permissions in `app-config.yaml` definieren

```yaml
permission:
  enabled: true
  policy:
    - resource: 'catalog-entity'
      actions: ['read']
      allow:
        - kind: 'Group'
          name: 'everyone'
    - resource: 'catalog-entity'
      actions: ['create', 'update', 'delete']
      allow:
        - kind: 'Group'
          name: 'platform-team'
    - resource: 'scaffolder-template'
      actions: ['use']
      allow:
        - kind: 'Group'
          name: 'developers'
```

Diese Konfiguration gewährt:

* Alle Benutzer können Catalog Entities lesen.
* Nur das `platform-team` kann Entities erstellen, aktualisieren oder löschen.
* Entwickler können Software Templates verwenden.

### Custom Permission Policies in TypeScript

Für komplexere Regeln können eigene Permission Policies in TypeScript implementiert werden.

```ts
import { PermissionPolicy } from '@backstage/plugin-permission-node';

export class CustomPermissionPolicy implements PermissionPolicy {
  async handle(request, user) {
    if (request.permission.name === 'catalog.entity.delete') {
      // Only allow deletion for entities owned by user's team
      const entity = request.resourceRef;
      const ownerTeam = entity.metadata.annotations['owner'];
      return user.groups.includes(ownerTeam);
    }
    return { result: AuthorizeResult.ALLOW };
  }
}
```

---

## Vollständiges Developer Portal für TaskFlow aufbauen

> **Abbildung:** Backstage Platform Integrations — Backstage als zentraler Hub, der GitHub, Kubernetes, Grafana, TechDocs, Catalog, Templates, Authentifizierung und RBAC verbindet.

Kombiniere alle Integrationen zu einem vollständigen Portal.

| Component             | Nutzen                                                                            |
| --------------------- | --------------------------------------------------------------------------------- |
| Catalog               | Alle TaskFlow Components sind registriert: Frontend, Backend, Datenbank und Redis |
| TechDocs              | Dokumentation je Component: API Reference, Deployment Guides und Troubleshooting  |
| GitHub Integration    | Commits, Pull Requests und Workflow Runs je Component anzeigen                    |
| Kubernetes Plugin     | Pod-Status, Logs und Ressourcennutzung anzeigen                                   |
| Monitoring Dashboards | Grafana-Dashboards für Performance-Metriken einbetten                             |
| Custom Plugins        | TaskFlow Dashboard mit anwendungsspezifischen Metriken                            |
| Software Templates    | Template `New FastAPI Microservice` für Self-Service-Projekterstellung            |
| Authentication        | GitHub OAuth für sicheren Portalzugriff                                           |
| Permissions           | RBAC zur Steuerung von Catalog-Änderungen und Template-Nutzung                    |

---

## Developer Workflow

1. Mit GitHub anmelden.
2. Im Catalog das TaskFlow Backend finden.
3. TechDocs für die API-Dokumentation lesen.
4. Im Kubernetes-Tab den Deployment-Status prüfen.
5. Das Grafana-Dashboard für Performance-Metriken öffnen.
6. Im GitHub-Tab aktuelle Commits und Pull Requests anzeigen.
7. Einen neuen Microservice über ein Template erstellen.
8. Das TaskFlow Dashboard für den Anwendungszustand überwachen.

Alles, was Entwickler benötigen, ist an einem Ort verfügbar. Navigation und Suche bleiben konsistent.

---

## Best Practices für Developer Experience

| Bereich         | Best Practice                                                                                                                                                         |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Portal Adoption | Backstage in das Onboarding neuer Entwickler aufnehmen, Workshops durchführen, interne Workflow-Guides erstellen und Team Champions benennen                          |
| Content Quality | Catalog-Registrierung über CI/CD automatisieren, TechDocs wie Code behandeln, Alerts für Integrationsfehler einrichten und einen Deprecation-Prozess pflegen          |
| Performance     | Backend-Caching für Catalog Queries nutzen, Plugin-Code lazy laden und langsame Seiten messen und optimieren                                                          |
| Security        | Least-Privilege-Berechtigungen vergeben, Zugriff auf sensible Daten protokollieren, Tokens regelmäßig rotieren und Backstage-Abhängigkeiten auf Schwachstellen prüfen |

---

## Einfluss auf die Developer Experience messen

Verfolge Metriken, um den Nutzen des Portals messbar zu machen.

| Kategorie              | Zu messende Metriken                                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Developer Productivity | Time to First Commit für neue Projekte, Onboarding-Zeit, Kontextwechsel pro Tag, Self-Service Completion Rate                         |
| Portal Adoption        | Daily Active Users, Page Views pro Component, Template-Nutzungsrate, Suchanfragen und Click-through Rate                              |
| Developer Satisfaction | NPS und Zufriedenheitswerte aus Umfragen, Support-Ticket-Volumen, Auffindbarkeit der Dokumentation, Feedback aus Entwicklerinterviews |

### Beispielergebnisse nach dem Rollout des TaskFlow Portals

| Metrik                                              |   Vorher |    Nachher |
| --------------------------------------------------- | -------: | ---------: |
| Zeit zum Erstellen eines neuen Microservice         | 2–3 Tage | 15 Minuten |
| Entwickler, die Dokumentation über Backstage finden |     30 % |       80 % |
| Kontextwechsel pro Tag                              |      20+ |         ~5 |
| Developer NPS Score                                 |       40 |         75 |

Nutze diese Metriken, um weitere Investitionen in das Portal zu begründen und Verbesserungsbereiche zu identifizieren.

---

## Beispiele für Self-Service-Automatisierung

### Automatisierte Bereitstellung von Umgebungen

* Template erstellt Dev- und Staging-Umgebungen in AWS.
* RDS-Datenbank und ElastiCache Redis werden provisioniert.
* VPC und Security Groups werden eingerichtet.
* Deployment nach EKS erfolgt mit Terraform.

### Onboarding-Automatisierung

* Template erstellt einen Slack Channel.
* Entwickler wird zu GitHub Teams hinzugefügt.
* Jira-Projekt wird erstellt.
* Willkommens-E-Mail mit relevanten Links wird versendet.

### Compliance Checks

* Automatisierte Security Scans für neue Projekte
* Prüfung der Lizenzkonformität
* Checkliste für GDPR- und Privacy-Anforderungen
* Approval Workflows für Produktionszugriff

### Dokumentationsgenerierung

* API-Dokumentation automatisch aus OpenAPI Specs generieren
* Architekturdiagramme aus Code erstellen
* Changelog aus Commits generieren
* TechDocs bei jedem Merge nach `main` aktualisieren

Diese Automatisierungen reduzieren manuelle Arbeit, erzwingen Standards und geben Entwicklern mehr Zeit für Feature-Entwicklung.

---

## Ausblick

Glückwunsch. Du hast Woche 12 abgeschlossen: Backstage – Internal Developer Portal.

Du hast gelernt:

* Grundlagen interner Developer Portals und Developer Experience
* React Framework im Vergleich zu Vue.js
* Backstage Service Catalog und Component-Registrierung
* React Hooks und Component-Entwicklung
* Backstage Plugins bauen
* TaskFlow Dashboard Plugin mit Echtzeitmetriken erstellen
* Software Templates und Scaffolder für Projekt-Automatisierung
* Plattformintegrationen mit GitHub, Kubernetes, Monitoring und TechDocs
* Authentifizierung, Autorisierung und Security
* Einfluss auf die Developer Experience messen

Du hast ein vollständiges internes Developer Portal für TaskFlow aufgebaut mit:

* Service Catalog für alle Components
* Custom Dashboard Plugin für Metriken
* Software Template zur Erstellung neuer Microservices
* GitHub-, Kubernetes- und Monitoring-Integrationen
* Documentation as Code mit TechDocs
* Self-Service Workflows für Entwickler

Diese Platform-Engineering-Fähigkeit verbessert die Produktivität deutlich. Sie verkürzt die Onboarding-Zeit und schafft eine bessere Developer Experience für die Organisation.

In den kommenden Wochen geht es im Platform Engineering Bootcamp weiter mit Observability über Prometheus und Grafana, Service Mesh mit Istio, Secrets Management mit Vault und GitOps mit ArgoCD.

Jede Woche baut auf dieser Grundlage auf und erweitert dein Platform-Engineering-Skillset.

Du hast jetzt die Fähigkeiten, interne Developer Portals zu bauen, die Developer Experience und Plattformfähigkeiten in deiner Organisation skalieren.

---

## Kernaussagen

* Die GitHub-Integration zeigt Repository-Informationen, Commits, Pull Requests und Workflows auf Catalog-Seiten über die Annotation `github.com/project-slug`.
* Das Kubernetes Plugin zeigt Pod-Status, Logs und Ressourcennutzung über die Annotation `backstage.io/kubernetes-id` und die Cluster-Konfiguration.
* TechDocs rendert Markdown-Dokumentation aus Repositories mithilfe von `mkdocs.yml` und der Annotation `backstage.io/techdocs-ref`.
* GitHub OAuth stellt die Authentifizierung über `auth.providers.github` bereit. RBAC steuert Berechtigungen für Catalog und Templates.
* Der Einfluss auf die Developer Experience wird über Metriken gemessen: Time to First Commit, Onboarding-Zeit, Portal Adoption und Zufriedenheitswerte.



# Platform Self-Service, Templates und Integration

> **Beta:** Dieses Lab befindet sich in der Beta-Phase. Inhalte können aktualisiert werden, während das Material weiter verbessert wird.

Baue ein vollständiges Self-Service-Portal für die Plattform mit Backstage.

Du erstellst Software Templates, die automatisch FastAPI-Microservices generieren. Außerdem integrierst du GitHub für Repository-Management, verbindest Kubernetes für Cluster-Transparenz, verlinkst Monitoring-Dashboards und implementierst Authentifizierung sowie RBAC.

So wird TaskFlow zu einer Self-Service-Plattform.

---

## Lab-Übersicht

| Eigenschaft        | Wert            |
| ------------------ | --------------- |
| Schwierigkeitsgrad | Fortgeschritten |
| Dauer              | 60 Minuten      |
| Typ                | Lab             |

---

## Voraussetzungen

Für dieses Lab benötigst du:

* `Week 12 Lab 1-2: Backstage Setup and React Plugin completed`
* GitHub Account mit Organisation oder persönlichen Repositories
* Zugriff auf einen Kubernetes Cluster für die Integration
* Prometheus/Grafana Dashboards für das TaskFlow Monitoring
* Verständnis von CI/CD Workflows aus Woche 3

---

## Über dieses Lab

In diesem Lab erstellst du Backstage Software Templates für Plattformautomatisierung und integrierst Backstage mit GitHub, Kubernetes und Monitoring-Tools.

Du baust Self-Service-Workflows, die automatisch:

* Code generieren,
* Repositories erstellen,
* CI/CD-Pipelines einrichten,
* neue Services im Catalog registrieren,
* Best Practices standardmäßig anwenden.

Im Mittelpunkt steht das Template `New FastAPI Microservice`. Es erstellt vollständige Projekte mit integrierten Standards für Entwicklung, Deployment und Betrieb.

Du arbeitest mit:

* `template.yaml`-Definitionen mit Parametern und Actions
* `skeleton/`-Verzeichnissen mit Template-Dateien
* `fetch:template`
* `publish:github`
* `catalog:register`
* GitHub-Integration für Repository-Sichtbarkeit
* Kubernetes-Integration für Pod-Status
* Prometheus- und Grafana-Dashboard-Verknüpfungen
* GitHub OAuth für Authentifizierung
* RBAC für Team-Berechtigungen
* einem vollständigen Developer Portal für die TaskFlow-Plattform

---

## Was du lernst

Dieses Lab besteht aus 5 Aufgaben.

| Nr. | Aufgabe                                                            | Ziel                                              |
| --: | ------------------------------------------------------------------ | ------------------------------------------------- |
|   1 | Software Template für FastAPI Microservice erstellen               | Diese Aufgabe abschließen, um im Lab fortzufahren |
|   2 | Template Actions für Codegenerierung und Publishing implementieren | Diese Aufgabe abschließen, um im Lab fortzufahren |
|   3 | Backstage mit GitHub integrieren                                   | Diese Aufgabe abschließen, um im Lab fortzufahren |
|   4 | Backstage mit Kubernetes und Monitoring integrieren                | Diese Aufgabe abschließen, um im Lab fortzufahren |
|   5 | Authentifizierung und Role-Based Access Control konfigurieren      | Diese Aufgabe abschließen, um im Lab fortzufahren |

---

## Systemanforderungen

Für dieses Lab benötigst du:

* einen modernen Webbrowser,
* eine stabile Internetverbindung,
* ungefähr 60 Minuten konzentrierte Arbeitszeit,
* Kenntnisse aus `Week 12 Lab 1-2: Backstage Setup and React Plugin completed`,
* einen GitHub Account mit Organisation oder persönlichen Repositories,
* Zugriff auf einen Kubernetes Cluster.
===========================================




# Plattformintegration und Self-Service-Automatisierung

Vervollständige dein internes Developer Portal, indem du Backstage mit GitHub, Kubernetes, Monitoring-Tools und Dokumentationssystemen integrierst.

In dieser Lektion konfigurierst du GitHub OAuth für die Authentifizierung, implementierst TechDocs für Documentation as Code, bindest Monitoring-Dashboards ein und misst Verbesserungen der Developer Experience für die TaskFlow-Plattform.

---

## Lernmaterial

Du hast bereits das TaskFlow Dashboard Plugin erstellt und Software Templates für das Projekt-Scaffolding gebaut. Jetzt wird Backstage mit dem weiteren Plattform-Ökosystem verbunden: GitHub, Kubernetes, Monitoring-Tools und Dokumentationssysteme.

Diese Integrationen machen Backstage vom eigenständigen Portal zum zentralen Hub für Entwickler-Workflows. Entwickler können Pull Requests prüfen, Deployment-Status einsehen, Metrik-Dashboards öffnen und Dokumentation lesen, ohne Backstage zu verlassen.

Das reduziert Kontextwechsel und erhöht die Produktivität.

In dieser abschließenden Lektion lernst du:

* GitHub-Integration
* Kubernetes-Plugin-Konfiguration
* Anbindung von Monitoring-Tools
* TechDocs für Documentation as Code
* Authentifizierung und Autorisierung
* Messung des Einflusses deines internen Developer Portals auf die Developer Experience

---

## Backstage mit GitHub integrieren

Die GitHub-Integration ermöglicht:

* Repository-Informationen auf Catalog-Seiten anzeigen
* Aktuelle Commits und Pull Requests anzeigen
* GitHub Actions Workflows verlinken
* Repository-Contributors anzeigen
* Issues und Pull Requests direkt aus Backstage erstellen

### Konfiguration in `app-config.yaml`

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

---

## GitHub App erstellen

Für Produktionsumgebungen wird eine GitHub App empfohlen. Sie ist sicherer als Personal Access Tokens und besser für Teams verwaltbar.

### Vorgehen

1. Öffne GitHub Settings → Developer Settings → GitHub Apps.
2. Klicke auf **New GitHub App**.
3. Konfiguriere die App:

| Einstellung         | Wert                             |
| ------------------- | -------------------------------- |
| Name                | `TaskFlow Backstage Portal`      |
| Homepage URL        | `https://backstage.taskflow.dev` |
| Webhook             | Deaktiviert, vorerst             |
| Repository contents | Read                             |
| Pull requests       | Read & Write                     |
| Metadata            | Read                             |
| Workflows           | Read                             |

4. Generiere den Private Key und lade ihn herunter.
5. Installiere die App in deiner Organisation.

### `app-config.yaml` mit GitHub App aktualisieren

```yaml
integrations:
  github:
    - host: github.com
      apps:
        - appId: ${GITHUB_APP_ID}
          clientId: ${GITHUB_APP_CLIENT_ID}
          clientSecret: ${GITHUB_APP_CLIENT_SECRET}
          webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}
          privateKey: ${GITHUB_APP_PRIVATE_KEY}
```

Damit kann Backstage auf Repositories zugreifen, Pull Requests erstellen und Workflows im Namen der App auslösen.

---

## GitHub-Annotationen im Catalog

Components verwenden Annotationen, um eine Verbindung zu GitHub herzustellen.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    github.com/project-slug: tekanaid/taskflow-backend
    github.com/workflows: ci.yml,deploy.yml
spec:
  type: service
  owner: platform-team
```

Mit diesen Annotationen zeigt Backstage auf jeder Component-Seite relevante GitHub-Informationen an.

| Backstage-Anzeige                       | Quelle                  |
| --------------------------------------- | ----------------------- |
| Repository README                       | GitHub Repository Root  |
| Zeitachse der letzten Commits           | GitHub Commits API      |
| Offene und gemergte Pull Requests       | GitHub PR API           |
| GitHub Actions Workflow Runs und Status | GitHub Actions API      |
| Contributors und Code Owners            | GitHub Contributors API |

Entwickler können bei Bedarf direkt zu GitHub wechseln. Die Übersicht erhalten sie jedoch direkt in Backstage.

---

## Backstage mit Kubernetes integrieren

Das Kubernetes Plugin zeigt Cluster-Informationen direkt in Backstage an.

### Plugin installieren

```bash
cd packages/app
yarn add @backstage/plugin-kubernetes
```

### Konfiguration in `app-config.yaml`

```yaml
kubernetes:
  serviceLocatorMethod:
    type: 'multiTenant'
  clusterLocatorMethods:
    - type: 'config'
      clusters:
        - url: https://kubernetes.default.svc
          name: production-cluster
          authProvider: 'serviceAccount'
          skipTLSVerify: false
          serviceAccountToken: ${K8S_SERVICE_ACCOUNT_TOKEN}
```

### Component Page erweitern

```tsx
// packages/app/src/components/catalog/EntityPage.tsx
import { EntityKubernetesContent } from '@backstage/plugin-kubernetes';

const serviceEntityPage = (
  <EntityLayout>
    <EntityLayout.Route path="/" title="Overview">
      <Grid container spacing={3}>
        <Grid item md={6}>
          <EntityAboutCard />
        </Grid>
      </Grid>
    </EntityLayout.Route>

    <EntityLayout.Route path="/kubernetes" title="Kubernetes">
      <EntityKubernetesContent />
    </EntityLayout.Route>
  </EntityLayout>
);
```

### Components mit Kubernetes-Labels annotieren

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    backstage.io/kubernetes-id: taskflow-backend
    backstage.io/kubernetes-namespace: production
```

Die Component-Seite des TaskFlow Backends zeigt danach:

* Deployment-Status mit Replicas und Ready Pods
* Pod-Liste mit Status und Logs
* Service Endpoints
* Ingress Routes
* Ressourcennutzung für CPU und Memory

Entwickler können Deployment-Probleme analysieren, ohne zu `kubectl` oder zur AWS Console zu wechseln.

---

## Monitoring-Tools anbinden

Backstage kann mit Grafana, Prometheus, Datadog oder anderen Monitoring-Plattformen verbunden werden.

---

## Grafana Plugin

### Plugin installieren

```bash
yarn add @k-phoen/backstage-plugin-grafana
```

### Konfiguration in `app-config.yaml`

```yaml
grafana:
  domain: https://grafana.taskflow.dev
  unifiedAlerting: true
```

### Dashboard-Links zu Components hinzufügen

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    grafana/dashboard-selector: "taskflow-backend"
    grafana/alert-label-selector: "service=taskflow-backend"
```

Dadurch werden Grafana-Dashboards direkt in Component-Seiten eingebettet.

Entwickler sehen:

* Request Rate, Latenz und Error Rate
* Trends zur Ressourcennutzung
* Aktive Alerts
* Links zu vollständigen Grafana-Dashboards

---

## Prometheus-Metriken

Für die Prometheus-Integration werden Annotationen verwendet.

```yaml
annotations:
  prometheus.io/rule: 'sum(rate(http_requests_total{service="taskflow-backend"}[5m]))'
```

Damit werden Prometheus-Metriken direkt in Backstage angezeigt. Ein zusätzlicher Wechsel zu Grafana ist nicht nötig.

---

## TechDocs: Documentation as Code

TechDocs wandelt Markdown-Dokumentation aus Repositories in strukturierte, durchsuchbare Dokumentation in Backstage um.

### TechDocs in `app-config.yaml` aktivieren

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```

### Produktionskonfiguration mit Cloud Storage

```yaml
techdocs:
  builder: 'external'
  publisher:
    type: 'awsS3'
    awsS3:
      bucketName: 'taskflow-techdocs'
      region: 'us-east-1'
      credentials:
        accessKeyId: ${AWS_ACCESS_KEY_ID}
        secretAccessKey: ${AWS_SECRET_ACCESS_KEY}
```

---

## Dokumentation zu Components hinzufügen

Erstelle ein `docs/`-Verzeichnis im Repository.

```text
taskflow-backend/
├── src/
├── docs/
│   ├── index.md
│   ├── api-reference.md
│   ├── deployment.md
│   └── troubleshooting.md
└── mkdocs.yml
```

### `mkdocs.yml` konfigurieren

```yaml
site_name: 'TaskFlow Backend Documentation'
site_description: 'FastAPI backend for TaskFlow application'

nav:
  - Home: index.md
  - API Reference: api-reference.md
  - Deployment: deployment.md
  - Troubleshooting: troubleshooting.md

plugins:
  - techdocs-core
```

### Component für TechDocs annotieren

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: taskflow-backend
  annotations:
    backstage.io/techdocs-ref: dir:.
```

`dir:.` bedeutet, dass die Dokumentation im selben Repository liegt. Alternativ kann `url:` für externe Repositories verwendet werden.

---

## Dokumentation schreiben

### `docs/index.md`

# TaskFlow Backend-Dokumentation

Willkommen in der Dokumentation des TaskFlow Backends. Dieser FastAPI-Service stellt die REST API für das Task Management bereit.

## Überblick

Das Backend übernimmt:

* CRUD-Operationen für Tasks
* Benutzerauthentifizierung
* Persistenz mit PostgreSQL
* Caching mit Redis

## Quick Start

Lokal starten:

```bash
docker-compose up
uvicorn src.main:app --reload
```

API-Dokumentation öffnen:

```text
http://localhost:8000/docs
```

## Architektur

Das Backend verwendet FastAPI mit einer PostgreSQL-Datenbank und Redis Cache.

> **Diagramm:** Das Architekturdiagramm wird hier eingefügt.

## API Endpoints

Weitere Details zu den Endpoints stehen in der API Reference.

TechDocs rendert diese Inhalte mit Navigation, Suche und Backstage Theme. Entwickler lesen die Dokumentation direkt in Backstage.

---

## Authentifizierung und Autorisierung

Konfiguriere die Authentifizierung für den Zugriff auf das Backstage Portal.

### GitHub OAuth

GitHub OAuth wird für diesen Anwendungsfall empfohlen.

```yaml
auth:
  environment: production
  providers:
    github:
      production:
        clientId: ${GITHUB_OAUTH_CLIENT_ID}
        clientSecret: ${GITHUB_OAUTH_CLIENT_SECRET}
```

### GitHub OAuth App erstellen

1. Öffne GitHub Settings → Developer Settings → OAuth Apps → New OAuth App.
2. Setze den Application Name auf `TaskFlow Backstage`.
3. Setze die Homepage URL auf `https://backstage.taskflow.dev`.
4. Setze die Authorization Callback URL auf `https://backstage.taskflow.dev/api/auth/github/handler/frame`.
5. Kopiere die Client ID und generiere ein Client Secret.
6. Setze die benötigten Umgebungsvariablen.

### Sign-in in `App.tsx` aktivieren

```tsx
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPage } from '@backstage/core-components';

const app = createApp({
  components: {
    SignInPage: props => (
      <SignInPage
        {...props}
        auto
        provider={{
          id: 'github-auth-provider',
          title: 'GitHub',
          message: 'Sign in using GitHub',
          apiRef: githubAuthApiRef,
        }}
      />
    ),
  },
});
```

Benutzer authentifizieren sich danach mit GitHub, bevor sie auf Backstage zugreifen.

---

## Autorisierung und Permissions

Implementiere rollenbasierte Zugriffskontrolle mit RBAC.

### Permissions in `app-config.yaml` definieren

```yaml
permission:
  enabled: true
  policy:
    - resource: 'catalog-entity'
      actions: ['read']
      allow:
        - kind: 'Group'
          name: 'everyone'
    - resource: 'catalog-entity'
      actions: ['create', 'update', 'delete']
      allow:
        - kind: 'Group'
          name: 'platform-team'
    - resource: 'scaffolder-template'
      actions: ['use']
      allow:
        - kind: 'Group'
          name: 'developers'
```

Diese Konfiguration gewährt:

* Alle Benutzer können Catalog Entities lesen.
* Nur das `platform-team` kann Entities erstellen, aktualisieren oder löschen.
* Entwickler können Software Templates verwenden.

### Custom Permission Policies in TypeScript

Für komplexere Regeln können eigene Permission Policies in TypeScript implementiert werden.

```ts
import { PermissionPolicy } from '@backstage/plugin-permission-node';

export class CustomPermissionPolicy implements PermissionPolicy {
  async handle(request, user) {
    if (request.permission.name === 'catalog.entity.delete') {
      // Only allow deletion for entities owned by user's team
      const entity = request.resourceRef;
      const ownerTeam = entity.metadata.annotations['owner'];
      return user.groups.includes(ownerTeam);
    }
    return { result: AuthorizeResult.ALLOW };
  }
}
```

---

## Vollständiges Developer Portal für TaskFlow aufbauen

> **Abbildung:** Backstage-Plattformintegrationen. Backstage dient als zentraler Hub und verbindet GitHub, Kubernetes, Grafana, TechDocs, Catalog, Templates, Authentifizierung und RBAC.

Kombiniere alle Integrationen zu einem vollständigen Portal.

| Component             | Nutzen                                                                            |
| --------------------- | --------------------------------------------------------------------------------- |
| Catalog               | Alle TaskFlow Components sind registriert: Frontend, Backend, Datenbank und Redis |
| TechDocs              | Dokumentation je Component: API Reference, Deployment Guides und Troubleshooting  |
| GitHub Integration    | Commits, Pull Requests und Workflow Runs je Component anzeigen                    |
| Kubernetes Plugin     | Pod-Status, Logs und Ressourcennutzung anzeigen                                   |
| Monitoring Dashboards | Grafana-Dashboards für Performance-Metriken einbetten                             |
| Custom Plugins        | TaskFlow Dashboard mit anwendungsspezifischen Metriken                            |
| Software Templates    | Template `New FastAPI Microservice` für Self-Service-Projekterstellung            |
| Authentication        | GitHub OAuth für sicheren Portalzugriff                                           |
| Permissions           | RBAC zur Steuerung von Catalog-Änderungen und Template-Nutzung                    |

---

## Developer Workflow

1. Mit GitHub anmelden.
2. Im Catalog das TaskFlow Backend finden.
3. TechDocs für die API-Dokumentation lesen.
4. Im Kubernetes-Tab den Deployment-Status prüfen.
5. Das Grafana-Dashboard für Performance-Metriken öffnen.
6. Im GitHub-Tab aktuelle Commits und Pull Requests anzeigen.
7. Einen neuen Microservice über ein Template erstellen.
8. Das TaskFlow Dashboard für den Anwendungszustand überwachen.

Alles, was Entwickler benötigen, ist an einem Ort verfügbar. Navigation und Suche bleiben konsistent.

---

## Best Practices für Developer Experience

| Bereich         | Best Practice                                                                                                                                                         |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Portal Adoption | Backstage in das Onboarding neuer Entwickler aufnehmen, Workshops durchführen, interne Workflow-Guides erstellen und Team Champions benennen                          |
| Content Quality | Catalog-Registrierung über CI/CD automatisieren, TechDocs wie Code behandeln, Alerts für Integrationsfehler einrichten und einen Deprecation-Prozess pflegen          |
| Performance     | Backend-Caching für Catalog Queries nutzen, Plugin-Code lazy laden und langsame Seiten messen und optimieren                                                          |
| Security        | Least-Privilege-Berechtigungen vergeben, Zugriff auf sensible Daten protokollieren, Tokens regelmäßig rotieren und Backstage-Abhängigkeiten auf Schwachstellen prüfen |

---

## Einfluss auf die Developer Experience messen

Verfolge Metriken, um den Nutzen des Portals messbar zu machen.

| Kategorie              | Zu messende Metriken                                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Developer Productivity | Time to First Commit für neue Projekte, Onboarding-Zeit, Kontextwechsel pro Tag, Self-Service Completion Rate                         |
| Portal Adoption        | Daily Active Users, Page Views pro Component, Template-Nutzungsrate, Suchanfragen und Click-through Rate                              |
| Developer Satisfaction | NPS und Zufriedenheitswerte aus Umfragen, Support-Ticket-Volumen, Auffindbarkeit der Dokumentation, Feedback aus Entwicklerinterviews |

### Beispielergebnisse nach dem Rollout des TaskFlow Portals

| Metrik                                              |   Vorher |    Nachher |
| --------------------------------------------------- | -------: | ---------: |
| Zeit zum Erstellen eines neuen Microservice         | 2–3 Tage | 15 Minuten |
| Entwickler, die Dokumentation über Backstage finden |     30 % |       80 % |
| Kontextwechsel pro Tag                              |      20+ |         ~5 |
| Developer NPS Score                                 |       40 |         75 |

Nutze diese Metriken, um weitere Investitionen in das Portal zu begründen und Verbesserungsbereiche zu identifizieren.

---

## Beispiele für Self-Service-Automatisierung

### Automatisierte Bereitstellung von Umgebungen

* Template erstellt Dev- und Staging-Umgebungen in AWS.
* RDS-Datenbank und ElastiCache Redis werden provisioniert.
* VPC und Security Groups werden eingerichtet.
* Deployment nach EKS erfolgt mit Terraform.

### Onboarding-Automatisierung

* Template erstellt einen Slack Channel.
* Entwickler wird zu GitHub Teams hinzugefügt.
* Jira-Projekt wird erstellt.
* Willkommens-E-Mail mit relevanten Links wird versendet.

### Compliance Checks

* Automatisierte Security Scans für neue Projekte
* Prüfung der Lizenzkonformität
* Checkliste für GDPR- und Privacy-Anforderungen
* Approval Workflows für Produktionszugriff

### Dokumentationsgenerierung

* API-Dokumentation automatisch aus OpenAPI Specs generieren
* Architekturdiagramme aus Code erstellen
* Changelog aus Commits generieren
* TechDocs bei jedem Merge nach `main` aktualisieren

Diese Automatisierungen reduzieren manuelle Arbeit, erzwingen Standards und geben Entwicklern mehr Zeit für Feature-Entwicklung.

---

## Ausblick

Glückwunsch. Du hast Woche 12 abgeschlossen: Backstage – Internal Developer Portal.

Du hast gelernt:

* Grundlagen interner Developer Portals und Developer Experience
* React Framework im Vergleich zu Vue.js
* Backstage Service Catalog und Component-Registrierung
* React Hooks und Component-Entwicklung
* Backstage Plugins bauen
* TaskFlow Dashboard Plugin mit Echtzeitmetriken erstellen
* Software Templates und Scaffolder für Projekt-Automatisierung
* Plattformintegrationen mit GitHub, Kubernetes, Monitoring und TechDocs
* Authentifizierung, Autorisierung und Security
* Einfluss auf die Developer Experience messen

Du hast ein vollständiges internes Developer Portal für TaskFlow aufgebaut mit:

* Service Catalog für alle Components
* Custom Dashboard Plugin für Metriken
* Software Template zur Erstellung neuer Microservices
* GitHub-, Kubernetes- und Monitoring-Integrationen
* Documentation as Code mit TechDocs
* Self-Service Workflows für Entwickler

Diese Platform-Engineering-Fähigkeit verbessert die Produktivität deutlich. Sie verkürzt die Onboarding-Zeit und schafft eine bessere Developer Experience für die Organisation.

In den kommenden Wochen geht es im Platform Engineering Bootcamp weiter mit Observability über Prometheus und Grafana, Service Mesh mit Istio, Secrets Management mit Vault und GitOps mit ArgoCD.

Jede Woche baut auf dieser Grundlage auf und erweitert dein Platform-Engineering-Skillset.

Du hast jetzt die Fähigkeiten, interne Developer Portals zu bauen, die Developer Experience und Plattformfähigkeiten in deiner Organisation skalieren.

---

## Kernaussagen

* Die GitHub-Integration zeigt Repository-Informationen, Commits, Pull Requests und Workflows auf Catalog-Seiten über die Annotation `github.com/project-slug`.
* Das Kubernetes Plugin zeigt Pod-Status, Logs und Ressourcennutzung über die Annotation `backstage.io/kubernetes-id` und die Cluster-Konfiguration.
* TechDocs rendert Markdown-Dokumentation aus Repositories mithilfe von `mkdocs.yml` und der Annotation `backstage.io/techdocs-ref`.
* GitHub OAuth stellt die Authentifizierung über `auth.providers.github` bereit. RBAC steuert Berechtigungen für Catalog und Templates.
* Der Einfluss auf die Developer Experience wird über Metriken gemessen: Time to First Commit, Onboarding-Zeit, Portal Adoption und Zufriedenheitswerte.

---

## Zusätzliche Ressourcen

| Ressource                              | Typ           |
| -------------------------------------- | ------------- |
| Backstage-Integrationen – Übersicht    | Dokumentation |
| Kubernetes Plugin Dokumentation        | Dokumentation |
| TechDocs Dokumentation                 | Dokumentation |
| Authentication and Authorization Guide | Dokumentation |

---

## Private Lesson Notes

Private Notizen während des Lernens erfassen.

---

## Study Group

### Backstage – Internes Developer Portal

Baue ein internes Developer Portal mit Backstage, um die Developer Experience zu verbessern. Lerne das React Framework durch die Entwicklung von Backstage Plugins. Die Teilnehmer kennen bereits Vue.js.

Erstelle Service Catalog, Dokumentation und Templates für TaskFlow.

> ⚠️ Pilotumfang: Kerninhalte des Bootcamps starten in Monat 5.

| Information       | Wert |
| ----------------- | ---: |
| Mitglieder        |    1 |
| Aktuelle Beiträge |    0 |

Tritt dieser Study Group bei, um an Diskussionen teilzunehmen und Beiträge zu sehen.

