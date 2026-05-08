
***
# Technischer Implementierungsbericht: Aufbau einer Backstage‑basierten Internal Developer Platform (IDP)

## 1. Ausgangspunkt und Ziel

Ziel der Implementierung war der Aufbau einer Internal Developer Platform (IDP) auf Basis von Backstage, mit dem Fokus auf:

*   zentrale Orchestrierung von Entwickler‑Workflows,
*   Standardisierung der Erstellung von Services,
*   Integration von Azure DevOps und GitHub,
*   Reduktion der direkten Tool‑Komplexität für Entwickler.

Der Prototyp sollte zeigen, ob Backstage als zentrale Plattform eingesetzt werden kann, **bevor** weitere Investitionen in eine reine Infrastructure‑as‑Code‑Orchestrierung erfolgen.

***

## 2. Ursprüngliche Zielarchitektur (AKS‑basiert)

Die initiale Zielarchitektur sah vor:

*   Backstage als containerisierte Anwendung,
*   Betrieb auf Azure Kubernetes Service (AKS),
*   Nutzung von Azure‑nativer Identity (Azure AD / Entra ID),
*   Absicherung über IAM und Kubernetes‑RBAC.

Diese Architektur entsprach einer produktionsnahen Enterprise‑IDP.

***

## 3. Einschränkungen und notwendige Anpassungen

Während der Umsetzung stellte sich heraus, dass:

*   keine ausreichenden IAM‑Berechtigungen für AKS vorlagen,
*   keine Azure Container Registry erstellt werden konnte,
*   keine Azure‑native Identity‑Integration möglich war.

Diese Einschränkungen waren organisatorischer Natur und **nicht technisch bedingt**.

**Entscheidung:**  
Die Implementierung wurde bewusst in einen **lokalen, containerisierten Proof‑of‑Concept** überführt, ohne das Zielbild aufzugeben.

***

## 4. Aufbau der lokalen Entwicklungsumgebung

Die Entwicklung erfolgte lokal unter Linux (Ubuntu / WSL2) mit:

*   Node.js (Backstage Runtime),
*   Yarn,
*   Docker und Docker Compose,
*   GitHub als Versionsverwaltung.

Backstage wurde als Monorepo initialisiert und lokal betrieben (Frontend + Backend).

***

## 5. Containerisierung von Backstage

Zur Vorbereitung des späteren AKS‑Betriebs wurde Backstage containerisiert:

*   Erstellung eines Dockerfiles für das Backend,
*   Nutzung der Backstage‑Build‑Artefakte (`skeleton.tar.gz`, `bundle.tar.gz`),
*   Trennung von Entwicklungs‑ und Produktionskonfiguration.

Das erzeugte Docker‑Image wurde lokal getestet und anschließend in Docker Hub veröffentlicht, da Azure Container Registry nicht verfügbar war.

***

## 6. Kubernetes‑Vorbereitung (ohne produktives AKS)

Obwohl kein produktives AKS‑Deployment möglich war:

*   wurden Kubernetes‑Deployments erstellt,
*   Image‑Rollouts getestet,
*   der grundsätzliche Betrieb in Kubernetes validiert.

Damit ist die AKS‑Fähigkeit **technisch nachgewiesen**, auch wenn sie organisatorisch nicht abgeschlossen werden konnte.

***

## 7. Entscheidung für Keycloak als IAM‑Ersatz

Da Azure‑native Identity nicht nutzbar war, wurde **Keycloak** als externer Identity Provider eingeführt.

Umsetzung:

*   Betrieb von Keycloak in Docker / Docker Compose,
*   Anlegen eines Realms (`Kuka-devops`),
*   Konfiguration eines OIDC‑Clients für Backstage,
*   Definition korrekter Redirect‑URIs für Backstage.

Zusätzlich wurden Anpassungen an der Backstage‑Authentifizierung vorgenommen, um OIDC zu unterstützen.

**Ergebnis:**  
Die OIDC‑Integration funktionierte grundsätzlich, erhöhte jedoch die Komplexität und Instabilität des Proof of Concept.

***

## 8. Bewusste Entscheidung für Guest Authentication

Da das Ziel des Proof of Concept **nicht Sicherheit**, sondern **Plattform‑Orchestrierung** war, wurde entschieden:

*   Keycloak **nicht produktiv zu aktivieren**,
*   stattdessen den Backstage Guest Provider zu nutzen.

Diese Entscheidung ermöglichte:

*   stabile Entwicklung,
*   Fokus auf Catalog und Scaffolder,
*   reproduzierbare Demonstration der Kernfunktionen.

**Wichtig:**  
Keycloak wurde **implementiert, getestet und bewertet**, nicht ignoriert.

***

## 9. Aufbau des Software Catalog

Der Software Catalog bildet das Herzstück der IDP.

Umgesetzt wurden:

*   `System`‑Entitäten zur fachlichen Strukturierung,
*   `Component`‑Entitäten für Services,
*   `Resource`‑Entitäten für Azure DevOps‑Projekte,
*   `User` und `Group` für Ownership.

Alle Services sind damit:

*   eindeutig zuordenbar,
*   governance‑fähig,
*   maschinenlesbar.

***

## 10. Governance‑Modellierung

Governance wurde **bewusst modellbasiert** umgesetzt:

*   Ownership über Catalog (`spec.owner`),
*   Teams über `Group`‑Entitäten,
*   kein technisches RBAC im PoC.

Dies entspricht einem realistischen Plattform‑Vorgehen:  
**Erst Modell, dann Durchsetzung.**

***

## 11. Implementierung des Scaffolder (Golden Path)

Der Scaffolder ist das zentrale Orchestrierungswerkzeug.

Es wurden mehrere Templates erstellt, u. a.:

*   Azure DevOps Service Template,
*   Azure DevOps Repository Template,
*   Pipeline Bootstrap Template.

Diese Templates ermöglichen:

*   standardisierte Service‑Erstellung,
*   automatische Erzeugung von Repository‑Strukturen,
*   Vorbereitung von CI/CD‑Pipelines,
*   automatische Registrierung im Catalog.

***

## 12. Integration von GitHub

GitHub wurde als zentrales Repository‑Backend genutzt.

Umsetzung:

*   Nutzung von GitHub CLI,
*   saubere Authentifizierung via Personal Access Token,
*   Behebung eines initialen Problems durch ungültige `GITHUB_TOKEN`‑Variable,
*   erfolgreicher Push der durch den Scaffolder erzeugten Repositories.

**Wichtig:**  
GitHub Auth für Backstage wurde **bewusst nicht aktiviert**, da GitHub nur als Repository‑Backend dient, nicht als User‑Identity.

***

## 13. Ergebnis des Proof of Concept

Am Ende der Implementierung existiert eine funktionierende IDP mit:

*   stabilem Backstage‑Betrieb,
*   Software Catalog als Single Source of Truth,
*   Golden Paths über Scaffolder,
*   automatisierter Repository‑Erstellung,
*   vorbereiteten CI/CD‑Strukturen,
*   klar dokumentierten Architekturentscheidungen.

***

## 14. Reproduzierbarkeit

Ein Dritter kann dieses Ergebnis reproduzieren, indem er:

1.  Backstage lokal installiert,
2.  Docker‑ und Kubernetes‑Artefakte übernimmt,
3.  Keycloak optional evaluiert,
4.  Guest Auth aktiviert,
5.  Catalog‑Entitäten definiert,
6.  Scaffolder Templates ausführt,
7.  GitHub CLI korrekt authentifiziert.

***

## 15. Schlussfolgerung

Der Proof of Concept zeigt, dass Backstage als Internal Developer Platform:

*   als zentrale Orchestrierungs‑Schicht fungieren kann,
*   technische Komplexität abstrahiert,
*   externe Tools kontrolliert integriert,
*   und eine fundierte Entscheidungsbasis für eine produktive AKS‑IDP liefert.

***

# A quoi sert exactement le prototype que j'ai construit**, **ce qu’il change pour une équipe**, et **ce qu’il deviendrait une fois déployé en production sur AKS**.

1.  ❓ **Quel problème réel ce prototype résout**
2.  🧠 **Ce que fait concrètement ton prototype (aujourd’hui)**
3.  👩‍💻 **Ce que ça change pour une équipe de développement**
4.  🚀 **À quoi il sert une fois déployé en production sur AKS**
5.  ✅ **Pourquoi ton prototype a du sens (même sans tout automatiser)**

***

## 1️⃣ Le problème réel (avant Backstage)

Avant ton prototype, une équipe de développement vit généralement ça :

*   Pour créer un nouveau service :
    *   demander un repo à l’équipe DevOps
    *   copier/coller une pipeline existante
    *   oublier des fichiers (README, catalog-info, docs)
    *   configurer Azure DevOps “à la main”
*   Chaque équipe fait un peu différemment
*   Personne n’a une vue globale :
    *   quels services existent ?
    *   qui en est responsable ?
    *   quel pipeline est utilisé ?
*   La plateforme (Azure DevOps, CI/CD, IAM) est **exposée directement aux développeurs**

👉 **Résultat** :

*   chaos progressif
*   dette organisationnelle
*   forte dépendance aux DevOps
*   mauvaise Developer Experience

***

## 2️⃣ Ce que fait concrètement TON prototype (aujourd’hui)

Ton prototype **n’est pas un outil de plus**.  
C’est une **couche de contrôle et d’orchestration** au-dessus des outils existants.

### Aujourd’hui, ton prototype permet :

✅ **Un point d’entrée unique**  
→ Backstage est **la porte d’entrée** pour les développeurs.

✅ **Un catalogue central (Software Catalog)**  
→ Tous les services sont visibles, documentés, reliés à :

*   un owner
*   un système
*   un repo
*   une pipeline

✅ **Des Golden Paths via le Scaffolder**  
→ Au lieu de dire “fais comme tu veux”, la plateforme dit :

> *“Voici COMMENT on crée un service chez nous.”*

✅ **Une standardisation automatique**  
Quand un développeur utilise un template :

*   le repo est créé avec la bonne structure
*   la pipeline est déjà là
*   la documentation est prête
*   le service est déjà enregistré dans le Catalog

👉 **Le développeur n’a rien à inventer.**

***

## 3️⃣ Ce que ça change pour une équipe de développement

### Avant ton IDP

*   le développeur doit comprendre :
    *   GitHub
    *   Azure DevOps
    *   CI/CD
    *   conventions internes
*   il fait des erreurs
*   il dépend de l’équipe plateforme

### Avec ton IDP

*   le développeur :
    *   va sur Backstage
    *   clique sur “Create”
    *   remplit un formulaire simple
*   la plateforme fait le reste

👉 **La complexité est déplacée** :

*   ❌ plus chez le développeur
*   ✅ dans la plateforme (là où elle doit être)

C’est exactement le **but du Platform Engineering**.

***

## 4️⃣ À quoi il sert une fois déployé en production sur AKS

Si tu déploies **ce même prototype sur AKS**, voici ce qu’il devient :

### 🔹 Pour les développeurs

*   Backstage devient **le portail officiel**
*   Plus besoin de connaître Azure DevOps en détail
*   Tous les services sont visibles au même endroit

### 🔹 Pour l’équipe plateforme / DevOps

*   Les règles sont codées une fois (templates)
*   Plus besoin de gérer des demandes manuelles
*   La gouvernance est **automatique**, pas bureaucratique

### 🔹 Pour l’entreprise

*   Vue claire sur :
    *   les services existants
    *   leur état (lifecycle)
    *   leurs responsables
*   Moins de chaos, plus de contrôle
*   Décisions plus faciles

👉 **Backstage devient le “cockpit” de l’ingénierie.**

***

## 5️⃣ Pourquoi ton prototype est pertinent (même s’il n’automatise pas tout)

⚠️ Point très important :  
Ton prototype **n’a pas besoin** de tout faire pour être utile.

Il ne :

*   remplace pas Azure DevOps ❌
*   remplace pas Terraform ❌

👉 **Il orchestre**.

C’est exactement ce que l’entreprise voulait évaluer :

> *“Est-ce que Backstage peut devenir la couche d’orchestration avant de continuer l’IaC ?”*

✅ **Réponse de ton prototype** :  
**Oui. Clairement.**

***

## ✅ Résumé en une phrase (clé)

> **Ton prototype sert à transformer une plateforme technique complexe  
> en une expérience simple, standardisée et gouvernée pour les développeurs.**

Ou encore plus simple :

> **Ce que ton prototype automatise, ce n’est pas l’infrastructure,  
> mais la manière dont les équipes travaillent avec elle.**

***
Je vais structurer la réponse exactement comme tu l’as demandé :

1.  ✅ **Une phrase de valeur pour la soutenance**
2.  ✅ **Un mapping clair “problème → solution” pour le mémoire**
3.  ✅ **Ce qui manque concrètement pour passer du PoC à la production sur AKS**

***

## 1️⃣ La valeur de ton prototype en **UNE phrase** (pour la soutenance)

Voici **la phrase clé**, que tu peux dire telle quelle à l’oral :

> **« Le prototype que j’ai développé sert à transformer des outils DevOps complexes comme Azure DevOps et GitHub en une plateforme simple et standardisée, afin que les équipes de développement puissent créer et gérer des services sans avoir à maîtriser l’infrastructure sous-jacente. »**

Version encore plus directe (si on te coupe la parole) :

> **« Mon prototype sert à standardiser la manière dont les équipes créent des services, en déplaçant la complexité des outils vers une plateforme centrale. »**

👉 Si quelqu’un comprend ça, **il a compris ton travail**.

***

## 2️⃣ Mapping clair **Problème → Solution** (pour ton mémoire)

### 🔴 Problème AVANT ton prototype

Dans une organisation classique :

*   Les développeurs doivent :
    *   créer eux‑mêmes des repositories,
    *   configurer des pipelines,
    *   connaître Azure DevOps, GitHub, CI/CD,
    *   respecter des conventions souvent mal documentées.
*   Chaque équipe fait différemment.
*   Les DevOps sont sollicités en permanence.
*   Il n’existe **aucune vue centrale** des services.

👉 **Problème central** :  
**La complexité de la plateforme est exposée directement aux développeurs.**

***

### 🟡 Ce que fait TON prototype (aujourd’hui)

Ton prototype introduit **une couche intermédiaire** : l’IDP (Backstage).

Concrètement, il apporte :

*   ✅ **Un point d’entrée unique** (Backstage)
*   ✅ **Un catalogue central** (Software Catalog)
*   ✅ **Des Golden Paths** (Scaffolder Templates)
*   ✅ **Une standardisation automatique**
*   ✅ **Une gouvernance visible** (owner, system, lifecycle)

👉 Le développeur **ne touche plus directement** :

*   ni à GitHub,
*   ni à Azure DevOps,
*   ni aux conventions internes.

Il remplit un formulaire → **la plateforme orchestre le reste**.

***

### 🟢 Solution APPORTÉE par ton prototype

| Problème                      | Solution apportée         |
| ----------------------------- | ------------------------- |
| Création manuelle de services | Templates Scaffolder      |
| Incohérence des projets       | Golden Paths              |
| Manque de visibilité          | Software Catalog          |
| Dépendance aux DevOps         | Automatisation plateforme |
| Mauvaise Developer Experience | Abstraction des outils    |

👉 **Tu n’automatises pas l’infrastructure.**  
👉 **Tu automatises la manière de travailler avec l’infrastructure.**

C’est ça, le **Platform Engineering**.

***

## 3️⃣ À quoi il sert **en production sur AKS** (très important)

Maintenant, la vraie question :  
**« Si on déploie ça en production sur AKS, à quoi ça sert vraiment ? »**

### 👩‍💻 Pour les développeurs

*   Backstage devient **le portail officiel**
*   Ils peuvent :
    *   créer un service,
    *   trouver la documentation,
    *   voir qui est responsable de quoi
*   **Sans connaître AKS, Azure DevOps, GitHub, IAM**

👉 Ils travaillent **plus vite**, **avec moins d’erreurs**.

***

### 🛠️ Pour l’équipe plateforme / DevOps

*   Les règles sont codées **une seule fois** dans les templates
*   Plus besoin de :
    *   créer des repos à la main
    *   corriger des pipelines mal faites
*   La gouvernance est **automatique**, pas bureaucratique

👉 L’équipe plateforme **reprend le contrôle** sans bloquer.

***

### 🏢 Pour l’entreprise

*   Vision claire de :
    *   tous les services existants
    *   leur état (lifecycle)
    *   leur propriétaire
*   Moins de chaos, plus de traçabilité
*   Décisions plus simples (audit, sécurité, évolution)

👉 Backstage devient **le cockpit de l’ingénierie**.

***

## 4️⃣ Ce qui manque pour passer du PoC à une vraie production AKS

Ton prototype est **juste**, mais **pas encore complet**.  
Et c’est **normal** pour un PoC.

### 🔧 Ce qu’il faudrait ajouter (concrètement)

1.  **Authentification production**
    *   Azure AD / Entra ID ou Keycloak stable
    *   RBAC réel (pas seulement visuel)

2.  **Déploiement AKS réel**
    *   Ingress
    *   TLS
    *   DNS
    *   Scaling
    *   Secrets (Key Vault)

3.  **Automatisation complète**
    *   Scaffolder qui :
        *   crée réellement les repos
        *   déclenche les pipelines
        *   provisionne l’infra (Terraform/Bicep)

4.  **Observabilité**
    *   Logs
    *   Monitoring
    *   Alertes

👉 **Mais attention** :  
Ces éléments **ne remettent pas en cause la valeur du PoC**.  
Ils sont la **suite logique**, pas un manque.

***

## ✅ Conclusion finale (très importante pour toi)

Si tu dois retenir **UNE chose** :

> **Ton prototype ne sert pas à montrer que tu sais configurer Backstage.  
> Il sert à montrer que tu sais transformer une organisation DevOps complexe en une plateforme simple, gouvernée et scalable.**

Et ça, **c’est exactement ce que l’entreprise cherche à évaluer**.

***
Dis‑moi ce que tu veux faire ensuite.

