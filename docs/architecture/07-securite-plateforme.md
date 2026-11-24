# 07 — Sécurité de la plateforme

## 1. Pourquoi ce chapitre est un chapitre d'architecture

Cette application concentre, par tenant : la cartographie des actifs critiques, les vulnérabilités
non corrigées avec leur exposition, les risques acceptés, les dérogations en cours, les plans de
continuité et les coordonnées de la cellule de crise.

Autrement dit : **le plan d'attaque optimal de l'entité, consolidé et tenu à jour par l'entité
elle-même.** En mode MSSP, une compromission de l'instance expose le portefeuille complet.

Trois conséquences qui remontent au découpage lui-même, et non à une checklist de durcissement :

1. **`key-management` est un service isolé.** La compromission d'un service métier ne doit pas
   donner accès aux clés de chiffrement des tenants.
2. **`integration-hub` est isolé et cloisonné.** Il détient les identifiants vers le SIEM, l'EDR et
   la CMDB du client — des accès souvent plus privilégiés que ceux de la plateforme elle-même. Il ne
   partage aucun secret avec les autres services.
3. **`audit-trail` n'expose aucune écriture modificative.** Pas d'API de suppression, pas d'API de
   mise à jour, aucune exception administrateur. Un journal d'audit effaçable par un administrateur
   compromis n'est pas un journal d'audit.

## 2. Exigences minimales

| Domaine | Exigence | Remarque |
|---|---|---|
| Authentification | MFA obligatoire pour tous les rôles en SaaS ; fédération vers l'IdP du client en on-premise | Pas d'exception « compte de service humain » |
| Autorisation | Point de décision unique ; refus par défaut ; périmètre effectif recalculé à chaque requête sensible | La révocation de mandat doit être immédiate |
| Chiffrement au repos | Par tenant ; BYOK/HYOK disponible en on-premise | Chiffrement applicatif des champs les plus sensibles, en plus du chiffrement disque |
| Chiffrement en transit | TLS entre services, y compris en déploiement groupé | Évite que le regroupement de déploiement dégrade la posture |
| Secrets | Dans `key-management` ou un coffre externe ; jamais en variables d'environnement de conteneur applicatif | |
| Journalisation | Toute lecture de données sensibles d'un tenant par un utilisateur de cabinet est journalisée | Exigence contractuelle vis-à-vis des clients du MSSP |
| Sauvegardes | Chiffrées, testées par restauration régulière, avec preuve de test | L'éditeur doit pouvoir produire cette preuve : elle lui sera demandée |
| Cloisonnement | Les données d'un tenant ne traversent pas sa région de résidence, y compris en télémétrie | |

## 3. L'éditeur est lui-même probablement assujetti

Un fournisseur de plateforme de conformité est un prestataire de services numériques pour des
entités assujetties. À ce titre, il entre dans le périmètre de la chaîne d'approvisionnement de ses
clients (art. 21.2.d) et sera évalué par eux. Selon son activité et sa taille, il peut aussi être
directement assujetti.

> Je ne peux pas qualifier votre structure sans connaître son secteur, sa taille et son activité
> exacte. **À faire qualifier explicitement**, idéalement avec le même moteur de scoping que celui
> que vous construisez : c'est aussi le meilleur test de ce moteur.

Conséquence pratique immédiate : prévoir dès le départ de pouvoir répondre aux questionnaires
fournisseurs de vos clients, et traiter la plateforme comme un périmètre d'homologation à part
entière. Les clients régulés le demanderont avant signature, et l'absence de réponse prête est un
motif de blocage commercial fréquent sur ce marché.
