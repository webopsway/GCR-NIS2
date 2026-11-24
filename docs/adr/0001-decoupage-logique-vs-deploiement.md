# ADR 0001 — Séparer le découpage logique du regroupement de déploiement

- **Statut** : proposé
- **Date** : 2026-11-24

## Contexte

La cible retenue est microservices. Le découpage de domaine aboutit à 26 bounded contexts. Le projet
est greenfield, sans client en production, et doit aussi se livrer en on-premise à des DSI de taille
moyenne.

## Décision

Conserver les 26 contextes comme **découpage logique** de référence (agrégats, contrats,
événements), et les regrouper en **4 unités déployables** en phase 1 : `platform`, `regulatory`,
`compliance`, `operations`.

Chaque module respecte dès le départ les règles de couplage de `06-contrats-et-evenements.md` comme
s'il était distant : pas de clé étrangère inter-contextes, pas de lecture de la base d'un autre
contexte, communication par interfaces.

Un contexte n'est extrait en service autonome que lorsqu'une **contrainte mesurée** l'impose :
volumétrie, isolation de sécurité exigée par un client, cadence d'équipe, obligation de
souveraineté.

## Conséquences

Positives : frontières de domaine corrigibles à faible coût tant qu'elles ne sont pas éprouvées ;
livraison on-premise réaliste ; pas de transaction distribuée tant que ce n'est pas nécessaire.

Négatives : la discipline modulaire doit être tenue par revue de code, car rien ne l'impose
techniquement dans un même binaire. C'est le risque principal de cette décision, et il doit être
traité comme tel — règle de revue explicite, et contrôle automatisé des imports entre modules.

Un découplage obtenu par le réseau est contraignant mais vérifiable ; un découplage obtenu par
convention est souple mais doit être outillé pour survivre.

## Alternatives écartées

- **26 services dès le départ** : fige des frontières non éprouvées derrière des appels réseau et
  charge l'équipe de complexité distribuée avant tout revenu.
- **Monolithe sans frontières internes** : rend l'extraction ultérieure aussi coûteuse qu'une
  réécriture, et incompatible avec l'isolation de sécurité exigée pour `integration-hub`.
