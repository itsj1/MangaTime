# MangaTime — notes pour Claude

App personnelle (un seul utilisateur) de suivi de **mangas / manhwas / webtoons**, **animés** et **séries** : où j'en suis, ce qu'il reste, quand sort la suite. Inspirée de l'ancienne app **TVTime**, avec le style de ma PWA **Fitness Tracker** (`itsj1/FitnessTracker`).

- En ligne : https://itsj1.github.io/MangaTime (GitHub Pages, branche `main`).
- Tout doit rester **gratuit** : pas de serveur, pas de service payant.
- Langue de l'interface et des messages de commit : **français**.

## Architecture

- PWA en **un seul fichier `index.html`** (CSS + JS inline, sans framework ni build), plus `sw.js`, `manifest.json` et les icônes.
- `sw.js` : réseau d'abord, cache en secours. **Incrémenter `CACHE` (`mangatime-vN`) à chaque version publiée** pour que le téléphone récupère la nouvelle.
- **La page elle-même ne défile pas** : `main#app` est fixé plein écran et c'est lui qui défile (`scrY()` / `scrTo(y)`, jamais `scrollY` / `scrollTo`), pour que le rebond d'iOS ne fasse pas bouger la barre d'onglets.
- Rendu : `render()` reconstruit `#app` en `innerHTML` ; les actions passent par `data-a=…` dans un seul écouteur de clic.
- État d'affichage : `TAB` (`disc` / `home` / `cal` / `list`), `VIEW` (clé de l'œuvre ouverte), `SUBV` (sous-page : catégorie, `next:T`, `stats`…), `F` (feuille ou pop-up ouvert). Navigation retour via `history.pushState` / `popstate`.
- `sheet.png` à la racine est une ancienne planche d'aperçu, inutile.

## Données

- `localStorage['mangatime-v1']` → `S = { items, del, cfg, sha, hv, pm }`.
- Une œuvre = `S.items[k]`, avec `k` = `al:<id>` (AniList) ou `tm:<id>` (TMDB).
  - `ty` : `A` animé, `T` série, `M` manga. `s` (mon suivi) : `cur`, `plan` (à voir plus tard), `done`, `pause`, `drop`.
  - `w` : épisodes/chapitres cochés (ids `"saison-épisode"` pour TMDB, nombres pour AniList).
  - `st` : statut de l'œuvre (`rel`, `soon`, `fin`, `stop`, `hia`). `nx` : prochain épisode annoncé. `la` : dernier épisode diffusé (TMDB).
  - `ss` (TMDB) : `[n° saison, nb d'épisodes, nom, n° du 1er épisode, chargé]` — certaines œuvres numérotent en continu (Hunter x Hunter : la saison 2 commence à l'épisode 27).
  - `lk` / `lkn` : lien perso pour lire / regarder (adresse, nom facultatif), réglé depuis le menu ⋯ de la fiche ; bouton « Lire » / « Regarder » sous la barre d'avancement.
  - `rt` : durée moyenne d'un épisode (min). `rel` / `relm` : chapitres sortis d'un manga en cours (estimé / saisi). `u` (modifié), `a` (ajouté), `lw` (dernier épisode coché), `g` (genres).
- `cfg` : clé TMDB, dépôt + fichier + jeton GitHub pour la synchro.

## Sources

- **TMDB** (clé API gratuite, saisie dans les réglages) : séries **et animés** — une seule fiche par œuvre, découpée en saisons ; saisons, épisodes (titres, vignettes, résumés), acteurs, genres, durées, prochain épisode.
- **AniList** (GraphQL, sans clé) : mangas / manhwas / webtoons, et animés seulement en secours si aucune clé TMDB n'est saisie.
- **Chapitres sortis d'un manga en cours** (AniList ne les donne pas) : MangaDex (`/manga?title=` puis `/aggregate`, fiche retrouvée via son lien AniList) + activité des lecteurs AniList ; on garde le max. Saisie manuelle seulement si rien n'est trouvé. Résultat de chaque source gardé dans `it.rs` (`{md, al}` : nombre, 0 = introuvable, ou « bloqué » / « erreur N ») et affiché sous « chapitres sortis » sur la fiche, avec la source retenue en gras et un bouton Revérifier. CORS de MangaDex toujours non confirmé : regarder cette ligne sur le téléphone.
- Mention TMDB obligatoire, présente en bas des réglages.

## Synchro entre appareils

Fichier `mangatime-data.json` dans un **dépôt GitHub privé**, via l'API Contents (même principe que Fitness Tracker), avec un jeton fine-grained « Contents : Read and write ». Fusion œuvre par œuvre selon `u` ; les suppressions sont datées dans `del`.

## Écrans

- **Discover** : recherche (croix pour effacer), filtres Animés / Mangas & Webtoons / Séries. Sans recherche : 4 rangées défilantes Tendances · Populaires · Mieux notés · Nouveautés (`catFetch()`, gardées dans `DRW`), chacune avec « Tout voir », + bouton « Explorer tout le catalogue ». Avec recherche : posters 3 par ligne, titre sur 2 lignes fixes + année.
- **Explorer** (`SUBV='exp'`, état `EX`) : rangées de bulles Tri / Genre / Année / Origine (mangas : Manga, Manhwa, Manhua) ou Pays (séries) / Statut, grille chargée page par page au défilement. AniList : `sort`, `genre`, `countryOfOrigin`, `startDate_greater/lesser` ; TMDB : `discover/tv` (`trending/tv/week` pour Tendances sans filtre).
- Rangées de bulles (`.chips`) élastiques quand elles tiennent dans l'écran : on peut les tirer, elles reviennent en place. `render()` garde la position horizontale des éléments `[data-sx]`.
- **Fiche** : genres, statut de suivi, barre d'avancement, prochaine sortie, saisons **repliées par défaut** (vignette + titre sur 2 lignes, bouton ⋯ si coupé), acteurs (rangée défilante, étiquette « voix »), bouton ⋯ en haut (ajouter / retirer, à voir plus tard, tout marquer, remettre à zéro, page source). Coche d'un épisode avec dialogue « précédents aussi / seulement celui-ci / annuler ». Mangas : seulement « J'en suis au chapitre… » + chapitres sortis (pas de grille, pas de saisons, pas de jour de sortie).
- **Home** : sections Séries / Animés / Mangas (titre sans nombre, « Tout voir (N) » toujours présent), chacune en deux rangées défilant vers la droite : « En cours » puis « À jour » (œuvre pas finie, tout vu) ; liste = cartes, grille = affiches ; puis Pas commencées, Pas regardées depuis un moment (30 j), À voir plus tard. Libellés : `S2E11 +3`, `Ch. 215 / 230`, `S1E12 · à jour`, `S2E11 · jeudi` (sur une ligne).
- **Calendrier** : « Sortis, à regarder » avec bulle pour valider (épisode suivant ou date du prochain ensuite), puis prochaines sorties jour par jour. Séries et animés uniquement.
- **Profil** : stats (œuvres, temps passé, épisodes vus, chapitres lus) + « Détails » (page Statistiques par type) ; rangées Mes séries / Mes animés / Mes mangas. « Tout voir » ouvre une page catégorie : recherche, filtres et tri, affichage grille ×2 à ×5 / liste / détaillée, groupes Vu récemment · Pas regardé depuis un moment · À jour · Complété (toujours affichés, même à 0).

## Design

- Thème sombre par défaut (clair automatique), accent **jaune TVTime** `#ffd400`.
- Barres : jaune = en cours, **vert** = à jour sur une œuvre en cours, **violet clair** = œuvre terminée entièrement vue, orange = pas regardée depuis un moment.
- Bouton rond pour cocher un épisode ; feuilles par le bas (glisser pour fermer, fermeture animée) ; réglages en cartes groupées comme Fitness Tracker ; engrenage en haut à droite.
- Onglets : Discover · Home · Calendrier · Profil.
- Bandeau « S2E5 marqué comme vu · Annuler » 4 s après chaque changement de coches (`commit()` → `undoToast()`, annule aussi l'ajout à la liste).
- Tirer vers le bas en haut de Home / Calendrier : actualise toutes les œuvres suivies (`refreshAll(true)`) et lance la synchro si elle est réglée (pastille `#ptr`).
- Retour en glissant depuis le bord gauche (24 px) sur la fiche et les sous-pages : la page suit le doigt, repart si > 35 % de l'écran ou geste rapide.
- Transitions de page comme Fitness Tracker (`slide()`, View Transitions API, repli en animation de `#app`) : fiche et sous-pages (« Tout voir », « Détails ») arrivent par la droite, retour vers la droite.

## Façon de travailler

- L'utilisateur décrit ses demandes en français, souvent à l'oral : reformuler si ambigu, proposer des idées en plus.
- Tester dans Chrome headless (Playwright) avec des réponses d'API simulées, vérifier sur captures, puis **pousser directement sur `main`** (le site se met à jour seul).
