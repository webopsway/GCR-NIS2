# ADR 0003 — Un seul chemin de code pour le SaaS et l'on-premise, isolation par RLS

- **Statut** : proposé
- **Date** : 2026-11-24

## Contexte

Trois modes de livraison : SaaS mutualisé, on-premise mono-tenant, et pilotage multi-clients par un
cabinet ou un MSSP.

## Décision

- `tenant_id` sur toutes les tables métier, en tête de clé primaire.
- Isolation par Row-Level Security PostgreSQL, activée sans exception.
- L'on-premise mono-tenant est le **même schéma avec une seule ligne de tenant**. Aucune branche
  conditionnelle sur le mode de déploiement dans le code métier.
- Le cabinet n'est pas un tenant parent : il obtient un `Mandate` borné, daté et révocable sur le
  tenant du client.

## Conséquences

Positives : une seule base de code à maintenir et à auditer ; une requête qui oublie son filtre de
tenant ne retourne rien au lieu de fuir des données, ce qui rend le défaut visible en développement.

Négatives : la RLS ne couvre pas les chemins privilégiés — migrations, tâches de fond, vues
matérialisées de `reporting`. Ces chemins doivent être audités séparément et exécutés avec un rôle
dédié non `BYPASSRLS` dès que possible. Les très gros tenants peuvent exiger un placement dédié, à
traiter comme une option de localisation et non comme un second modèle de données.

## Alternatives écartées

- **Base par tenant** : isolation plus forte, mais multiplie les migrations par le nombre de clients
  et rend la consolidation de portefeuille MSSP coûteuse.
- **Deux produits distincts SaaS et on-premise** : divergence inévitable des chemins de code et
  doublement du coût de maintenance.
