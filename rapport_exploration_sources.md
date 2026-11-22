# Rapport d’exploration de sources multimodales

## 1. Contexte et objectif

CheckItAI souhaite constituer un corpus permettant d’étudier la désinformation multimodale, c’est-à-dire des publications dans lesquelles le texte et l’image sont associés. L’objectif de cette exploration est d’identifier des sources accessibles, de qualifier leurs données et de proposer une méthode d’extraction automatisable.

Le rapport distingue :

- les sources utiles pour **collecter des publications récentes** ;
- les sources utiles pour **obtenir des labels vrai/faux ou des annotations** ;
- les sources qui fournissent une image associée au texte, et celles qui ne fournissent qu’un lien vers une image.

Une publication sera considérée comme multimodale seulement si son texte et son image peuvent être reliés de manière déterministe dans la même entrée.

## 2. Schéma minimal d’une publication

Le pipeline devra normaliser les sources vers un schéma commun :

```json
{
  "id": "identifiant_unique",
  "source": "nom_de_la_source",
  "title": "titre",
  "text": "texte ou résumé disponible",
  "image_url": "https://...",
  "local_image_path": "images/identifiant.jpg",
  "article_url": "https://...",
  "published_at": "2026-11-22T10:00:00Z",
  "author": null,
  "domain": "example.org",
  "language": "fr",
  "label": null,
  "label_source": null,
  "extraction_method": "api|rss|scraping|dataset"
}
```

Les champs `label` et `label_source` doivent rester nuls lorsqu’aucun label documenté n’est disponible. La réputation d’un média ou le score d’un post ne constitue pas automatiquement un label de vérité.

## 3. Critères de qualification

Chaque source est évaluée selon les critères suivants :

1. présence du texte et de l’image dans une même publication ;
2. format et facilité d’accès ;
3. langue et couverture thématique ;
4. présence et qualité des labels ;
5. stabilité de la méthode d’extraction ;
6. métadonnées disponibles ;
7. conditions d’utilisation et droits de réutilisation ;
8. risques de biais, de doublons et de bruit.

## 4. Source 1 — NewsAPI

### Identification

- **Type :** API REST d’articles d’actualité.
- **Format :** réponse JSON.
- **Accès :** clé API et respect des limites du plan utilisé.
- **Documentation :** l’endpoint `everything` permet de rechercher des articles et expose notamment le titre, la description, l’URL, la date, la source et une URL d’image lorsqu’elle est fournie ([documentation NewsAPI](https://newsapi.org/docs/endpoints/everything)).

### Modalités et langues

La modalité textuelle comprend généralement le titre et une description ; le contenu complet dépend de la source et de l’offre utilisée. Une image peut être disponible via le champ `urlToImage`, mais elle n’est pas garantie pour chaque article. La recherche peut être filtrée par langue à l’aide de codes ISO-639-1.

### Labels

- **Label vrai/faux :** absent par défaut.
- **Interprétation possible :** aucune étiquette de vérité ne doit être déduite du média d’origine.
- **Enrichissement recommandé :** rattacher ultérieurement l’URL à une base de fact-checking et conserver `label_source`.

### Méthode d’extraction proposée

Utiliser `requests` ou `httpx` avec pagination :

1. envoyer une requête sur `/v2/everything` ou `/v2/top-headlines` ;
2. sélectionner les langues, domaines et dates ;
3. extraire le titre, la description, l’URL, la date et `urlToImage` ;
4. télécharger l’image avec un délai et un timeout ;
5. vérifier le type MIME, la taille et le code HTTP ;
6. écrire une ligne JSONL par publication ;
7. journaliser les articles sans image.

### Qualité, avantages et limites

- **Avantages :** structure JSON, métadonnées utiles, automatisation simple, plusieurs langues et domaines.
- **Limites :** quotas, images manquantes, texte parfois tronqué, labels absents et biais de couverture.
- **Droits :** vérifier les conditions NewsAPI et les droits de chaque média avant de redistribuer le texte ou les images. Pour un prototype, conserver prioritairement les URLs, les métadonnées et les images autorisées.

## 5. Source 2 — Reddit

### Identification

- **Type :** réseau social composé de posts, liens, images, vidéos et commentaires.
- **Format :** objets JSON via API, selon le mode d’accès autorisé.
- **Documentation :** le modèle officiel d’un post expose notamment le titre, le corps, l’URL, la miniature, la date, le subreddit et les informations de média ([modèle Post Reddit](https://developers.reddit.com/docs/api/public-api/classes/Post)).

### Modalités et langues

Le titre et le corps du post constituent la modalité texte. L’image peut être hébergée par Reddit ou être référencée par une URL externe. Les posts peuvent aussi contenir une galerie ; le pipeline devra donc prévoir une liste d’images ou sélectionner explicitement la première image.

La langue dépend du subreddit et des utilisateurs. Elle doit être détectée a posteriori, par exemple avec `langdetect` ou fastText, plutôt que supposée à partir du nom du subreddit.

### Labels

- **Label vrai/faux :** généralement absent et peu fiable.
- **Signaux secondaires :** flair, score, nombre de commentaires, modération et liens vers des sites de fact-checking.
- **Précaution :** un flair ou un score ne doit pas être converti automatiquement en vérité terrain.

### Méthode d’extraction proposée

Utiliser l’accès officiel autorisé, avec authentification si nécessaire :

1. sélectionner des subreddits liés à l’actualité et à la vérification des faits ;
2. récupérer les posts selon une fenêtre temporelle ;
3. extraire titre, corps, URL, date, subreddit et métadonnées de média ;
4. identifier les posts image, lien ou galerie ;
5. télécharger uniquement les médias autorisés ;
6. conserver l’URL du post et le contexte de collecte ;
7. supprimer les données si la politique de la plateforme l’exige.

Reddit indique que les contenus supprimés et les données associées doivent être supprimés du stockage ; cette contrainte doit être prévue dans le pipeline ([règles Reddit Data API](https://support.reddithelp.com/hc/en-us/articles/16160319875092-Reddit-Data-API-Wiki)).

### Qualité, avantages et limites

- **Avantages :** diversité de textes, images natives, commentaires et signaux sociaux.
- **Limites :** bruit, ironie, opinions, doublons, biais communautaires, contenu supprimé et labels faibles.
- **Droits :** utiliser l’accès officiel et respecter les règles de conservation, d’identification et de redistribution.

## 6. Source 3 — Flux RSS de sites d’actualité

### Identification

- **Type :** flux de syndication publié par un site ou un média.
- **Format :** XML RSS ou Atom.
- **Accès :** téléchargement périodique du flux avec une fréquence raisonnable.
- **Référence technique :** RSS définit notamment l’élément `enclosure`, qui peut référencer un fichier externe avec une URL, une taille et un type MIME ([spécification RSS 2.0](https://www.rssboard.org/rss-specification)).

### Modalités et langues

Un item contient généralement un titre, un résumé, une date et un lien vers l’article. L’image peut apparaître dans `media:content`, `media:thumbnail`, `enclosure` ou dans le HTML de l’article. La langue est celle du média ; elle doit être conservée depuis les métadonnées lorsqu’elle existe et contrôlée par détection automatique.

### Labels

- **Label vrai/faux :** absent.
- **Interprétation :** la source indique l’origine éditoriale, pas la véracité de chaque affirmation.
- **Enrichissement :** comparer les URLs avec une base de fact-checking ou une seconde source indépendante.

### Méthode d’extraction proposée

Utiliser `feedparser` :

1. maintenir une liste de flux autorisés ;
2. télécharger chaque flux à intervalle régulier ;
3. extraire `title`, `summary`, `link`, `published` et les champs média ;
4. si l’image manque, consulter l’article avec une requête contrôlée et analyser les balises `og:image` ;
5. associer l’image à l’item grâce à son URL canonique ;
6. stocker le résultat normalisé en JSONL ou Parquet.

### Qualité, avantages et limites

- **Avantages :** méthode légère, lisible, adaptée à une exécution planifiée et à plusieurs médias.
- **Limites :** contenu souvent résumé, images parfois absentes, structures variables, risque de duplication avec NewsAPI.
- **Droits :** respecter les conditions du site, `robots.txt` et les limites de fréquence. Le flux ne donne pas automatiquement le droit de redistribuer l’image.

## 7. Source 4 — FakeNewsNet

### Identification

- **Type :** dépôt/dataset de recherche sur la détection de fake news.
- **Format :** fichiers et structures JSON, scripts de collecte et métadonnées selon les sous-ensembles.
- **Langue principale :** anglais.
- **Référence :** le dépôt distingue notamment des exemples provenant de Politifact et GossipCop, avec des labels `fake` et `real` ([dépôt FakeNewsNet](https://github.com/KaiDMML/FakeNewsNet)).

### Modalités et labels

Le dataset associe des informations d’articles à un contexte social issu de plateformes. Les labels sont plus adaptés à l’évaluation supervisée que ceux de NewsAPI, Reddit ou RSS, car ils proviennent de collections liées à des sites de vérification ou de classement documentées par le projet.

Il faut néanmoins vérifier, pour chaque sous-ensemble, quels champs sont réellement disponibles localement : texte, image, URL, tweets, profils et métadonnées sociales ne sont pas nécessairement présents avec la même complétude.

### Méthode d’extraction proposée

1. cloner ou télécharger le dépôt et identifier la licence ;
2. lire le README et la configuration avant toute collecte ;
3. charger les JSON avec Python ;
4. normaliser le titre, le texte, l’image, la source et le label ;
5. conserver l’origine exacte du label (`politifact`, `gossipcop`, etc.) ;
6. compléter les médias manquants uniquement si les URLs restent accessibles et si les droits le permettent ;
7. séparer les données d’entraînement, de validation et de test par article ou événement pour éviter les fuites.

### Qualité, avantages et limites

- **Avantages :** présence de labels documentés, intérêt pour l’entraînement et l’évaluation, contexte social.
- **Limites :** principalement anglophone, liens anciens ou cassés, couverture thématique limitée, risque de biais provenant des sites de fact-checking.
- **Droits :** vérifier la licence du dépôt, les conditions des contenus externes et les restrictions des plateformes citées.

## 8. Comparaison synthétique

| Source | Format | Texte | Image | Langue | Labels | Méthode | Rôle recommandé |
|---|---|---|---|---|---|---|---|
| NewsAPI | JSON/API REST | titre, description, parfois contenu | URL facultative | multilingue | absents | `requests`/`httpx` | collecte d’actualité |
| Reddit | JSON/API officielle | titre, corps, commentaires | image, galerie ou URL | variable | faibles/absents | API autorisée | diversité sociale |
| Flux RSS | XML RSS/Atom | titre, résumé, lien | `media`, `enclosure` ou HTML | langue du média | absents | `feedparser` | collecte planifiée |
| FakeNewsNet | JSON/dépôt | articles et contexte | selon sous-ensemble | surtout anglais | `fake`/`real` documentés | lecture de fichiers | entraînement/évaluation |

## 9. Architecture d’extraction recommandée

Le pipeline peut être organisé en modules indépendants :

```text
collectors/
  newsapi_collector.py
  reddit_collector.py
  rss_collector.py
  fakenewsnet_loader.py
normalization/
  schema.py
  language.py
  deduplication.py
validation/
  validate_records.py
storage/
  jsonl_writer.py
  parquet_writer.py
```

Chaque collecteur retourne le même objet normalisé. Un orchestrateur lance les collecteurs, journalise les erreurs et écrit les résultats dans `data/raw` puis `data/processed`.

## 10. Contrôles qualité indispensables

Pour chaque publication :

- vérifier que le titre ou le texte n’est pas vide ;
- vérifier que l’URL est valide et canonique ;
- vérifier que l’image répond avec un type MIME image ;
- calculer un hash de l’image pour détecter les doublons ;
- détecter les textes et images identiques provenant de plusieurs sources ;
- conserver la date et le fuseau horaire ;
- mesurer la langue ;
- conserver la provenance du label ;
- exclure les entrées où l’image n’est pas reliée au texte ;
- produire un rapport d’exécution : nombre collecté, nombre multimodal, erreurs, doublons et images invalides.

Les indicateurs à publier dans le rapport d’exécution sont notamment :

```text
total_records
records_with_text
records_with_image
records_text_and_image
invalid_images
duplicates
records_by_language
records_by_source
records_by_label
```

## 11. Cas typiques à surveiller

Le corpus doit distinguer :

- une fausse affirmation accompagnée d’une image réelle mais sortie de son contexte ;
- une image authentique associée à un événement différent ;
- une image modifiée ou générée, accompagnée d’un texte trompeur ;
- un titre sensationnaliste dont le contenu ne confirme pas l’affirmation ;
- une opinion ou une satire, qui ne doit pas être automatiquement classée comme fake news.

Ces cas justifient de conserver séparément les champs texte, image, contexte, source, date et label. Un modèle ne doit pas apprendre uniquement la réputation du domaine ou le style visuel d’une source.

## 12. Recommandation finale

La combinaison recommandée pour un premier prototype est :

1. **NewsAPI** pour une collecte structurée d’articles ;
2. **flux RSS** pour une collecte régulière et peu coûteuse ;
3. **Reddit** pour des posts sociaux et des images d’utilisateurs, avec les précautions de conservation et d’accès ;
4. **FakeNewsNet** comme source de référence pour les labels documentés et l’évaluation.

Cette combinaison couvre plusieurs méthodes d’acquisition : API REST, API de réseau social, parsing RSS et lecture de dataset. Elle permet aussi de séparer les données récentes non annotées des données historiques annotées.

La principale limite est l’absence de labels fiables dans les sources de collecte en temps réel. Il faudra donc éviter de transformer automatiquement « média reconnu », « post populaire » ou « opinion controversée » en `real` ou `fake`. Les labels doivent être reliés à une source de vérification explicite et conservés avec leur provenance.

## 13. Références consultées

- [NewsAPI — endpoint Everything](https://newsapi.org/docs/endpoints/everything)
- [Reddit Developers — modèle Post](https://developers.reddit.com/docs/api/public-api/classes/Post)
- [Reddit — Data API Wiki](https://support.reddithelp.com/hc/en-us/articles/16160319875092-Reddit-Data-API-Wiki)
- [RSS 2.0 Specification](https://www.rssboard.org/rss-specification)
- [FakeNewsNet — dépôt GitHub](https://github.com/KaiDMML/FakeNewsNet)

**Date de consultation :** 22 novembre 2026.
