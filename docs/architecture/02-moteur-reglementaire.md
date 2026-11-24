# 02 — Moteur réglementaire : le noyau du produit

## 1. Le découpage qui compte le plus

La décision structurante de toute l'application est la **séparation entre le savoir réglementaire et
l'état de conformité du client** :

| | Savoir réglementaire | État de conformité |
|---|---|---|
| Nature | Partagé par tous les tenants | Propre à un tenant |
| Rythme de changement | Lent, par publication officielle | Quotidien |
| Qui écrit | L'éditeur (vous) | Le client |
| Modèle | Immuable, versionné, publié | Mutable, historisé |
| Confidentialité | Public ou semi-public | Hautement sensible |
| Service | `regulatory-catalog` | `assessment`, `control`, `evidence`… |

Confondre les deux est l'erreur classique des outils GRC : on crée une table `exigences` par tenant,
on y copie le référentiel, et dix-huit mois plus tard la mise à jour d'ISO 27002 demande une
migration par client, impossible à auditer. **Le référentiel n'appartient pas au tenant. Le tenant
possède son positionnement au regard d'une version donnée du référentiel.**

## 2. Modèle générique multi-référentiels

Un seul méta-modèle doit porter NIS 2, le ReCyF, ISO/IEC 27001 et 27002, DORA, le RGPD, le NIST CSF
et le CRA. Le piège serait de modéliser NIS 2 puis de tordre le modèle pour les suivants.

```
Framework                 ex. "NIS2", "RECYF", "ISO27001", "DORA", "NIST-CSF"
  └── FrameworkVersion    ex. "NIS2:2022-12-27", "RECYF:v2.5-2026-03-17"
        └── Requirement   nœud hiérarchique : article, objectif, contrôle, clause
              ├── Measure            moyen acceptable de conformité (MAC)
              ├── EvidenceSpec       nature de preuve attendue, fraîcheur exigée
              └── Applicability      condition : catégorie EE/EI, secteur, taille, pays
```

Points de conception à ne pas rater :

- **`Requirement` est récursif.** Une hiérarchie à profondeur libre absorbe aussi bien
  `Art. 21.2.d` que `ISO 27001 A.5.19` ou `ReCyF Objectif 17 › MAC 17.3`. Imposer trois niveaux
  fixes casse au premier référentiel qui en a quatre.
- **`Applicability` est une expression évaluable, pas un booléen.** Les objectifs 16 à 20 du ReCyF
  ne s'appliquent qu'aux EE ; certaines exigences ne s'appliquent qu'à certains secteurs d'annexe.
  Cette condition doit être une donnée interprétée par le moteur de scoping, pas une règle en dur.
- **Une `FrameworkVersion` publiée est immuable.** Une correction produit une nouvelle version. Sans
  cela, un client contrôlé ne peut pas démontrer au regard de quel texte il a été évalué.
- **Le contenu est un artefact distribuable.** Paquet signé (contenu + signature de l'éditeur),
  importable hors ligne. C'est la seule façon de servir un on-premise sans accès Internet, et cela
  règle au passage la question de la confiance dans le contenu importé.

## 3. Le crosswalk : implémenter une fois, satisfaire N exigences

C'est la valeur économique de l'outil en multi-référentiels. Le client ne veut pas tenir quatre
registres de contrôles parallèles.

```
Crosswalk
  source_requirement_id   ex. NIS2 Art. 21.2.(c)
  target_requirement_id   ex. ISO 27001:2022 A.5.29
  relation                EQUIVALENT | COUVRE | COUVRE_PARTIELLEMENT | CONTRIBUE_A
  coverage_ratio          0.0 – 1.0
  rationale               justification textuelle, opposable en audit
  authored_by             éditeur | tenant
```

Trois règles tirées de l'expérience des outils qui échouent sur ce point :

1. **La relation est orientée et asymétrique.** « ISO A.5.29 couvre NIS2 21.2.c » n'implique pas la
   réciproque. Un graphe non orienté produit des scores de conformité faux, et un score de
   conformité faux dans un rapport au conseil d'administration est un incident produit.
2. **La justification est obligatoire.** Un mapping sans `rationale` n'est pas défendable devant un
   auditeur. Le champ ne doit pas être optionnel au niveau du schéma.
3. **Le tenant peut amender le crosswalk de l'éditeur, sans l'écraser.** Deux couches : mapping
   livré (immuable, versionné) et surcharges du tenant (traçables, avec auteur et date). Le score
   doit pouvoir être recalculé dans les deux configurations.

> Je ne fournis pas ici de table de correspondance NIS2 ↔ ISO 27001. Ces mappings existent dans des
> publications spécialisées, mais je n'ai pas de source vérifiée à vous citer, et une table de
> correspondance inventée serait exactement le genre de contenu qui se retrouve dans un rapport
> d'audit client. Le mapping initial doit être produit ou revu par un praticien, puis versionné
> comme le reste du contenu.

## 4. Moteur de scoping : qualification et applicabilité

Le service `scoping` répond à trois questions, dans cet ordre :

1. **Suis-je assujetti ?** Secteur (annexes I et II), taille, implantations, désignation éventuelle
   par l'autorité, cas d'assujettissement indépendant de la taille.
2. **Dans quelle catégorie ?** Entité essentielle ou importante — ce qui change le régime de
   supervision (ex ante / ex post), le plafond de sanction et, en France, le périmètre
   d'objectifs ReCyF applicables.
3. **Quelles exigences me sont opposables ?** Résolution de `Applicability` sur l'ensemble des
   versions de référentiels activées pour ce tenant.

Conception :

- **Les règles de qualification sont du contenu versionné, pas du code.** Même raison que pour le
  référentiel : la transposition française n'est pas stabilisée. Un moteur de règles déclaratif
  (table de décision évaluée, pas un DSL maison trop expressif) est préférable à des `if` en Python.
- **Le résultat de qualification est horodaté et conservé.** Un client qui franchit un seuil
  d'effectifs change de catégorie. L'historique de qualification est une preuve en soi.
- **Le moteur produit une explication, pas seulement un verdict.** « EE au titre du secteur énergie,
  annexe I, sur critère d'effectifs ≥ 250 » est utilisable ; « EE » ne l'est pas. L'explicabilité
  est ici une exigence métier.
- **Multi-pays dès la conception.** Une entité implantée dans plusieurs États membres relève de
  plusieurs transpositions. Le modèle doit porter la juridiction comme dimension, même si la v1 ne
  livre que la France. Ajouter la dimension plus tard est une migration lourde.

## 5. Pourquoi un service séparé pour la notification réglementaire

`incident` et `regulatory-reporting` sont deux services distincts, alors qu'une lecture rapide les
fusionnerait. La raison :

- Le cycle de vie d'un incident est un invariant métier stable : détection, qualification,
  confinement, résolution, retour d'expérience.
- Le canal, le format, le destinataire et le contenu exigé de la notification sont **propres à
  chaque juridiction et à chaque version de la réglementation**. En France, le portail de l'autorité
  nationale ; ailleurs, le CSIRT compétent, avec d'autres formulaires.

Les faire cohabiter dans un même service couple un invariant stable à la partie la plus instable et
la plus juridiquement risquée du produit. Chaque évolution d'un formulaire d'autorité
redéploierait la gestion d'incident.

En pratique, `regulatory-reporting` expose un port `AuthoritySubmissionGateway` avec un adaptateur
par juridiction, et conserve la **preuve d'envoi et l'accusé de réception** comme agrégat propre :
en cas de litige, ce qui compte n'est pas que l'incident ait été traité, mais que la notification
ait été transmise dans les délais, et qu'on puisse le prouver.
