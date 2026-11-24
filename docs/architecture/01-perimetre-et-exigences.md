# 01 — Périmètre fonctionnel et exigences structurantes

## 1. Ce que l'application doit faire

GCR-NIS2 est une plateforme de **gouvernance, conformité et gestion des risques** destinée aux
entités assujetties à la directive NIS 2, ainsi qu'aux cabinets et MSSP qui pilotent la conformité
de plusieurs entités.

Elle doit permettre à une entité de :

1. **Savoir si elle est assujettie**, dans quelle catégorie, et à quelles obligations.
2. **Démontrer** qu'elle met en œuvre les mesures de gestion des risques exigées.
3. **Notifier** les incidents importants aux autorités dans les délais réglementaires.
4. **Prouver** tout ce qui précède lors d'un contrôle, avec des preuves horodatées et traçables.

Le point 4 est la contrainte dominante. Une plateforme GRC qui ne produit pas de preuve opposable
n'a pas d'intérêt réglementaire. Cela se traduit en architecture par trois propriétés non
négociables : journalisation inaltérable, historisation des états de conformité, et versionnement
du référentiel au regard duquel la conformité a été évaluée.

## 2. Obligations NIS 2 à couvrir

Référence : directive (UE) 2022/2555.

| Article | Obligation | Domaine applicatif |
|---|---|---|
| Art. 3 + Annexes I & II | Qualification entité essentielle (EE) / importante (EI) selon secteur et taille | Scoping |
| Art. 20 | Responsabilité des organes de direction : approbation des mesures, supervision, formation des dirigeants | Gouvernance |
| Art. 21.2 (a) | Politiques d'analyse des risques et de sécurité des SI | Risques, Politiques |
| Art. 21.2 (b) | Gestion des incidents | Incidents |
| Art. 21.2 (c) | Continuité d'activité, sauvegardes, reprise, gestion de crise | Continuité |
| Art. 21.2 (d) | Sécurité de la chaîne d'approvisionnement | Tiers |
| Art. 21.2 (e) | Sécurité de l'acquisition, du développement et de la maintenance ; gestion et divulgation des vulnérabilités | Vulnérabilités |
| Art. 21.2 (f) | Politiques et procédures d'évaluation de l'efficacité des mesures | Contrôles |
| Art. 21.2 (g) | Cyberhygiène de base et formation | Sensibilisation |
| Art. 21.2 (h) | Politiques de cryptographie et de chiffrement | Politiques, Contrôles |
| Art. 21.2 (i) | Sécurité RH, contrôle d'accès, gestion des actifs | Actifs, Contrôles |
| Art. 21.2 (j) | MFA ou authentification continue, communications sécurisées et d'urgence | Contrôles |
| Art. 22 | Évaluations coordonnées des risques des chaînes d'approvisionnement critiques (niveau UE) | Tiers |
| Art. 23 | Notification des incidents importants : alerte précoce, notification, rapport intermédiaire, rapport final ; information des destinataires de services | Incidents, Notification réglementaire |
| Art. 27 | Enregistrement auprès de l'autorité / registre ENISA pour certains types d'entités | Scoping |
| Art. 29 / 30 | Partage volontaire d'informations, notifications volontaires | Incidents |
| Art. 32 / 33 | Mesures de supervision : contrôles ex ante pour les EE, ex post pour les EI | Évaluation, Preuves |
| Art. 34 | Sanctions administratives | Pilotage du risque de sanction |

> ⚠️ **À verrouiller avec un juriste avant implémentation du moteur de qualification.**
> Les seuils de taille (effectifs, chiffre d'affaires, bilan), les exclusions sectorielles, les
> entités assujetties indépendamment de leur taille, et les montants exacts de sanctions de
> l'article 34 ne doivent pas être codés depuis une note d'architecture. Ils doivent être repris
> du texte consolidé sur EUR-Lex et de la transposition nationale applicable. Une erreur de seuil
> dans le moteur de qualification est une erreur à conséquence juridique directe pour le client.

### Délais de l'article 23

Les délais suivants constituent le cœur du domaine « incident » et doivent être modélisés comme des
objets de premier rang, pas comme des rappels d'agenda :

- **Alerte précoce** : 24 heures après la prise de connaissance de l'incident important.
- **Notification d'incident** : 72 heures après la prise de connaissance.
- **Rapport intermédiaire** : sur demande de l'autorité ou du CSIRT.
- **Rapport final** : un mois après la notification d'incident.

> Je suis confiant sur ces quatre délais, mais la définition exacte du point de départ
> (« prise de connaissance ») et le régime applicable aux incidents en cours au moment du rapport
> final sont à confirmer sur le texte. C'est le paramètre qui déclenche l'horloge : il mérite une
> lecture juridique explicite.

## 3. Contrainte spécifique : le référentiel national est un sujet mouvant

État au 24 novembre 2026, d'après des sources secondaires (voir `docs/references/sources.md`) :

- La transposition française — « loi relative à la résilience des infrastructures critiques et au
  renforcement de la cybersécurité », dite loi Résilience — **n'est pas promulguée**. Le texte a
  été adopté en première lecture à l'Assemblée nationale le 17 novembre 2026 ; une commission mixte
  paritaire reste nécessaire pour accorder les versions Sénat et Assemblée.
- La Commission européenne aurait saisi la Cour de justice à l'encontre de la France le 8 juillet
  2026 pour transposition incomplète.
- L'ANSSI diffuse un **Référentiel Cyber France (ReCyF)**, décrit par les sources consultées comme
  un document de travail non contraignant, structuré en 20 objectifs de sécurité : les objectifs 1
  à 15 formant un socle commun aux EE et EI, les objectifs 16 à 20 s'appliquant aux seules EE.
  Chaque objectif est assorti de « moyens acceptables de conformité » (MAC).

> ⚠️ Ces éléments proviennent de sources secondaires qui se contredisent sur plusieurs points :
> numéro de version du ReCyF, nombre exact d'objectifs, et caractère contraignant ou non du
> référentiel. **À vérifier sur Légifrance, sur le dossier législatif de l'Assemblée nationale et
> sur la publication primaire de l'ANSSI avant toute saisie de contenu réglementaire.**

**Conséquence architecturale directe, et elle est majeure :** le contenu réglementaire national ne
peut pas être codé en dur, ni même figé dans un schéma de base de données spécifique. Il change
pendant que l'application est développée. Le référentiel doit être un **artefact de données
versionné, signé et distribuable indépendamment du code** — y compris hors ligne pour les
déploiements on-premise. Voir `02-moteur-reglementaire.md` et `docs/adr/0002`.

## 4. Exigences non fonctionnelles structurantes

| # | Exigence | Origine | Impact sur le découpage |
|---|---|---|---|
| NFR-1 | Code unique déployable en SaaS mutualisé et en mono-tenant on-premise | Choix produit | Interdit les services managés propriétaires dans le noyau ; impose une abstraction infra |
| NFR-2 | Un cabinet/MSSP accède à N tenants clients sous mandat révocable et traçable | Choix produit | Modèle d'autorisation ABAC avec le mandat comme attribut ; pas de duplication de données |
| NFR-3 | Journal d'audit inaltérable et vérifiable | Art. 32/33 — contrôle | Service dédié, append-only, chaînage par empreinte |
| NFR-4 | Reconstitution de l'état de conformité à une date passée | Contrôle a posteriori | Historisation des décisions de conformité (voir ADR 0004) |
| NFR-5 | Souveraineté des données ; hébergement qualifiable | Clientèle régulée FR | PostgreSQL, stockage objet compatible S3, OIDC ; aucune dépendance cloud propriétaire |
| NFR-6 | Les horloges réglementaires survivent aux redémarrages et aux pannes | Art. 23 | Ordonnanceur persistant, pas de timer en mémoire |
| NFR-7 | Chiffrement par tenant, avec apport de clés client (BYOK) en on-premise | Clientèle régulée | Service de gestion de clés isolé ; chiffrement applicatif des champs sensibles |
| NFR-8 | La plateforme doit pouvoir passer l'audit qu'elle administre | Cohérence | Sécurité de la plateforme traitée comme un domaine, pas comme une checklist (voir `07`) |

### NFR-8 mérite d'être explicite

Cette application concentre, pour chaque client, la cartographie complète de ses actifs critiques,
de ses vulnérabilités non corrigées, de ses risques acceptés et de ses dérogations en cours. C'est
une cible de très grande valeur. En mode MSSP, une compromission expose le portefeuille entier.

Le traiter comme une application métier ordinaire serait une faute de conception. Les conséquences
sont développées dans `07-securite-plateforme.md`.
