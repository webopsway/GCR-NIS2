# 04 — Modèle de données pivot

## 1. Le graphe central

Tout l'intérêt d'un GRC multi-référentiels tient dans un seul chemin de données :

```
Requirement (catalogue, partagé, versionné)
     ▲  many-to-many
     │
  Control (tenant) ──── Test ──── TestResult
     │                              │
     │                              ▼
     └──────────────────────────► Evidence ──► Document
     │
     ├── mitige ──► Risk ──► Scenario ──► Asset
     │
     └── écart ──► Finding ──► Action (plan de remédiation)
```

Lu dans l'autre sens, c'est la réponse à la question d'un auditeur :

> « Montrez-moi comment vous satisfaites l'objectif 17. »
> → exigence → contrôles rattachés → tests exécutés → résultats → preuves horodatées → écarts
> ouverts → actions en cours avec échéances.

Si ce chemin n'est pas traversable en une requête, l'outil ne sert pas son objet. C'est le critère
d'acceptation technique le plus important du produit.

## 2. Tables pivots essentielles

### `control_requirement` — la table qui porte la valeur du produit

```sql
CREATE TABLE control_requirement (
    tenant_id          uuid        NOT NULL,
    control_id         uuid        NOT NULL,
    framework_version  text        NOT NULL,  -- ex. 'RECYF:v2.5-2026-03-17'
    requirement_ref    text        NOT NULL,  -- ex. 'OBJ-17.3'
    coverage           numeric(3,2) NOT NULL CHECK (coverage BETWEEN 0 AND 1),
    rationale          text        NOT NULL CHECK (length(trim(rationale)) > 0),
    asserted_by        uuid        NOT NULL,
    asserted_at        timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, control_id, framework_version, requirement_ref)
);
```

Trois choix à noter :

- **`framework_version` fait partie de la clé.** Le même contrôle peut couvrir une exigence dans
  une version du référentiel et plus dans la suivante. Sans la version dans la clé, une mise à jour
  de référentiel détruit silencieusement l'historique de conformité.
- **`rationale` est `NOT NULL` avec contrainte de non-vacuité.** Contrainte métier appliquée au
  niveau du schéma, parce qu'un rattachement sans justification n'est pas défendable en audit et
  qu'aucune équipe ne remplit un champ facultatif.
- **`tenant_id` en tête de la clé primaire**, systématiquement, pour toutes les tables tenant. Voir
  `05-tenancy-et-deploiement.md`.

### `evidence` — fraîcheur et péremption

Une preuve n'est pas un fichier : c'est un fichier **avec une date de validité**. Un rapport de test
d'intrusion de 2023 ne prouve rien en 2026. Le modèle doit porter la péremption, et le calcul du
score de conformité doit en tenir compte.

```sql
CREATE TABLE evidence (
    tenant_id      uuid        NOT NULL,
    evidence_id    uuid        NOT NULL,
    document_id    uuid,                      -- référence vers le service document
    collected_at   timestamptz NOT NULL,
    valid_until    timestamptz,               -- NULL = sans péremption, à justifier
    source         text        NOT NULL,      -- MANUAL | CONNECTOR:<nom> | TEST:<id>
    integrity_hash bytea       NOT NULL,
    PRIMARY KEY (tenant_id, evidence_id)
);
```

Une preuve issue d'un connecteur automatique et une preuve téléversée à la main n'ont pas la même
force probante. Le champ `source` doit être exploité dans la restitution, pas seulement stocké : un
tableau de bord affichant 95 % de conformité sur des preuves intégralement manuelles et périmées est
un tableau de bord qui trompe son lecteur.

## 3. Le score de conformité : décision à prendre explicitement

C'est le sujet qui provoque le plus de dette dans les outils GRC. Avant la première ligne de code :

1. **Le score est-il calculé ou stocké ?** Recommandation : **calculé** à partir des faits
   (contrôles, tests, preuves, écarts) et **matérialisé** dans `reporting` comme projection
   horodatée. Un score stocké comme champ mutable devient incohérent avec les faits et personne ne
   sait plus lequel croire.
2. **Une exigence partiellement couverte compte-t-elle ?** Il faut un choix tranché et documenté.
   La solution la plus défendable en audit : afficher séparément « couvert », « partiellement
   couvert », « non couvert », et ne jamais produire un pourcentage unique sans son détail.
3. **Que vaut une exigence non applicable ?** Elle sort du dénominateur, et la raison de son
   exclusion doit être tracée. Sinon le score se pilote en déclarant des exigences non applicables.

> Il n'existe pas de méthode de scoring normative imposée par NIS 2 à ma connaissance. Ce qui précède
> relève du choix produit, pas d'une obligation réglementaire — à présenter comme tel aux clients,
> et à ne pas afficher comme un « taux de conformité NIS 2 » officiel, formulation qui expose
> l'éditeur.

## 4. Historisation : reconstituer un état passé

Exigence NFR-4. Deux approches, à ne pas mélanger :

| | Table d'historique (`*_history`) | Journal d'événements de domaine |
|---|---|---|
| Coût initial | Faible | Moyen à élevé |
| Reconstitution | Par table, laborieuse | Native |
| Explication du changement | Perdue, ou dans un champ libre | Portée par l'intention de l'événement |
| Adapté à | Référentiels, actifs, politiques | Conformité, incidents, gouvernance |

Recommandation, détaillée dans l'ADR 0004 : **journal d'événements sur les domaines à enjeu de
preuve** (`control`, `assessment`, `evidence`, `incident`, `governance`, `remediation`), **tables
d'historique classiques ailleurs**. Généraliser l'event sourcing à toute l'application est un coût
que ce produit n'a pas besoin de payer sur les actifs ou les référentiels.

## 5. Identifiants et références inter-services

- **Identifiants internes : UUID v7** (ordonnés dans le temps, donc bons voisins de l'index B-tree,
  contrairement à l'UUID v4, et sans fuite de volumétrie contrairement aux séquences).
- **Les références croisées entre services sont des identifiants, jamais des clés étrangères.** Pas
  de `FOREIGN KEY` entre schémas de services distincts, même tant que tout cohabite dans la même
  instance PostgreSQL. C'est la discipline qui rend l'extraction ultérieure possible ; la relâcher
  « juste pour cette table » annule la propriété.
- **Les exigences se référencent par `framework_version` + `requirement_ref` textuels stables**, pas
  par UUID de ligne de catalogue. Un paquet de contenu réimporté ne doit pas invalider les
  rattachements existants.
