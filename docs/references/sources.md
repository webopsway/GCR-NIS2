# Sources et niveau de fiabilité

Ce fichier existe parce que le contenu réglementaire de ce produit sera opposable à des clients
devant une autorité de contrôle. Toute affirmation réglementaire doit être traçable à une source
primaire. Ce fichier doit être tenu à jour.

## Règle de travail

| Niveau | Usage autorisé |
|---|---|
| **Source primaire** — EUR-Lex, Légifrance, publication officielle de l'autorité | Saisie de contenu réglementaire dans `regulatory-content/` |
| **Source secondaire** — presse spécialisée, cabinets, blogs | Orientation et veille uniquement. **Jamais** de saisie de contenu |
| **Mémoire du rédacteur** | Aucun usage. Les références d'articles et les seuils se vérifient |

## Sources primaires à consulter avant toute saisie de contenu

Non encore consultées dans le cadre de cette note d'architecture — à faire avant le lot 1 :

- Directive (UE) 2022/2555 (NIS 2), texte consolidé sur EUR-Lex — articles 3, 20 à 23, 27, 29, 30,
  32 à 34, et annexes I et II.
- Règlement d'exécution de la Commission précisant les exigences techniques et les cas d'incidents
  importants pour certains types d'entités numériques. **Référence à récupérer sur EUR-Lex** : je ne
  la cite pas de mémoire.
- Recommandation 2003/361/CE, pour la définition des seuils de taille utilisés par la qualification.
- Dossier législatif de la loi de transposition française, sur le site de l'Assemblée nationale et
  du Sénat, puis texte promulgué sur Légifrance.
- Publication primaire du référentiel de l'ANSSI, sur le site de l'agence.
- ISO/IEC 27001:2022 et ISO/IEC 27002:2022 — **textes sous licence payante**, à acquérir. Ne pas
  saisir le contenu depuis des reprises trouvées en ligne : c'est à la fois une question
  d'exactitude et de droit d'auteur.

## Consultations effectuées le 2026-11-24 (sources secondaires)

Recherche sur l'état de la transposition française et sur le référentiel ANSSI. **Toutes ces sources
sont secondaires et ne doivent pas servir de base à la saisie de contenu.**

Ce qui en ressort, avec les contradictions relevées :

- La loi française de transposition ne serait **pas promulguée** au 24 novembre 2026. Le texte
  aurait été adopté en première lecture à l'Assemblée nationale le 17 novembre 2026 (464 voix contre
  64), une commission mixte paritaire restant nécessaire.
- La Commission européenne aurait saisi la Cour de justice à l'encontre de la France le 8 juillet
  2026.
- Le référentiel ANSSI serait structuré en 20 objectifs : 1 à 15 en socle commun aux entités
  essentielles et importantes, 16 à 20 réservés aux entités essentielles, chacun assorti de
  « moyens acceptables de conformité ».
- **Contradictions explicites entre les sources** sur : le numéro de version du référentiel, le
  nombre exact d'objectifs, et son caractère contraignant ou non. Une source affirmait que la loi
  était adoptée et les décrets d'application publiés, ce que les autres contredisent.

Sources consultées :

- [IT for Business — Loi Résilience : l'Assemblée adopte le texte qui transpose NIS 2](https://www.itforbusiness.fr/loi-resilience-nis-2-assemblee-adopte-110073)
- [Sénat — Projet de loi relatif à la résilience des infrastructures critiques et au renforcement de la cybersécurité](https://www.senat.fr/rap/l24-393/l24-393_mono.html)
- [Leto — Loi Résilience NIS 2 : adoptée à l'Assemblée](https://www.leto.legal/news/nis2-loi-resilience-adoptee-assemblee-464-voix-17-novembre-2026)
- [NIS 2 Directive — Transposition France](https://www.nis-2-directive.com/Transposition/France.html)
- [nis-solutions.eu — France](https://nis-solutions.eu/countries/france/)
- [Le Monde Informatique — L'ANSSI publie un référentiel accompagnant la conformité NIS2](https://www.lemondeinformatique.fr/actualites/lire-l-anssi-publie-un-referentiel-accompagnant-la-conformite-nis2-99753.html)
- [cyber.gouv.fr — Découvrez la directive NIS 2 et passez à l'action (PDF)](https://messervices.cyber.gouv.fr/documents-ressources/2026-03-27-NIS2.pdf)
- [Meldis — Les 20 objectifs de sécurité NIS2 de l'ANSSI](https://meldis.fr/blog/nis2-objectifs-securite-anssi/)
- [Phinasoft — ReCyF : le référentiel cyber France de l'ANSSI pour NIS 2](https://phinasoft.com/fr/guides/nis-2/recyf-referentiel-cyber-france/)
- [bzhunt.fr — NIS 2 en France : les 20 objectifs du RECYF](https://www.bzhunt.fr/nis2/)

Parmi ces sources, celle de `cyber.gouv.fr` est la seule émanant du domaine de l'autorité : c'est
celle à privilégier, et à remonter à sa publication primaire.

## Ce que je n'ai pas vérifié et ne dois pas être cité comme vérifié

- Les montants exacts des sanctions de l'article 34.
- Les seuils précis d'effectifs, de chiffre d'affaires et de bilan, et l'articulation exacte des
  conjonctions dans leur définition — point qui change le résultat de la qualification.
- La liste exhaustive des secteurs des annexes I et II et de leurs sous-secteurs.
- Toute table de correspondance NIS 2 ↔ ISO/IEC 27001. Aucune n'est fournie dans cette note,
  délibérément.
- La liste des fournisseurs d'hébergement qualifiés.
