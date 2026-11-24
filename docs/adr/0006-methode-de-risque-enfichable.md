# ADR 0006 — Méthode d'analyse de risque enfichable

- **Statut** : proposé
- **Date** : 2026-11-24

## Contexte

Le marché français attend EBIOS Risk Manager. D'autres contextes sont habitués à ISO/IEC 27005, et
certaines directions financières à une quantification de type FAIR. Les trois méthodes n'ont ni le
même vocabulaire, ni les mêmes objets, ni la même échelle.

## Décision

`risk` expose un port `RiskMethod`. Chaque méthode est un adaptateur qui définit ses objets, ses
échelles et son calcul. Le noyau de `risk` ne connaît que les concepts communs : objet de risque,
scénario, évaluation, traitement, décision.

EBIOS RM est le premier adaptateur livré.

## Conséquences

Positives : ajout d'une méthode sans refonte du modèle ; un même tenant peut appliquer des méthodes
différentes selon le périmètre, ce qui correspond à la pratique réelle des grandes organisations.

Négatives : le noyau commun doit rester volontairement pauvre, ce qui limite ce que `reporting` peut
agréger entre méthodes. Comparer un risque EBIOS et un risque FAIR n'a pas de sens, et l'outil ne
doit pas laisser croire le contraire — la restitution doit refuser l'agrégation inter-méthodes
plutôt que produire un chiffre dénué de signification.
