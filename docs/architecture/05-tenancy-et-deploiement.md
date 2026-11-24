# 05 — Multi-tenancy, mandats MSSP et modes de déploiement

## 1. Le problème posé

Trois modes de livraison à servir avec un seul code :

1. **SaaS souverain mutualisé** — N entités clientes sur une instance.
2. **On-premise mono-tenant** — chez une entité assujettie, éventuellement sans accès Internet.
3. **Cabinet / MSSP** — un prestataire pilote la conformité de N entités, avec consolidation.

Le réflexe habituel est de traiter 1 et 2 comme deux produits. C'est l'erreur qui double le coût de
maintenance et finit par faire diverger les deux chemins de code.

## 2. Décision : un seul chemin de code, le mono-tenant est un cas dégénéré du multi-tenant

- `tenant_id` sur **toutes** les tables métier, en tête de clé primaire.
- Isolation par **Row-Level Security PostgreSQL**, activée sans exception, avec le `tenant_id`
  courant posé par la couche applicative dans une variable de session.
- En on-premise : exactement le même schéma, avec **une seule ligne de tenant**. Aucune branche
  conditionnelle, aucun `if mode == "onprem"` dans le code métier.

Conséquence : la requête qui oublie son filtre de tenant ne fuit pas de données, elle ne retourne
rien — l'échec est visible en développement au lieu d'être une fuite en production. C'est le seul
argument qui compte vraiment pour la RLS.

### Ce que la RLS ne couvre pas, et qu'il faut traiter séparément

- **Les migrations et les tâches de fond** s'exécutent souvent avec un rôle privilégié qui contourne
  la RLS. Ces chemins doivent être audités à part, et exécutés avec un rôle dédié non `BYPASSRLS`
  dès que c'est possible.
- **Les très gros tenants** peuvent justifier un schéma dédié ou une instance dédiée. À prévoir
  comme une option de placement (le tenant porte sa localisation physique), pas comme un second
  modèle de données.
- **Les exports et les projections de `reporting`** doivent revérifier le périmètre : une vue
  matérialisée mal filtrée est le point de fuite classique de ce type de produit.

## 3. Le mandat MSSP : l'agrégat le plus sensible du produit

Un cabinet ne doit **pas** être modélisé comme un tenant parent possédant ses clients. Les données
restent celles du client ; le cabinet obtient un accès délégué, borné et révocable.

```
Mandate
  mandate_id
  provider_tenant_id     le cabinet / MSSP
  client_tenant_id       l'entité assujettie
  scope                  domaines et périmètres accessibles
  permissions            lecture | contribution | pilotage
  granted_by             qui, chez le client, a consenti
  valid_from / valid_until
  revoked_at / revoked_by
```

Règles à tenir :

- **Un mandat a une échéance.** Un accès délégué perpétuel est intenable pour le client comme pour
  l'éditeur.
- **Le client peut révoquer unilatéralement, et la révocation est immédiate**, y compris sur les
  sessions en cours. Un cache d'autorisation de 15 minutes sur une révocation de mandat est un
  défaut de sécurité, pas un détail de performance.
- **Toute action du cabinet est journalisée avec sa double identité** : l'utilisateur du cabinet
  *et* le mandat qui l'autorisait. Sans cela, le journal d'audit du client est inexploitable.
- **La consolidation de portefeuille ne lit que des agrégats.** Le tableau de bord du cabinet affiche
  des indicateurs par client ; il ne doit pas constituer un chemin d'accès détourné aux données
  détaillées hors du périmètre de chaque mandat.

> Point à faire valider juridiquement, qui n'est pas qu'une question technique : le statut du
> cabinet au regard du RGPD (sous-traitant ou responsable conjoint) et les conséquences
> contractuelles de l'accès délégué. Je n'ai pas de réponse vérifiée à vous donner sur ce point, et
> il peut influer sur le modèle de consentement à implémenter.

## 4. Contraintes du mode on-premise sur l'architecture

Le mode on-premise interdit, dans le noyau, tout ce qui suit :

| Interdit | Alternative portable |
|---|---|
| File d'attente managée propriétaire | Broker déployable — RabbitMQ, NATS, ou PostgreSQL en file pour les petits volumes |
| Stockage objet propriétaire | API compatible S3 (MinIO, Ceph, ou l'offre du client) |
| Fonctions serverless propriétaires | Conteneurs, ordonnanceur applicatif |
| Base de données managée propriétaire | PostgreSQL, version et extensions explicitement supportées |
| Service d'identité propriétaire | OIDC/SAML vers l'IdP du client |
| Service managé de chiffrement | Interface `key-management` + adaptateur KMIP/HSM |
| Appel sortant obligatoire pour le contenu réglementaire | **Paquet de contenu signé, importable hors ligne** |

Cette dernière ligne est celle qu'on oublie et qui bloque le premier déploiement souverain. Le
mécanisme de mise à jour du référentiel doit être conçu dès le départ comme un import de fichier
signé, l'actualisation en ligne n'étant qu'une commodité par-dessus.

### Conséquence sur le nombre de services déployables

Un client on-premise doit pouvoir exploiter la plateforme avec une équipe d'infrastructure
ordinaire. Livrer 26 conteneurs, un broker, un ordonnanceur et un maillage de services à une DSI de
taille moyenne est un frein commercial réel, pas une préférence esthétique. C'est l'argument le plus
concret en faveur du regroupement de déploiement décrit dans `08-trajectoire.md` : la cible
microservices reste valable pour le SaaS, tandis que l'on-premise se livre en un petit nombre
d'unités à partir du même code.

## 5. Résidence et souveraineté des données

- Le `Tenant` porte sa **région de résidence** ; aucune donnée métier ne traverse les régions, y
  compris dans les journaux et la télémétrie.
- Les journaux d'exploitation sont purgés des données métier par conception, pas par filtrage a
  posteriori. Un identifiant de tenant dans une trace est acceptable ; un nom d'actif critique ne
  l'est pas.
- Pour les clients visant un hébergement qualifié, le choix du fournisseur et les exigences de
  qualification applicables sont à vérifier auprès de sources officielles au moment de la décision.
  > Je ne liste pas ici de fournisseurs qualifiés : cette liste évolue et je n'ai pas de source
  > vérifiée à jour à vous citer. À confirmer sur le site de l'autorité compétente.
