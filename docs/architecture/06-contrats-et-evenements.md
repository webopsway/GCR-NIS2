# 06 — Contrats, événements et règles de couplage

## 1. Règles de couplage entre services

Ces règles sont ce qui rend le découpage réel. Sans elles, un découpage en 26 services produit un
monolithe distribué : tous les inconvénients du distribué, aucun de ses avantages.

1. **Un agrégat a un seul service propriétaire.** Lui seul écrit. Les autres lisent par API ou par
   projection locale alimentée par événements.
2. **Aucune clé étrangère entre services**, même en base partagée. Référence par identifiant.
3. **Les dépendances synchrones descendent les plans** (voir `03`). Jamais de remontée synchrone.
   Un plan inférieur qui a besoin de parler à un plan supérieur émet un événement.
4. **Aucun cycle synchrone.** Un cycle de dépendance synchrone est un défaut bloquant, pas une dette.
5. **Un service ne lit jamais la base d'un autre service.** Y compris pour « juste une jointure de
   reporting ». `reporting` travaille sur ses propres projections.
6. **Les règles d'autorisation vivent dans `authorization`.** Aucun service ne réimplémente de
   logique de périmètre ; le mandat MSSP ne doit être interprété qu'en un seul endroit.

## 2. Événements de domaine structurants

Nommage au passé, intention métier, pas de notification technique anémique.

| Événement | Émetteur | Consommateurs principaux | Effet |
|---|---|---|---|
| `EntityQualified` | `scoping` | `assessment`, `reporting`, `regulatory-catalog` | Détermine les exigences opposables au tenant |
| `FrameworkVersionPublished` | `regulatory-catalog` | `assessment`, `control`, `crosswalk`, `reporting` | Déclenche l'analyse d'impact du changement de référentiel |
| `ControlImplementationChanged` | `control` | `assessment`, `reporting`, `risk` | Recalcul de couverture et de risque net |
| `TestFailed` | `control` | `remediation`, `notification`, `risk` | Ouvre un écart, notifie le propriétaire |
| `EvidenceExpired` | `evidence` | `reporting`, `notification`, `assessment` | Dégrade la couverture : sans ce signal, les tableaux de bord mentent avec le temps |
| `FindingOpened` / `FindingClosed` | `assessment` | `remediation`, `reporting` | Alimente le plan d'action |
| `RiskAccepted` | `remediation` | `governance`, `audit-trail`, `reporting` | Exige une trace de décision au bon niveau hiérarchique |
| `WaiverExpiring` | `remediation` | `notification`, `governance` | Évite les dérogations dormantes |
| `IncidentQualifiedAsSignificant` | `incident` | `regulatory-reporting`, `scheduler`, `notification` | **Démarre les horloges de l'article 23** |
| `SubmissionAcknowledged` | `regulatory-reporting` | `incident`, `audit-trail`, `reporting` | Preuve de dépôt dans les délais |
| `RegulatoryDeadlineApproaching` | `scheduler` | `notification`, `incident`, `reporting` | Escalade avant échéance |
| `RegulatoryDeadlineMissed` | `scheduler` | `governance`, `notification`, `audit-trail` | Événement à conserver : il est aussi une preuve, y compris défavorable |
| `MandateGranted` / `MandateRevoked` | `tenancy` | `authorization`, `audit-trail` | Révocation d'accès à propagation immédiate |
| `SupplierRiskChanged` | `third-party` | `risk`, `reporting`, `notification` | Propagation le long de la chaîne d'approvisionnement |
| `VulnerabilityMatchedToAsset` | `vulnerability` | `risk`, `remediation` | SLA de remédiation |
| `PolicyApproved` | `policy` | `governance`, `evidence`, `awareness` | Preuve d'approbation par l'organe de direction |

### Deux événements à ne pas négliger

**`EvidenceExpired`** est l'événement qui distingue un outil GRC utile d'une base documentaire. Sans
lui, un client atteint 90 % de conformité la première année et conserve ce chiffre indéfiniment
pendant que ses preuves pourrissent. La péremption doit dégrader le score, visiblement.

**`RegulatoryDeadlineMissed`** doit être conservé, pas corrigé. La tentation d'« annuler » l'événement
après rattrapage est forte. Il faut y résister : une plateforme de conformité qui permet d'effacer
la trace d'un délai manqué perd exactement la propriété pour laquelle on l'achète.

## 3. Garanties de livraison attendues

| Propriété | Choix | Justification |
|---|---|---|
| Livraison | Au moins une fois | Le « exactement une fois » coûte cher et n'est pas nécessaire ici |
| Consommation | **Idempotente, obligatoire** | Corollaire du point précédent ; une clé d'idempotence par événement |
| Ordre | Garanti par agrégat uniquement | L'ordre global est inutile et coûteux |
| Publication | **Transactionnelle (outbox)** | Un `TestFailed` perdu est un écart jamais ouvert, donc une non-conformité invisible |
| Rejouabilité | Requise sur les domaines à enjeu de preuve | Reconstruction des projections de `reporting` sans perte |

Le motif outbox est non négociable sur ce produit : la perte silencieuse d'un événement de domaine
ne produit pas une erreur visible, elle produit un tableau de bord faux. C'est la pire catégorie de
défaut pour un outil de conformité.

## 4. Style d'API

- **REST + OpenAPI** pour les API exposées (clients, intégrations, UI). Le public visé — DSI, RSSI,
  auditeurs, intégrateurs — attend du REST documenté, pas du GraphQL.
- **Contrats internes typés** entre services, générés depuis un schéma unique pour éviter la dérive
  entre producteur et consommateur.
- **Versionnement explicite** dès la v1 (`/v1/…`). Un client on-premise reste des mois en retard de
  version : l'absence de versionnement rend les montées de version impossibles à échelonner.
- **Pas de BFF spécifique par écran au départ.** Une passerelle d'API et une UI qui compose. Le BFF
  devient utile quand l'UI a plusieurs formes (web, mobile, portail auditeur), pas avant.

## 5. Pile technique retenue

Python et PostgreSQL, conformément à votre choix. Précisions utiles au découpage :

| Brique | Proposition | Justification |
|---|---|---|
| API | **FastAPI** | Typage et génération OpenAPI native, adaptés à des contrats inter-services stricts |
| Modèle de données | SQLAlchemy 2.x + Alembic | Contrôle fin du SQL, nécessaire pour la RLS et les requêtes de couverture |
| Validation | Pydantic v2 | Frontière entre données externes et agrégats métier |
| Tâches de fond | Worker + file durable | Les horloges réglementaires exigent la persistance ; **un ordonnanceur en mémoire est disqualifié** |
| Moteur de règles | Tables de décision évaluées | Pour le scoping et l'applicabilité. **Éviter un DSL maison** : il devient vite impossible à auditer |
| Base | PostgreSQL 16+ | RLS, JSONB pour le contenu réglementaire, `tstzrange` pour les périodes de validité |
| Front | À décider séparément | Hors périmètre de ce découpage |

> Django plutôt que FastAPI serait défendable pour l'administration et les permissions prêtes à
> l'emploi. Je recommande FastAPI parce que le produit est d'abord une API multi-services avec des
> contrats stricts, et que l'ORM et l'admin de Django poussent vers un modèle de données centralisé
> — exactement ce que ce découpage cherche à éviter. Si l'équipe a une forte culture Django, ce choix
> se discute : il est moins coûteux qu'une équipe qui lutte contre son outillage.
