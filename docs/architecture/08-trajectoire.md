# 08 — Trajectoire de livraison et regroupement de déploiement

## 1. Regroupement des 26 contextes en unités déployables

Le découpage logique de `03` reste la référence. Voici comment le déployer sans payer d'emblée le
coût de 26 artefacts.

### Phase 1 — 4 unités déployables

| Unité | Contextes regroupés | Pourquoi ce regroupement |
|---|---|---|
| `platform` | identity, tenancy, authorization, audit-trail, document, workflow, notification, key-management, scheduler | Socle stable, change peu, aucun intérêt à le fragmenter tôt |
| `regulatory` | regulatory-catalog, crosswalk, scoping | Même cycle de vie : celui des publications réglementaires. Peut rester en lecture seule et se mettre en cache agressivement |
| `compliance` | governance, policy, assessment, control, evidence, remediation, asset, risk, third-party, vulnerability, awareness | Le cœur métier, là où les frontières sont encore incertaines. **À garder dans un même binaire jusqu'au premier client en production** |
| `operations` | incident, regulatory-reporting, continuity, exercise, reporting, integration-hub | Contraintes de disponibilité et de latence différentes du cœur métier |

Discipline à tenir dans `compliance` : un module par contexte, dépendances explicites, aucun import
croisé de modèle, communication par interfaces. Sans cette discipline, l'unité devient un monolithe
enchevêtré et le regroupement coûte au lieu de rapporter.

### Phase 2 — extractions quand un besoin réel l'impose

Candidats à l'extraction, par ordre de probabilité :

1. **`integration-hub`** — pour l'isolation de sécurité. Raison la plus légitime et la plus précoce.
2. **`regulatory-reporting`** — rythme de changement propre, risque juridique, besoin de
   disponibilité renforcée pendant un incident.
3. **`reporting`** — dimensionnement en lecture, requêtes lourdes, consolidation de portefeuille.
4. **`regulatory-catalog`** — si vous commercialisez le contenu réglementaire séparément.
5. **`vulnerability`** — si la volumétrie d'ingestion devient significative.

Critère d'extraction, à écrire dans la définition de fin de phase 1 : **on extrait un contexte quand
on peut nommer la contrainte mesurée qui l'impose** (volumétrie, isolation exigée par un client,
cadence d'équipe, obligation de souveraineté), jamais par principe d'architecture.

## 2. Séquence fonctionnelle

### Lot 1 — Socle et qualification *(fondation)*
- `platform` : tenants, utilisateurs, autorisation, journal d'audit, documents.
- `regulatory` : méta-modèle, import de paquet signé, **contenu NIS 2 articles 20 à 23**.
- `scoping` : moteur de qualification EE/EI avec explication.
- **Démontrable** : une entité se qualifie et obtient la liste des exigences qui lui sont opposables.

### Lot 2 — Conformité et preuve *(première valeur réelle)*
- `control`, `evidence`, `assessment`, `remediation`.
- Le chemin complet exigence → contrôle → test → preuve → écart → action.
- **Démontrable** : le chemin d'audit de `04-modele-pivot.md` est traversable de bout en bout.

### Lot 3 — Incidents et notification *(le lot à plus fort enjeu)*
- `incident`, `regulatory-reporting`, `scheduler` avec les horloges de l'article 23.
- Adaptateur pour l'autorité française ; port générique pour les autres juridictions.
- **Démontrable** : un incident qualifié déclenche les horloges 24 h / 72 h / 1 mois, produit les
  notifications et conserve la preuve de dépôt.

> ⚠️ Lot à ne pas livrer sans revue juridique. Une notification mal formée ou hors délai engage le
> client. Le mode « brouillon validé par un humain avant transmission » doit être le mode par
> défaut, et la transmission automatique une option explicitement activée par le client.

### Lot 4 — Risques, actifs et chaîne d'approvisionnement
- `asset`, `risk` avec EBIOS RM comme première méthode, `third-party`, `vulnerability`.
- Premiers connecteurs CMDB et scanner de vulnérabilités.

### Lot 5 — Multi-référentiels
- `crosswalk` et le contenu ISO/IEC 27001 et 27002, puis DORA, RGPD, NIST CSF.
- **Démontrable** : un contrôle implémenté une fois satisfait des exigences de plusieurs
  référentiels, avec justification du rattachement.

### Lot 6 — Continuité, culture, consolidation
- `continuity`, `exercise`, `awareness`, `reporting` avec la vue de portefeuille MSSP.

### Lot 7 — Industrialisation on-premise
- Paquet de déploiement, import de contenu hors ligne, BYOK, agent collecteur.

> Je ne chiffre pas ces lots en charge ou en délai : sans connaître la taille et la composition de
> l'équipe, toute estimation que je donnerais serait inventée. La séquence et les dépendances entre
> lots, en revanche, sont solides et c'est là qu'est l'information utile.

## 3. Pourquoi cet ordre

- **`scoping` avant tout** : sans qualification, l'application ne sait pas quelles exigences
  afficher. C'est aussi la fonction la plus facile à démontrer commercialement et la moins coûteuse
  à construire.
- **Le lot 2 avant le lot 3** : la notification d'incident est le lot à plus fort risque juridique.
  Elle se construit mieux sur un socle de preuve et de journalisation déjà éprouvé.
- **Le crosswalk en lot 5, pas plus tôt** : c'est un multiplicateur de valeur, inutile tant que le
  chemin de conformité mono-référentiel n'est pas solide. Le construire tôt produit un graphe de
  correspondances sur un modèle de contrôles encore instable.
- **L'on-premise en dernier, mais ses contraintes dès le lot 1.** On ne livre pas l'on-premise tôt,
  mais on s'interdit dès le départ ce qui le rendrait impossible — voir le tableau d'interdits de
  `05-tenancy-et-deploiement.md`. C'est la différence entre reporter une livraison et se condamner à
  une réécriture.

## 4. Arborescence de dépôt proposée

Monorepo, un package Python par contexte, regroupement de déploiement par composition :

```
GCR-NIS2/
├── docs/
│   ├── architecture/            ce découpage
│   ├── adr/                     décisions d'architecture
│   └── references/              sources réglementaires, à vérifier et tenir à jour
├── contracts/
│   ├── openapi/                 contrats d'API par contexte
│   └── events/                  schémas des événements de domaine
├── regulatory-content/          contenu réglementaire versionné, signé
│   ├── nis2/2022-12-27/
│   ├── recyf/                   ⚠️ version à confirmer sur la source ANSSI
│   ├── iso27001/2022/
│   └── crosswalk/
├── libs/
│   ├── gcr-kernel/              types partagés, tenant_id, erreurs, horloge
│   ├── gcr-eventbus/            outbox, publication, consommation idempotente
│   ├── gcr-authz/               client du point de décision d'autorisation
│   └── gcr-testing/             fixtures, builders, jeux de données de conformité
├── services/
│   ├── platform/                unité de déploiement (phase 1)
│   │   ├── identity/
│   │   ├── tenancy/
│   │   ├── authorization/
│   │   ├── audit_trail/
│   │   └── ...
│   ├── regulatory/
│   ├── compliance/
│   └── operations/
├── deploy/
│   ├── saas/
│   └── onprem/                  paquet mono-tenant, import hors ligne
└── tools/
    ├── content-signer/          signature des paquets de contenu réglementaire
    └── content-linter/          validation du contenu avant publication
```

Deux points sur cette arborescence :

- **`regulatory-content/` est versionné dans le dépôt mais déployé séparément du code.** C'est ce qui
  permet de publier une mise à jour de référentiel sans livrer l'application — propriété
  indispensable quand le texte national n'est pas stabilisé.
- **`tools/content-linter/` n'est pas un confort.** Une erreur dans le contenu réglementaire se
  propage à tous les tenants et produit des évaluations fausses. Le contenu mérite une chaîne de
  validation aussi stricte que le code : schéma, références croisées, complétude des `rationale`,
  cohérence des conditions d'applicabilité.

## 5. Les trois risques majeurs du projet

| Risque | Pourquoi il est sérieux | Atténuation retenue |
|---|---|---|
| **Le référentiel national n'est pas stabilisé** | La loi française n'est pas promulguée et le ReCyF est présenté comme un document de travail. Du contenu codé en dur serait à reprendre | Contenu comme donnée versionnée et signée ; aucun article en dur dans le code (ADR 0002) |
| **Le scoring de conformité devient incontestable alors qu'il est conventionnel** | Un client construit son rapport au conseil d'administration sur un pourcentage dont la méthode de calcul est un choix produit | Détail toujours affiché avec le score ; vocabulaire qui ne suggère pas une mesure officielle ; méthode documentée et versionnée |
| **La complexité distribuée précède le premier client** | 26 services sans trafic ni frontières éprouvées | Regroupement en 4 unités en phase 1, extraction sur contrainte mesurée |
