<div align="center">

<img src="assets/logo.png" alt="Volontariapp" width="180" />

# Volontariapp

**Connecter les bénévoles aux organisations.**

[LinkedIn — Victor Agahi](https://www.linkedin.com/in/victor-agahi-49b97b292/) · [LinkedIn — Clément Pasteau](https://www.linkedin.com/in/cl%C3%A9ment-pasteau-489615350/) · [LinkedIn — Victor Giroud](https://www.linkedin.com/in/victor-giroud-99096924b/)

[![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Rust](https://img.shields.io/badge/Rust-000000?style=for-the-badge&logo=rust&logoColor=white)](https://www.rust-lang.org/)
[![NestJS](https://img.shields.io/badge/NestJS-E0234E?style=for-the-badge&logo=nestjs&logoColor=white)](https://nestjs.com/)
[![React Native](https://img.shields.io/badge/React_Native-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)](https://reactnative.dev/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Neo4j](https://img.shields.io/badge/Neo4j-008CC1?style=for-the-badge&logo=neo4j&logoColor=white)](https://neo4j.com/)
[![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)

</div>

---

## Le Projet

Volontariapp est une plateforme mettant en relation les **bénévoles** et les **organisations** qui proposent des missions et événements. Les utilisateurs découvrent des missions, s'abonnent à des organisations et suivent un fil d'actualités ; les organisateurs créent des événements et gèrent les participations.

Le backend est conçu pour qu'**aucune requête utilisateur n'attende jamais un traitement métier lourd** : chaque action (créer un événement, publier un post, s'abonner) répond en quelques millisecondes, pendant qu'une machinerie asynchrone (Outbox, files BullMQ, Redis Streams) exécute le travail réel en arrière-plan et notifie le client par WebSocket une fois terminé.

## Architecture — Vue d'ensemble

> 📚 **Documentation complète (modèle C4)** : [`Volontariapp/docs`](https://github.com/Volontariapp/docs) — [C1 System Context](https://github.com/Volontariapp/docs/blob/main/C1-System-Context.md) · [C2 Containers](https://github.com/Volontariapp/docs/blob/main/C2-Containers.md) · [C3 Async Patterns & Flows](https://github.com/Volontariapp/docs/blob/main/C3-Async-Patterns-And-Flows.md) · [C4 Deployment & Infrastructure](https://github.com/Volontariapp/docs/blob/main/C4-Deployment-And-Infrastructure.md) · [Structure Monorepo & NPM](https://github.com/Volontariapp/docs/blob/main/Monorepo-Structure.md)

```mermaid
flowchart TD
    user(["Bénévole / Organisateur"]) -- "HTTPS & WSS" --> gw["API Gateway"]

    subgraph sync ["Synchrone (gRPC)"]
        gw --> ms_user["ms-user"]
        gw --> ms_event["ms-event"]
        gw --> ms_post["ms-post"]
        gw --> ms_social["ms-social (Neo4j)"]
    end

    gw -- "proxy" --> ws["ws-service"]

    subgraph async ["Asynchrone (Event-Driven)"]
        ob["outbox-runners"] --> redis[("Redis — Queues & Streams")]
        wk["workers-runners"] --> redis
        pp["post-processors-runner"] --> redis
        redis --> ws
    end

    ms_user -.-> ob
    ms_event -.-> ob
    ms_post -.-> ob
    ms_social -.-> ob
```

- **Microservices isolés par domaine** : chaque domaine métier (`ms-user`, `ms-event`, `ms-post`, `ms-social`, `ms-storage`) possède son propre dépôt, son propre processus et sa propre base de données. Aucun état n'est partagé entre domaines — seul **Redis** sert de bus de messages commun.
- **Transactional Outbox** : chaque écriture métier est enregistrée en même temps qu'un événement dans la table `event_outbox`, dans la même transaction SQL atomique, pour garantir qu'aucun message n'est jamais perdu.
- **Scatter-Gather** : une action déclenche plusieurs traitements indépendants en parallèle (géocodage, graphe social, etc.) ; le client n'est notifié en succès que lorsque toutes les étapes ont abouti — avec compensation automatique (rollback de sagas) en cas d'échec partiel.
- **GitOps** : l'intégralité de l'infrastructure Kubernetes (K3s) est pilotée depuis le dépôt [`deploy`](https://github.com/Volontariapp/deploy) via ArgoCD — Pod Security Admissions, Sealed Secrets, Network Policies en Default-Deny.

## Intelligence Artificielle & Serveur MCP (`mcp-meta-indexer`)

Pour permettre aux agents d'IA (Claude Code, Antigravity, Cursor) d'indexer et de naviguer dans l'intégralité des dépôts de l'organisation sans saturer leur contexte, le projet intègre un serveur **Model Context Protocol (MCP)** haute performance en **Rust** : [**`mcp-meta-indexer`**](https://github.com/Volontariapp/mcp-meta-indexer).

- 🔍 **`smart_search`** : Recherche plein texte combinée à un parser **Tree-sitter AST** (bloc cible + squelette architectural) avec repli flou (Fuzzy Matching Skim/Clangd).
- 🕸️ **`find_dependents`** : Graphe des imports en mémoire vive pour résoudre en $O(1)$ les dépendances d'un contrat ou d'un package partagé.
- ⚡ **`analyze_impact`** : Cartographie causale complète des flux asynchrones (Outbox, Redis Streams, BullMQ, Post-processors, triades de sagas et WebSockets).
- 🌐 **`analyze_grpc`** : Cartographie synchrone gRPC de bout en bout (`.proto`, contrats Gateway, interfaces NestJS et contrôleurs `@GrpcMethod`).
- 📚 **`search_docs`** : Recherche ciblée et extraction de sections conceptuelles dans le repo [`docs`](https://github.com/Volontariapp/docs) (~200 tokens par concept).

## Écosystème des Dépôts

### Documentation & Outils IA
| Dépôt | Rôle |
|---|---|
| [**docs**](https://github.com/Volontariapp/docs) | Référentiel officiel de la documentation d'architecture C4, ADRs et guides techniques. |
| [**mcp-meta-indexer**](https://github.com/Volontariapp/mcp-meta-indexer) | Serveur MCP Rust pour l'indexation AST et l'analyse causale de la codebase. |

### Points d'entrée (Gateways & Clients)
| Dépôt | Rôle |
|---|---|
| [**nativapp**](https://github.com/Volontariapp/nativapp) | Application mobile React Native (Expo SDK 54). |
| [**api-gateway**](https://github.com/Volontariapp/api-gateway) | Point d'entrée HTTP/REST unique, authentification JWT et routage gRPC. |
| [**ws-service**](https://github.com/Volontariapp/ws-service) | Microservice WebSocket temps réel (Pub/Sub, notifications, Scatter-Gather). |

### Microservices métiers (synchrones, gRPC)
| Dépôt | Rôle |
|---|---|
| [**ms-user**](https://github.com/Volontariapp/ms-user) | Identité, authentification et profils utilisateurs. |
| [**ms-event**](https://github.com/Volontariapp/ms-event) | Missions et événements — cœur de métier. |
| [**ms-post**](https://github.com/Volontariapp/ms-post) | Fil d'actualités et contenu. |
| [**ms-social**](https://github.com/Volontariapp/ms-social) | Graphe social (Neo4j) — abonnements et recommandations. |
| [**ms-storage**](https://github.com/Volontariapp/ms-storage) | Gestion des médias et stockage objet. |

### Infrastructure asynchrone (Event-Driven)
| Dépôt | Rôle |
|---|---|
| [**outbox-runners**](https://github.com/Volontariapp/outbox-runners) | Daemons d'extraction transactionnelle PostgreSQL → Redis Streams. |
| [**workers-runners**](https://github.com/Volontariapp/workers-runners) | Flotte de traitement des tâches d'arrière-plan (BullMQ). |
| [**post-processors-runner**](https://github.com/Volontariapp/post-processors-runner) | Consommation des Redis Streams, clôture des Sagas et notifications. |

### Outillage & Déploiement
| Dépôt | Rôle |
|---|---|
| [**npm-packages**](https://github.com/Volontariapp/npm-packages) | Monorepo des librairies partagées `@volontariapp/*` (domaines, contrats, shared). |
| [**proto-registry**](https://github.com/Volontariapp/proto-registry) | Registre unique des contrats Protocol Buffers (SSOT gRPC). |
| [**deploy**](https://github.com/Volontariapp/deploy) | Source de vérité GitOps (Kubernetes, K3s, ArgoCD, sécurité infra). |
| [**ci-tools**](https://github.com/Volontariapp/ci-tools) | Workflows et actions GitHub Actions partagés pour les pipelines CI/CD. |
| [**changelog-checker**](https://github.com/Volontariapp/changelog-checker) | Validation automatisée des versions et changelogs. |
| [**meta**](https://github.com/Volontariapp/meta) | Méta-dépôt d'orchestration — scripts d'onboarding et environnement local unifié. |

## Pour commencer

Le point d'entrée pour cloner et lancer l'écosystème complet en local se trouve dans le dépôt [**meta**](https://github.com/Volontariapp/meta) :

```bash
git clone git@github.com:Volontariapp/meta.git
cd meta
bash root.sh   # Full Setup — clone, configure et installe tous les sous-dépôts
```

---

<div align="center">

© EPITA — Projet pédagogique. Tous droits réservés.

</div>
