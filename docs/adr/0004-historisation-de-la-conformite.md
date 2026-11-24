# ADR 0004 — Journal d'événements sur les domaines à enjeu de preuve uniquement

- **Statut** : proposé
- **Date** : 2026-11-24

## Contexte

Un contrôle réglementaire peut porter sur un état passé : quelles mesures étaient en place à telle
date, quelles preuves existaient, qui avait accepté quel risque. Les tables mutables ne permettent
pas de répondre.

## Décision

Journal d'événements de domaine, avec rejouabilité, sur : `control`, `assessment`, `evidence`,
`incident`, `governance`, `remediation`.

Tables d'historique classiques ailleurs : `asset`, `regulatory-catalog`, `policy`, `third-party`,
`risk`.

## Conséquences

Positives : reconstitution native de l'état de conformité à une date passée, avec l'intention du
changement et non seulement sa valeur avant/après. Les projections de `reporting` sont
reconstructibles.

Négatives : deux styles de persistance dans la même application, donc une frontière à documenter et
à tenir. Le coût cognitif est réel. Il est préférable au coût de généraliser l'event sourcing à des
domaines qui n'en ont pas besoin, comme l'inventaire d'actifs.

## Point de vigilance

L'événement `RegulatoryDeadlineMissed` doit être conservé et non corrigible. Permettre d'effacer la
trace d'un délai manqué ferait perdre à la plateforme la propriété pour laquelle elle est achetée.
