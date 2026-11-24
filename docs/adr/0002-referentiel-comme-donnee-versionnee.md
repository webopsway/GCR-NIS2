# ADR 0002 — Le contenu réglementaire est une donnée versionnée et signée, pas du code

- **Statut** : proposé
- **Date** : 2026-11-24

## Contexte

L'application doit porter NIS 2, le référentiel national français, ISO/IEC 27001 et 27002, DORA, le
RGPD, le NIST CSF et le CRA.

Au 24 novembre 2026, d'après les sources consultées (voir `docs/references/sources.md`) : la loi
française de transposition n'est pas promulguée, et le référentiel diffusé par l'ANSSI est présenté
comme un document de travail. Le contenu réglementaire va donc changer **pendant** le développement,
puis continuer à changer en production.

## Décision

Le contenu réglementaire est un **artefact de données** : paquets versionnés, immuables une fois
publiés, signés par l'éditeur, importables hors ligne, déployables indépendamment du code
applicatif.

Aucun article, aucun objectif, aucun seuil de qualification n'est écrit dans le code Python. Les
règles d'applicabilité et de qualification sont des tables de décision évaluées, livrées avec le
contenu.

## Conséquences

Positives : une mise à jour réglementaire ne demande pas de livraison applicative ; l'on-premise
hors ligne est servi par le même mécanisme ; l'historique permet de démontrer au regard de quelle
version un tenant a été évalué.

Négatives : le méta-modèle doit être assez général pour absorber des référentiels de structures
différentes, ce qui est plus coûteux qu'un modèle taillé pour NIS 2. Une chaîne de validation du
contenu (`tools/content-linter/`) devient indispensable, car une erreur de contenu se propage à tous
les tenants.

## Note de fiabilité

Le statut de la transposition française et la version du référentiel ANSSI proviennent de sources
secondaires qui se contredisent sur plusieurs points. **À vérifier sur Légifrance, sur le dossier
législatif de l'Assemblée nationale et sur la publication primaire de l'ANSSI avant toute saisie de
contenu.** Cette incertitude est précisément ce qui justifie la décision.
