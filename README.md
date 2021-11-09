# Consolidation des ressources respectant un schéma

## Objectif

L'objectif de ces scripts de consolidation est d'aller chercher sur data.gouv.fr les ressources respectant potentiellement un schéma donné et de les concaténer afin de créer un fichier de référence contenant les données consolidées relatives à un schéma (ou plus précisément un fichier consolidé par version de schéma). Le choix ou non de créer ces fichiers consolidés pour un schéma ou une version de schéma est entièrement géré par les fichiers de configuration. Suite à la génération des fichiers consolidés, le script est aussi capable de modifier les ressources considérées pour y ajouter/mettre à jour/supprimer leurs métadonnées "schema" (et de notifier les producteurs des ressources concernées par e-mail ou par discussion sur data.gouv.fr).

## Contenu du repository

- consolidation_tableschema.ipynb : script de consolidation des ressources respectant un schéma de type "tableschema"
- consolidation_jsonschema.ipynb (à améliorer) : script de consolidation des ressources respectant un schéma de type "jsonschema"
- config_tableschema.yml : fichier de configuration pour piloter le script de consolidation côté tableschema
- config_jsonschema.yml : fichier de configuration pour piloter le script de consolidation côté jsonschema
- ref_tables/ : dossier contenant la documentation concernant les ressources considérées pour la consolidation (une ligne par ressource)
- consolidated_data/ : dossier contenant les fichiers consolidés
- report_tables/ : dossier contenant les fichiers de statistiques agrégées concernant les ressources considérées pour la consolidation
- requirements.txt : liste des librairies Python nécessaires
- README.md : présent fichier

## Fonctionnement du script

Les deux scripts fonctionnent selon les étapes suivantes :
1. Listing des ressources qui respectent potentiellement le schéma : celles contenant la métadonnée schéma correspondante, celles comportant des tags particuliers et celles trouvées par recherche de mots-clés particuliers (ceci est réalisé pour tous les schémas dont la consolidation est activée via le fichier de configuration). Les ressources explicitement exclues de la consolidation (via le fichier de configuration) ainsi que les fichiers consolidés (s'ils existent déjà) sont ignorés.
2. Pour chacune des ressources trouvées à l'étape précédente, le script vérifie si la ressource est valide ou non vis-à-vis du schéma (ceci est réalisé pour toutes les versions du schéma, sauf celles explicitement écartées via le fichier de configuration).
3. Téléchargement des ressources valides pour au moins une version du schéma.
4. Concaténation des ressources avec déduplication. La déduplication est basée sur la clé primaire (lorsqu'elle est spécifiée par le schéma) ou sur la totalité des champs spécifés par le schéma (dans le cas contraire). En cas d'observations dupliquées, l'observation la plus récente (sur la base de la date de dernière modification de la ressource source) est conservée. Seuls les champs spécifiés par le schéma sont conservés dans le fichier consolidé final (même si des champs supplémentaires sont présents dans les ressources utilisées).
5. S'ils n'existent pas encore, création des jeux de données qui contiendront les ressources "fichiers consolidés" (titre et description du jeu de données basés sur un template). Si nécessaire, création des ressources "fichiers consolidés" par version du schéma et upload des fichiers consolidés.
6. Mise à jour de la métadonnée "schema" (et de la version appropriée) des ressources considérées : ajout, update ou suppression. Les producteurs sont alors notifiés par e-mail ou par commentaire sur le jeu de données (Dans l'état actuel du script, seuls l'ajout et l'update sont activés, les lignes de code permettant la suppression et la notification aux producteurs sont commentées).
7. Upload sur le même jeu de données que les fichiers consolidés de la table "ref_table" du schéma (en Documentation).
8. Génération des fichiers de statistiques agrégées (`report_tables`) concernant les ressources considérées pour la consolidation.

### Précisions sur le script `consolidation_tableschema.ipynb`

Pour le moment, la consolidation n'est réalisée pour une version de schéma donnée que s'il existe au moins 5 ressources respectant cette version du schéma.

### Précisions sur le script `consolidation_jsonschema.ipynb`

Pour le moment, ce script n'est capable de consolider que les schémas dont les fichiers sont de la forme :

```
{ 'key': [obs_1, obs_2, obs_3,...] }
```

où les `obs_i` sont eux-mêmes des objets de type `dict`.

Certains schémas ont d'autres spécifications qui rendent la phase de consolidation plus complexes (car les fichiers JSON ont alors une autre structure). Ce script est donc encore en cours de construction.

## Fichiers de configuration

Les fichiers de configuration (YAML) permettent de piloter différents aspects du processus de consolidation. Chaque schéma contient sa configuration dans la clé qui porte son nom technique (exemple : `etalab/schema-irve`), configuration qui peut contenir les champs suivants :

- `consolidate` : `true` ou `false` pour choisir d'activer ou non la consolidation pour la totalité du schéma (quelque soit le reste du contenu de sa configuration, ce schéma sera ignoré dans toutes ses versions pour la consolidation)
- `consolidated_dataset_id` : string contenant l'ID de jeu de donnée de consolidation du schéma sur data.gouv.fr (généré automatiquement par le script si absent du fichier de configuration et `consolidate=true`)
- `latest_resource_ids` : champ contenant comme clés les versions du schéma qui ont été consolidées et comme valeurs les strings contenant les ID des ressources "fichiers consolidés" correspondant (générés automatiquement si absents du fichier de configuration, si `consolidate=true` et si la version du schéma n'est pas dans `drop_versions` (cf. ci-dessous))
- `documentation_resource_id` : string contenant l'ID de la ressource sur data.gouv.fr contenant la "ref_table" du schéma (généré automatiquement par le script si absent du fichier de configuration et `consolidate=true`)
- `exclude_dataset_ids` : liste de strings contenant les ID des jeux de données que l'on souhaite explicitement exclure du processus de consolidation (à noter : le `consolidated_dataset_id` est déjà automatiquement ignoré par le script et n'a pas besoin d'être ajouté ici). A saisir manuellement uniquement.
- `drop_versions` : liste de strings contenant les versions du schéma pour lesquelles on ne souhaite pas créer de fichier consolidé. A saisir manuellement uniquement.
- `search_words` : liste de strings contenant les mots-clés à utiliser pour la recherche de ressources via "search". Par défaut, le nom non-technique officiel du schéma est automatiquement inclus dans cette liste à sa création. Cette liste peut cependant être modifiée à souhait et même supprimée si on ne souhaite pas faire appel au "search" pour trouver des ressources.
- `tags` : liste de strings contenant les tags à utiliser pour la recherche de ressources par tag. A saisir manuellement uniquement.

Pour le moment, sans intervention manuelle, tout nouveau schéma du catalogue officiel est ajouté au fichier de configuration avec comme configuration par défaut l'absence de consolidation (et son nom non-technique comme mot-clé par défaut pour la recherche de ressources) :

```
etalab/schema-irve:
  consolidate: false
  search_words:
  - "Infrastructures de recharge pour v\xE9hicules \xE9lectriques"
```

Exemple de configuration plus complète :

```
etalab/schema-irve:
  consolidate: true
  consolidated_dataset_id: '5448d3e0c751df01f85d0572'
  latest_resource_ids:
    2.0.2: '8d9398ae-3037-48b2-be19-412c24561fbb'
  documentation_resource_id: '41b0514d-956d-4e42-80ef-1dcf88cc74e9'
  exclude_dataset_ids:
  - '54231d4a88ee38334b5b9e1d'
  drop_versions:
  - '1.0.0'
  - '1.0.1'
  - '1.0.2'
  - '1.0.3'
  - '2.0.0'
  - '2.0.1'
  search_words:
  - "Infrastructures de recharge pour v\xE9hicules \xE9lectriques"
```

## TODO List/Pistes d'amélioration

- Rendre le script `consolidation_jsonschema.ipynb` utilisable dans le cas général
- Ajouter le suivi des fichiers vides (ou avec anomalies) au script `consolidation_jsonschema.ipynb`
- Ajouter la conservation uniquement des champs spécifiés par le schéma au script `consolidation_jsonschema.ipynb`
- Voir si un système de déduplication plus sophistiqué (primary key?) est possible dans le script `consolidation_jsonschema.ipynb`
- Pour la date à prendre en compte au moment de la déduplication, certains schémas ont un champ "date" qu'il serait peut-être plus pertinent d'utiliser que la date de dernier update de la ressource dont est issue l'observation

