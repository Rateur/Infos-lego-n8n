# Veille LEGO sur Discord avec n8n

Deux workflows n8n qui publient l'actualité LEGO sur un salon Discord : l'un trie des flux RSS par date, l'autre confie la recherche, la traduction et le tri à un LLM.

## Ce que ça fait

Chaque workflow est lancé par un déclencheur planifié et envoie un message (un embed Discord) par article : titre, texte, lien, image si elle existe.

| | LEGO sans IA | LEGO avec IA |
|---|---|---|
| Source | 5 flux RSS (Hoth Bricks, Brickset, The Brothers Brick, Hello Bricks, Jay's Brick Blog) | Recherche d'actualités Serper : `LEGO 2026 sets leaks rumors`, 10 résultats, 7 derniers jours |
| Tri | Garde les articles publiés il y a moins de 24 h | Le LLM écarte ce qui n'est pas LEGO. Puis un script ne garde que les textes qui contiennent « lego » |
| Texte publié | Extrait de l'article, tel quel | Résumé traduit en français par le LLM |
| Clés à fournir | Aucune, juste l'adresse du webhook Discord | Serper, OpenRouter et l'adresse du webhook Discord |

La version avec IA n'utilise pas les flux RSS. Elle part d'une recherche Serper.

### Captures

Sans IA : le workflow, puis le rendu sur Discord.

![Workflow n8n sans IA](https://i.imgur.com/w05HuE9.png)
![Rendu Discord sans IA](https://i.imgur.com/eJKvL3g.png)

Avec IA : le workflow, puis le rendu sur Discord.

![Workflow n8n avec IA](https://i.imgur.com/ucpF9wN.png)
![Rendu Discord avec IA](https://i.imgur.com/6ZMCGRU.png)

## Comment c'est fait

```text
.
├── docker-compose.yml    # n8n en local
├── LEGO sans IA.json     # workflow RSS
├── LEGO avec IA.json     # workflow Serper + LLM
└── README.md
```

- n8n, auto-hébergé avec Docker Compose (image `docker.n8n.io/n8nio/n8n`, port 5678, données dans le volume `n8n_storage`, fuseau `Europe/Paris`).
- Nœuds utilisés : Schedule Trigger, RSS Feed Read, Merge, Filter, Edit Fields, Code (JavaScript), HTTP Request, Wait.
- APIs : Serper.dev pour la recherche, OpenRouter pour le LLM, webhook Discord pour la publication. Les appels à Serper et OpenRouter sont de simples nœuds HTTP Request, sans nœud LangChain.

**LEGO sans IA.** Schedule Trigger, lecture des 5 flux, Merge, Filter (`pubDate` après maintenant moins 24 h), Edit Fields (titre, extrait, lien, date, première image trouvée dans `content:encoded`), Code (construit l'embed : titre limité à 256 caractères, description à 2 048), HTTP Request vers Discord, Wait.

**LEGO avec IA.** Schedule Trigger, Serper, Code « Mixeur » (met les résultats en un seul texte), LLM, Edit Fields « Extraction », Code « Filtre LEGO », HTTP Request « Post Discord », Wait. Le LLM est `google/gemma-3-4b-it:free` via OpenRouter. Le prompt lui demande de supprimer ce qui n'est pas LEGO, de tout traduire en français et de renvoyer du JSON `{ "news": [ { title, summary, link, image } ] }`. Le script « Filtre LEGO » retire les balises Markdown autour de la réponse, la parse, puis garde les entrées dont le titre ou le résumé contient « lego ».

## Lancer le projet

Prérequis : Docker avec Docker Compose.

1. Démarrer n8n à la racine du dépôt :

   ```bash
   docker compose up
   ```

   Ajoute `-d` pour le laisser tourner en arrière-plan. n8n est ensuite sur http://localhost:5678.

2. Importer les workflows : dans n8n, crée un workflow, ouvre le menu en haut à droite, choisis « Import from file », puis sélectionne `LEGO sans IA.json` ou `LEGO avec IA.json`.

3. Renseigner l'adresse du webhook Discord (voir ci-dessous).

4. Pour la version avec IA, créer les deux identifiants (voir ci-dessous).

5. Lancer le workflow à la main pour tester, puis l'activer pour que le déclencheur planifié tourne.

### Webhook Discord

Dans les deux fichiers, l'adresse est un exemple :

```text
https://discord.com/api/webhooks/VOTRE_ID/VOTRE_JETON
```

1. Dans Discord : Paramètres du salon, Intégrations, Webhooks, Nouveau webhook, puis copie l'adresse.
2. Dans n8n, ouvre le nœud d'envoi (`HTTP Request` dans la version sans IA, `Post Discord` dans la version avec IA) et remplace l'exemple dans le champ URL.

Ne commite jamais cette adresse. Quiconque la connaît peut écrire dans ton salon. Une fois collée dans un nœud, elle est exportée avec le workflow : avant de commiter un `.json` modifié, remets l'exemple. Si une vraie adresse a déjà été commitée, elle reste dans l'historique git : supprime ce webhook dans Discord et crées-en un autre.

### Clés OpenRouter et Serper (version avec IA seulement)

Les fichiers JSON ne contiennent aucune clé, seulement le nom de deux identifiants n8n de type « Header Auth » (`OpenRouter` et `Serper`).

1. Dans n8n, crée un identifiant « Header Auth » pour chaque service. Le nom de l'en-tête et le format de la valeur sont dans la documentation du service. En pratique : `Authorization` avec `Bearer <ta clé OpenRouter>`, et `X-API-KEY` avec ta clé Serper.
2. Ouvre les nœuds `LLM` (OpenRouter) et `Serper (recherche)`, et choisis ton identifiant dans le champ d'authentification.

Les identifiants restent dans le volume `n8n_storage`, pas dans le dépôt.

## État et limites

- Projet personnel, sans tests automatiques.
- Dans la version sans IA, les nœuds `Jay's Brick Blog` et `RSS Read4` portent le nom de l'autre : le premier lit `hellobricks.com`, le second `jaysbrickblog.com`.
- La fréquence d'exécution n'est pas dans les fichiers : le Schedule Trigger est exporté avec ses valeurs par défaut. À confirmer et à régler dans n8n.
- Aucun des deux workflows ne retient les articles déjà publiés. La version sans IA filtre sur 24 h, la version avec IA sur 7 jours : un article peut revenir si le workflow tourne plus souvent que cette fenêtre.
- Le nœud Wait (2, unité non précisée dans le JSON) est placé après l'envoi et ne mène à rien. Il ne met pas de délai entre deux messages. Discord limite le débit des webhooks : avec beaucoup d'articles, des envois peuvent être refusés. Les nœuds d'envoi sont en `continueRegularOutput`, donc l'erreur ne stoppe pas le workflow.
- Version avec IA : `JSON.parse` n'est pas protégé. Si le modèle ne renvoie pas du JSON valide, l'étape échoue. Le modèle est une variante gratuite d'un petit modèle (4B) : sa disponibilité chez OpenRouter est À confirmer.
- Version sans IA : l'image est lue avec `content:encoded` sans test préalable. Un flux qui n'a pas ce champ peut faire échouer l'étape. Non testé.
- L'image Docker n'est pas épinglée sur une version : une mise à jour de n8n peut changer le comportement des nœuds.

## Auteur

Valentin Périès, développeur IA à Lyon. [valentinperies.fr](https://valentinperies.fr) · [GitHub](https://github.com/Rateur)
