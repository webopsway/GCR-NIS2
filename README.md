# GCR-NIS2

Plateforme de gouvernance, conformité et gestion des risques pour les entités assujetties à la
directive NIS 2 (UE) 2022/2555, et pour les cabinets et MSSP pilotant la conformité de plusieurs
entités.

## État du projet

**Phase de conception.** Ce dépôt contient actuellement le découpage architectural ; aucun code
applicatif n'est encore écrit.

→ **[Commencer par docs/README.md](docs/README.md)**

## En bref

- 26 bounded contexts en 6 plans, regroupés en 4 unités déployables pour la phase 1.
- Séparation stricte entre le savoir réglementaire (partagé, versionné, immuable) et l'état de
  conformité du tenant (propre, historisé).
- Un seul chemin de code pour le SaaS mutualisé et l'on-premise mono-tenant ; isolation par RLS
  PostgreSQL.
- Contenu réglementaire livré en paquets de données signés, déployables sans livraison applicative —
  parce que la transposition nationale n'est pas stabilisée.
- Python (FastAPI) + PostgreSQL.

## Avertissement sur le contenu réglementaire

Les références réglementaires de cette documentation sont des **points d'entrée de travail, pas une
source de droit**. Les articles, seuils et délais doivent être vérifiés sur les sources primaires
(EUR-Lex, Légifrance, publications de l'autorité nationale) avant toute implémentation.

Voir [docs/references/sources.md](docs/references/sources.md), qui distingue explicitement ce qui a
été vérifié de ce qui ne l'a pas été.
