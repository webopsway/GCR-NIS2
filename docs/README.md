# Documentation GCR-NIS2

Découpage d'une plateforme de gouvernance, conformité et gestion des risques pour les entités
assujetties à la directive NIS 2, et pour les cabinets et MSSP qui pilotent plusieurs entités.

## Paramètres retenus

| Paramètre | Choix |
|---|---|
| Déploiement | SaaS souverain mutualisé + on-premise mono-tenant + mode cabinet/MSSP multi-clients |
| Architecture cible | Microservices (26 bounded contexts), regroupés en 4 unités déployables en phase 1 |
| Référentiels | NIS 2 art. 20 à 23, transposition française, ISO/IEC 27001 et 27002, DORA, RGPD, NIST CSF, CRA |
| Pile | Python (FastAPI) + PostgreSQL |

## Plan de lecture

| Document | Contenu | À lire si |
|---|---|---|
| [01 — Périmètre et exigences](architecture/01-perimetre-et-exigences.md) | Obligations NIS 2 à couvrir, exigences non fonctionnelles, contraintes réglementaires | Vous découvrez le projet |
| [02 — Moteur réglementaire](architecture/02-moteur-reglementaire.md) | Méta-modèle multi-référentiels, crosswalk, moteur de qualification | **Le cœur du produit. À lire en premier après 01** |
| [03 — Découpage en services](architecture/03-decoupage-services.md) | 26 bounded contexts en 6 plans, responsabilités, agrégats | Vous cherchez le découpage lui-même |
| [04 — Modèle pivot](architecture/04-modele-pivot.md) | Graphe central, tables pivots, scoring, historisation | Vous modélisez la base |
| [05 — Tenancy et déploiement](architecture/05-tenancy-et-deploiement.md) | RLS, mandats MSSP, contraintes on-premise | Vous travaillez sur l'isolation ou le packaging |
| [06 — Contrats et événements](architecture/06-contrats-et-evenements.md) | Règles de couplage, événements de domaine, pile technique | Vous implémentez une frontière entre services |
| [07 — Sécurité de la plateforme](architecture/07-securite-plateforme.md) | Pourquoi cette application est une cible de grande valeur | Avant toute décision d'infrastructure |
| [08 — Trajectoire](architecture/08-trajectoire.md) | Regroupement de déploiement, séquence de lots, arborescence de dépôt | Vous planifiez la livraison |

## Décisions d'architecture

| ADR | Décision |
|---|---|
| [0001](adr/0001-decoupage-logique-vs-deploiement.md) | Séparer le découpage logique du regroupement de déploiement |
| [0002](adr/0002-referentiel-comme-donnee-versionnee.md) | Le contenu réglementaire est une donnée versionnée et signée, pas du code |
| [0003](adr/0003-tenancy-rls-chemin-unique.md) | Un seul chemin de code SaaS et on-premise, isolation par RLS |
| [0004](adr/0004-historisation-de-la-conformite.md) | Journal d'événements sur les domaines à enjeu de preuve uniquement |
| [0005](adr/0005-separation-incident-notification.md) | Séparer la gestion d'incident de la notification réglementaire |
| [0006](adr/0006-methode-de-risque-enfichable.md) | Méthode d'analyse de risque enfichable |

## Fiabilité du contenu réglementaire

[docs/references/sources.md](references/sources.md) distingue ce qui est vérifié de ce qui ne l'est
pas, et fixe la règle : **aucune saisie de contenu réglementaire depuis une source secondaire.**

À lire avant d'écrire la première ligne de `regulatory-content/`.
