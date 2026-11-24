# 03 — Découpage en domaines et en services

## 1. Avertissement d'ingénierie, puis le découpage demandé

Vous avez retenu une cible microservices. Le découpage ci-dessous est complet et conçu pour cette
cible. Je dois cependant être clair sur un point avant de le dérouler :

**26 services déployables indépendamment sur un produit greenfield, c'est un piège bien documenté.**
Les frontières de domaine d'un GRC multi-référentiels ne sont pas connues avec certitude avant le
premier client en production. Démarrer distribué fige des frontières fausses derrière des appels
réseau, et déplacer ensuite un agrégat d'un service à l'autre coûte dix fois ce que coûte le
déplacer entre deux modules d'un même binaire. S'y ajoutent, à la charge d'une équipe qui n'a pas
encore de revenu : transactions distribuées, cohérence éventuelle dans les scores de conformité,
observabilité répartie, et une chaîne de livraison on-premise à 26 artefacts.

Ma recommandation : **garder exactement ce découpage comme découpage logique** — bounded contexts,
agrégats, contrats — et le déployer d'abord en un petit nombre d'unités (voir `08-trajectoire.md`),
chaque module respectant déjà ses frontières comme s'il était distant. L'extraction devient alors
une opération mécanique, à faire sur les services pour lesquels un besoin réel l'impose
(dimensionnement, isolation, rythme d'équipe, contrainte de souveraineté).

Le découpage logique est de toute façon le livrable qui a de la valeur. Le mode de déploiement est
une décision réversible ; les frontières de domaine ne le sont pas.

## 2. Vue d'ensemble : 6 plans, 26 bounded contexts

```
┌─ PLAN 6 ── PILOTAGE ─────────────────────────────────────────────────────┐
│  reporting · integration-hub                                             │
├─ PLAN 4 ── INCIDENTS & CRISE ──────┬─ PLAN 5 ── CULTURE ─────────────────┤
│  incident · regulatory-reporting   │  awareness                          │
│  continuity · exercise             │                                     │
├─ PLAN 3 ── RISQUES & ACTIFS ───────┴─────────────────────────────────────┤
│  asset · risk · third-party · vulnerability                              │
├─ PLAN 2 ── GOUVERNANCE & CONFORMITÉ ─────────────────────────────────────┤
│  governance · policy · assessment · control · evidence · remediation     │
├─ PLAN 1 ── NOYAU RÉGLEMENTAIRE ──────────────────────────────────────────┤
│  regulatory-catalog · crosswalk · scoping                                │
├─ PLAN 0 ── SOCLE PLATEFORME ─────────────────────────────────────────────┤
│  identity · tenancy · audit-trail · document · workflow · notification   │
│  key-management                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

Les dépendances descendent : un plan ne dépend jamais d'un plan supérieur. Toute remontée passe par
un événement, jamais par un appel synchrone. C'est l'invariant qui rend l'extraction possible plus
tard.

## 3. Plan 0 — Socle plateforme

| Service | Responsabilité | Agrégats possédés | Notes de conception |
|---|---|---|---|
| `identity` | Authentification, fédération OIDC/SAML, MFA, sessions, provisionnement SCIM | `User`, `Credential`, `Session`, `IdentityProvider` | En on-premise, déléguer à l'IdP du client plutôt que gérer des mots de passe. En SaaS, MFA obligatoire pour tous les rôles sans exception. |
| `tenancy` | Tenants, hiérarchie groupe/filiales, portefeuille MSSP, mandats, abonnements, résidence des données | `Tenant`, `OrgUnit`, `Mandate`, `Subscription` | Le `Mandate` (cabinet → client, périmètre, échéance, révocation) est l'agrégat le plus sensible du produit. Voir `05`. |
| `authorization` | Décision d'autorisation ABAC, politiques, périmètres effectifs | `Policy`, `RoleBinding` | Séparé de `identity` : l'authentification est déléguable, l'autorisation métier ne l'est pas. Point de décision unique ; aucun service ne réimplémente de règle d'accès. |
| `audit-trail` | Journal inaltérable des actions et décisions | `AuditEntry` (append-only, chaîné par empreinte) | Écriture seule depuis les autres services. Aucune API de suppression ni de mise à jour, même pour un administrateur. Export vérifiable hors ligne. |
| `document` | Stockage, versionnement, visa, rétention, coffre de preuves | `Document`, `DocumentVersion`, `RetentionPolicy` | Stockage objet compatible S3 ; chiffrement applicatif par tenant. Ne contient pas de logique métier de conformité. |
| `workflow` | Moteur de tâches, approbations, échéances, délégations, escalades | `ProcessInstance`, `Task`, `Approval` | Mutualisé par tous les domaines. Évite que `policy`, `remediation` et `assessment` écrivent chacun leur moteur d'approbation. |
| `notification` | Courriel, SMS, webhooks, connecteurs de messagerie ; préférences, escalades | `NotificationRequest`, `Template`, `Preference` | Transport uniquement. Ne décide jamais *quand* notifier : ce sont les domaines qui émettent. |
| `key-management` | Clés par tenant, rotation, BYOK/HYOK on-premise | `TenantKey`, `KeyRotation` | Isolé pour que la compromission d'un service métier ne donne pas accès aux clés. Interface alignée sur KMIP / un HSM pour les clients qui l'exigent. |
| `scheduler` | Horloges persistantes, échéances réglementaires, campagnes récurrentes | `ScheduledJob`, `RegulatoryClock` | **Critique.** Les délais de 24 h / 72 h / 1 mois de l'article 23 passent ici. Persistant, idempotent, rejouable, avec alerte en cas de retard de l'ordonnanceur lui-même. |

## 4. Plan 1 — Noyau réglementaire

| Service | Responsabilité | Agrégats possédés | Notes |
|---|---|---|---|
| `regulatory-catalog` | Référentiels, versions, exigences, MAC, spécifications de preuve | `Framework`, `FrameworkVersion`, `Requirement`, `Measure`, `EvidenceSpec` | Lecture seule pour les tenants. Contenu livré en paquets signés. Voir `02`. |
| `crosswalk` | Correspondances inter-référentiels, couverture, surcharges tenant | `CrosswalkEdge`, `TenantOverride` | Peut démarrer dans `regulatory-catalog` et être extrait quand le graphe devient volumineux. La séparation logique, elle, est utile tout de suite. |
| `scoping` | Qualification EE/EI, applicabilité, obligations d'enregistrement, juridictions | `EntityProfile`, `Qualification`, `ApplicabilityResolution` | Résultats horodatés et conservés. Produit toujours une explication, pas seulement un verdict. |

## 5. Plan 2 — Gouvernance et conformité

| Service | Responsabilité | Agrégats possédés | Couvre |
|---|---|---|---|
| `governance` | Organes de direction, approbations, revues de direction, RACI, registre des décisions, formation des dirigeants | `GoverningBody`, `Decision`, `ManagementReview`, `Accountability` | Art. 20 |
| `policy` | Corpus documentaire : rédaction, revue, approbation, publication, attestation de lecture, révision périodique | `Policy`, `PolicyVersion`, `Acknowledgement` | Art. 21.2 (a), (h) |
| `assessment` | Campagnes d'évaluation et d'audit, questionnaires dérivés du référentiel, maturité, écarts | `Campaign`, `Questionnaire`, `Response`, `Finding` | Art. 32/33 |
| `control` | Registre des contrôles, propriétaires, plan de test, exécution, efficacité | `Control`, `ControlImplementation`, `Test`, `TestResult` | Art. 21.2 (f) et support de (i), (j) |
| `evidence` | Collecte, horodatage, fraîcheur, péremption, rattachement multi-exigences, chaîne de custody | `Evidence`, `EvidenceLink`, `FreshnessRule` | Transverse — la preuve est ce qui rend tout le reste opposable |
| `remediation` | Écarts → plan d'action, responsables, échéances, acceptation de risque, dérogations | `ActionPlan`, `Action`, `Waiver`, `RiskAcceptance` | Transverse |

Deux précisions qui évitent des erreurs de modélisation fréquentes :

- **`Control` appartient au tenant ; `Requirement` appartient au catalogue.** Le lien est une
  relation many-to-many entre les deux. C'est précisément cette relation qui réalise la promesse
  « j'implémente un contrôle, je satisfais plusieurs exigences dans plusieurs référentiels ».
- **`Waiver` doit porter une date de fin obligatoire.** Une dérogation sans échéance est une
  non-conformité déguisée, et c'est ce que cherche un auditeur. Le schéma doit l'interdire, pas
  l'application.

## 6. Plan 3 — Risques et actifs

| Service | Responsabilité | Agrégats possédés | Couvre |
|---|---|---|---|
| `asset` | Actifs primordiaux (processus, données, services) et supports (SI, réseau, OT, cloud, SaaS), criticité DICT, dépendances, périmètres | `Asset`, `AssetDependency`, `Scope` | Art. 21.2 (i) |
| `risk` | Registre, scénarios, sources de risque, vraisemblance/gravité, risque brut et net, traitement, appétence, revues | `Risk`, `Scenario`, `Treatment`, `RiskAppetite` | Art. 21.2 (a) |
| `third-party` | Fournisseurs, criticité, évaluations, clauses cyber, certifications, dépendances en cascade, plan de sortie | `Supplier`, `SupplierAssessment`, `ContractClause`, `Dependency` | Art. 21.2 (d), art. 22 |
| `vulnerability` | Ingestion CVE/KEV/avis CERT, rapprochement avec les actifs, SLA de remédiation, divulgation coordonnée | `Vulnerability`, `AssetExposure`, `DisclosureCase` | Art. 21.2 (e) |

Notes :

- **La méthode de risque est enfichable** (voir ADR 0006). EBIOS Risk Manager est attendu par le
  marché français ; ISO/IEC 27005 et FAIR par d'autres. Coder une seule méthode dans le modèle rend
  les deux autres impossibles sans refonte. `risk` expose un port `RiskMethod` ; les méthodes sont
  des adaptateurs.
- **`third-party` doit modéliser la dépendance transitive.** L'article 21.2 (d) porte sur la chaîne,
  pas sur le fournisseur direct. Un modèle à un seul niveau ne répond pas à l'exigence. Prévoir un
  graphe, avec une profondeur de parcours bornée et un garde-fou contre les cycles.
- **`asset` est très souvent alimenté depuis l'extérieur** (CMDB, CSPM, inventaire SaaS). La saisie
  manuelle doit être le mode dégradé, pas le mode principal, sinon l'inventaire est périmé au bout
  d'un trimestre et tout l'édifice de conformité repose sur des données fausses.

## 7. Plan 4 — Incidents et crise

| Service | Responsabilité | Agrégats possédés | Couvre |
|---|---|---|---|
| `incident` | Cycle de vie, qualification du caractère important, chronologie, impacts, retour d'expérience | `Incident`, `Timeline`, `ImpactAssessment`, `Lesson` | Art. 21.2 (b) |
| `regulatory-reporting` | Alerte précoce, notification, rapport intermédiaire, rapport final ; information des destinataires de services ; preuve d'envoi et accusés | `Submission`, `SubmissionReceipt`, `AuthorityChannel` | Art. 23, art. 30 |
| `continuity` | BIA, RTO/RPO, PCA/PRA, stratégie de sauvegarde, gestion de crise, main courante | `BusinessImpactAnalysis`, `ContinuityPlan`, `CrisisCell` | Art. 21.2 (c) |
| `exercise` | Exercices de crise, tests de PRA, campagnes de simulation, preuve d'entraînement | `Exercise`, `ExerciseRun`, `Debrief` | Support de (c) et (g) |

Points d'attention :

- **La qualification « incident important » est le point le plus délicat du domaine.** Elle dépend
  de critères de la directive et, pour certains types d'entités, de seuils quantitatifs fixés par
  règlement d'exécution européen. Ces seuils doivent être du contenu versionné dans
  `regulatory-catalog`, interprété par `incident` — jamais des constantes dans le code.
  > Un règlement d'exécution de la Commission précise ces exigences et seuils pour une liste de
  > types d'entités numériques. Je ne cite pas sa référence de mémoire : **à récupérer sur EUR-Lex**
  > avant de saisir le contenu.
- **L'horloge démarre à la qualification, et son point de départ doit être justifiable.** Stocker
  l'instant de prise de connaissance, qui l'a établi, et sur quel fondement. En contrôle, c'est la
  première chose contestée.
- **`exercise` peut légitimement démarrer dans `continuity`.** Les deux partagent le vocabulaire et
  les acteurs ; l'extraire d'emblée coûte plus qu'il ne rapporte.

## 8. Plan 5 — Culture et sensibilisation

| Service | Responsabilité | Agrégats | Couvre |
|---|---|---|---|
| `awareness` | Plans de formation par population (dirigeants, administrateurs, tous), suivi de complétion, attestations | `TrainingPlan`, `Enrollment`, `Attestation` | Art. 21.2 (g), art. 20 (formation des dirigeants) |

Recommandation : **ne pas construire un LMS.** Ce service doit surtout savoir consommer les données
de complétion d'un LMS existant et en produire la preuve rattachée aux exigences. Construire de la
diffusion de contenu pédagogique ici est une dérive de périmètre coûteuse et sans différenciation.

## 9. Plan 6 — Pilotage et intégrations

| Service | Responsabilité | Agrégats | Notes |
|---|---|---|---|
| `reporting` | Tableaux de bord, indicateurs, consolidation groupe et portefeuille MSSP, liasse d'audit, rapport au conseil | Vues matérialisées ; aucun agrégat propre | Lit par événements et projections ; n'écrit jamais dans les domaines. |
| `integration-hub` | Connecteurs EDR, SIEM, CSPM, CMDB, IAM, scanners, ITSM, GED | `Connector`, `SyncRun`, `FieldMapping` | Un adaptateur par outil, derrière une couche de normalisation. Les données externes n'entrent jamais directement dans un agrégat métier. |

`integration-hub` est le service dont la surface d'attaque est la plus large : il détient des
identifiants vers les systèmes de sécurité du client. Il doit être isolé, ne jamais voir les clés
d'autres services, et ses secrets doivent résider dans `key-management`. En on-premise, le modèle
est un agent collecteur chez le client qui pousse vers la plateforme, jamais l'inverse.

## 10. Ce qui est volontairement exclu du périmètre

Dire ce que le produit ne fait pas est aussi utile que le découpage lui-même :

- **Pas de SIEM, pas d'EDR, pas de scanner.** La plateforme consomme leurs verdicts, elle ne les
  produit pas. Franchir cette limite transforme un produit GRC en produit de détection, avec un
  tout autre modèle de coûts et d'exigences temps réel.
- **Pas de conseil juridique automatisé.** L'application rend les textes opérationnels et trace la
  démonstration ; elle ne dit pas au client s'il est en infraction. La nuance est importante en
  termes de responsabilité de l'éditeur, et elle a des conséquences sur la formulation de chaque
  écran de verdict.
- **Pas de gestion de la conformité produit CRA.** Le CRA porte sur les produits comportant des
  éléments numériques, avec une logique de cycle de vie produit distincte. Le traiter comme un
  référentiel de plus dans `regulatory-catalog` est faisable ; en faire un domaine complet est un
  second produit.
