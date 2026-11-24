# ADR 0005 — Séparer la gestion d'incident de la notification réglementaire

- **Statut** : proposé
- **Date** : 2026-11-24

## Contexte

L'article 23 de la directive impose une alerte précoce sous 24 h, une notification sous 72 h, un
rapport intermédiaire sur demande et un rapport final sous un mois. Le canal, le format et le
contenu exigé dépendent de la juridiction et de la version de la réglementation nationale.

## Décision

Deux contextes distincts :

- `incident` : cycle de vie métier, qualification du caractère important, chronologie, impacts,
  retour d'expérience. Invariant stable.
- `regulatory-reporting` : production et transmission des notifications, preuve d'envoi et accusés
  de réception. Un adaptateur par juridiction derrière un port
  `AuthoritySubmissionGateway`.

La preuve de dépôt est un agrégat propre à `regulatory-reporting`, conservé indépendamment du cycle
de vie de l'incident.

## Conséquences

Positives : l'évolution d'un formulaire d'autorité ne touche pas la gestion d'incident ; l'ajout
d'une juridiction est un adaptateur ; ce qui compte en litige — la preuve de transmission dans les
délais — a un propriétaire clair.

Négatives : une frontière de plus à franchir sur un parcours utilisateur qui se vit comme continu.
L'UI doit masquer cette séparation, ce qui demande un soin particulier sur ce parcours.

## Règle produit associée

Le mode par défaut est **brouillon soumis à validation humaine avant transmission**. La transmission
automatique reste une option que le client active explicitement. Une notification erronée ou hors
délai engage le client, pas l'éditeur — raison de plus pour ne pas la déclencher sans intervention
humaine par défaut.
