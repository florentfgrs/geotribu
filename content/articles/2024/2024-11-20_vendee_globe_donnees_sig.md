---
title: "Suivre le Vendée Globe 2024 depuis un SIG"
authors:
	- Florent FOUGÈRES
categories:
	- article
comments: true
date: 2024-11-20
description: "Créer et visualiser les données SIG de l'avancement de la course du Vendée Globe 2024"
icon: material/sail-boat
image: https://cdn.geotribu.fr/img/articles-blog-rdp/articles/2024/vendee_globe_donnees_sig/trajectoire.png
license: beerware
robots: index, follow
tags:
    - QGIS
	- Python
	- Pandas
	- Geopandas
	- Voile
	- Vendée Globe
---

# Suivre le Vendée Globe 2024 depuis un SIG

## Le Vendée Globe, c’est quoi ?

Avant de commencer à parler SIG et aspects techniques, parlons du Vendée Globe.

C’est une course à la voile en solitaire, sans escale et sans assistance, autour du monde. Elle a lieu tous les 4 ans depuis 1989. Le départ se fait aux Sables d’Olonne. Le parcours consiste à descendre l’Atlantique, puis passer successivement sous l’Afrique et le Cap de Bonne Espérance, sous l’Australie et le Cap Leeuwin et enfin sous l’Amérique du Sud et le Cap Horn, pour remonter en Vendée le plus rapidement possible. Le record a été établi par Armel Le CLéac'h lors de l'édition 2016-2017 avec un trajet de 74 jours 3 heures et 35 minutes.

![carte](https://upload.wikimedia.org/wikipedia/commons/thumb/3/33/Vend%C3%A9e_Globe_map-fr.svg/2560px-Vend%C3%A9e_Globe_map-fr.svg.png)

## Suivre l’avancée

Qui dit course autour du monde, dit forcément carte pour suivre l’évolution des participants. Le site officiel de l’épreuve propose une carte interactive pour visualiser cette avancée.

![carte_interactive](https://cdn.geotribu.fr/img/articles-blog-rdp/articles/2024/vendee_globe_donnees_sig/carte_interactive.png)

J’ai donc cherché s’il existait une API ou un web service fournissant les données de positionnement pour les visualiser dans un SIG, comme QGIS par exemple. Après quelques recherches, je n’ai rien trouvé de tel.

J’ai trouvé une [discussion](https://www.reddit.com/r/Vendee_Globe/s/Gbli34xyQO) sur Reddit à ce sujet, mais sans réponse concluante.

En revanche, j’ai fini par découvrir que le site officiel publie toutes les 4 heures un fichier Excel contenant les données de navigation et les coordonnées des bateaux. 

![tableur](https://cdn.geotribu.fr/img/articles-blog-rdp/articles/2024/vendee_globe_donnees_sig/tableur.png)

Ce fichier communique chaque jour les positions à 2h, 6h, 10h, 14h, 18h et 22h, avec un retard de 1h. Par exemple, le fichier de 10h est fourni à 11h (c’est un élément qui sera à prendre en compte dans l’industrialisation du processus).

Ce tableau contient le rang, le nom du bateau et du skipper, mais également la vitesse et le cap sur les dernières 30 min, les dernières 24h et depuis le dernier pointage.

À partir de ce tableur, le but sera donc de construire des données géographiques de la course, que ce soit pour tracer la trajectoire, mais aussi pour agréger tous les pointages.

## Les étapes à suivre

Il faut commencer par récupérer les informations relatives aux positions des bateaux. Cela signifie télécharger les fichiers Excel, car le site ne permet pas de les récupérer en masse. J'ai donc étudié la structure de l'URL pour comprendre comment elles étaient générées et ainsi pouvoir reconstruire ces liens de téléchargement.

Ensuite, il est nécessaire de traiter la manière dont les données de localisation sont présentées. En effet, les positions des bateaux sont souvent fournies sous un format de coordonnées géographiques en degrés, minutes et secondes (DMS). Bien que ce format soit utile, il n'est pas directement compatible avec les outils de géomatique. Il est donc indispensable de les convertir en degrés décimaux, un format plus standard et précis, qui permet de travailler facilement avec des cartes et des systèmes d'information géographique (SIG).

Enfin, il est important d'exporter ces données SIG dans un format compatible, comme le Geopackage ou le GeoJSON. Une fois converties, ces données peuvent être utilisées dans n'importe quel SIG, qu'il s'agisse d'un SIG bureautique comme QGIS ou d'une carte web SIG avec des outils comme MapLibre ou Leaflet.

## Industrialiser la méthode

Pour automatiser le processus décrit ci-dessus, j’ai créé un [projet GitHub](https://github.com/florentfgrs/Vendee-Globe-2024) qui automatise ces tâches avec des scripts Python. Il fonctionne en lignes de commande, et elles sont pour le moment au nombre de deux (voir plus bas).

Pour le téléchargement, j’utilise la bibliothèque `requests`.

Pour la lecture du tableur, le nettoyage des données et la création de géométrie, j’utilise `pandas`, `geopandas` et `shapely`. Il y a un peu de nettoyage de données à faire, car les cellules contiennent des sauts de ligne.

Pour aller plus dans le détail technique, une fois le fichier téléchargé, les étapes successives sont :

1. **Ouverture du fichier dans un dataframe en ne gardant que les colonnes et les lignes qui nous intéressent.**  
Il s'agit de [charger](https://github.com/florentfgrs/Vendee-Globe-2024/blob/main/src/processor.py#L37) le fichier Excel et d'extraire dans un dataframe, on garde uniquement les données pertinentes pour la suite du traitement, tout en ignorant les informations superflues.

2. **Création des en-têtes ([headers](https://github.com/florentfgrs/Vendee-Globe-2024/blob/main/src/processor.py#L69)).**  
Les en-têtes du fichier Excel sont souvent constitués de cellules fusionnées, ce qui rend leur récupération difficile. De plus, les noms de colonnes sont parfois trop verbeux, il faut donc les simplifier pour les rendre plus exploitables.

3. **[Nettoyage des données.](https://github.com/florentfgrs/Vendee-Globe-2024/blob/main/src/processor.py#L72)**  
Cette étape consiste à supprimer les sauts de ligne, les caractères spéciaux ou toute autre anomalie qui pourrait perturber le traitement des données.

4. **Création du timestamp.**  
Un timestamp doit être [généré](https://github.com/florentfgrs/Vendee-Globe-2024/blob/main/src/processor.py#L79) pour chaque pointage afin de pouvoir suivre l'évolution de la position des bateaux au fil du temps. Il sera également utile pour construire la trajectoire.

5. **Conversion des colonnes latitude et longitude de degrés DMS vers degrés décimaux.**  
Il faut d'abord [parser](https://github.com/florentfgrs/Vendee-Globe-2024/blob/main/src/processor.py#L84) les coordonnés pour obtenir les degrés, minutes, secondes et orientation. Puis faire la [conversion](https://github.com/florentfgrs/Vendee-Globe-2024/blob/main/src/processor.py#L9).

6. **Création de la géométrie.**  
À partir des coordonnées converties, il faut [créer](https://github.com/florentfgrs/Vendee-Globe-2024/blob/main/src/processor.py#L106-L116) des géométries. Cela consiste à générer des points pour les positions des bateaux (lors des pointages) ou des lignes pour tracer les trajectoires.

7. **Exportation vers un format SIG vectoriel.**  
[Export](https://github.com/florentfgrs/Vendee-Globe-2024/blob/main/src/processor.py#L155) vers le format [Geopackage](https://www.geopackage.org/).


Pour l’instant, ce projet propose deux fonctionnalités :

### Obtenir le dernier pointage

Il s’agit d’une couche de points indiquant la dernière position communiquée des concurrents. Le format obtenu est un geopackage.

```shell
python get_last_ranking.py --output-dir ./data_vg
```

Le résultat obtenu est une couche de points du dernier pointage en date. Par exemple, si j'exécute cette ligne de commande à 14h45, j'aurai le pointage de 10h (et non celui de 14h à cause du décalage de publication de 1h).

Une fois affiché dans QGIS et avec un peu de travail sur le style, voici le résultat :

![dernier_pointage](https://cdn.geotribu.fr/img/articles-blog-rdp/articles/2024/vendee_globe_donnees_sig/dernier_pointage.png)

!!! tip "Expression QGIS pour afficher uniquement le nom des navigateurs sur les étiquettes"
	```python
	regexp_substr("bateau", '^[^-]+')
	```

### Obtenir l’intégralité des pointages et la trace depuis le départ

Il s’agit d’une couche de points indiquant tous les pointages de chaque bateau depuis le départ, ainsi qu’une couche de lignes reliant ces points pour former la trajectoire des bateaux. Le format obtenu est également un geopackage.

```bash
python get_race_progress.py --output-dir ./data_vg
```

On obtient un geopackage qui contient deux couches :

- Une couche de l'historique de tous les pointages depuis le départ.
- Une couche de ligne qui reproduit la trajectoire de chaque bateau.

![trajectoire](https://cdn.geotribu.fr/img/articles-blog-rdp/articles/2024/vendee_globe_donnees_sig/trajectoire.png)

### Les données attributaires

Dans les deux fonctionnalités ont retoruvent la table atttributaire des couches toutes les informations du tableur. J'ai seulement ajouter une colonne timestamp, elle est utilise pour relier les pointages entre eux et créer la couche des trajectoire.

![table attributaire](https://cdn.geotribu.fr/img/articles-blog-rdp/articles/2024/vendee_globe_donnees_sig/table_attrib.png)

!!! info "Dans les noms de colonnes les prefixe signifie"
	- `30m` = Depuis 30 minutes
	- `last_rank` = Depuis le pointage précédent
	- `24h` = Depuis 24h

Peut être faudrait-il enlever les unités dans les données pour avoir des valeurs numériques ? Dans ce cas, il faudrait peut-être ajouté les unités dans les noms de colonnes. C'est un des pistes d'amélioration. J'aimerais aussi séparer le nom du skipper et le nom du bateau dans deux colonnes distinct. Les contributions pour améliorés ce code sont les bienvenues.

## Pour aller plus loin

Cette première étape n’est qu’un POC (Proof of Concept) le code peut encore être optimisé et je vais continuer de le faire tout au long de la course (en espérant que le formalisme et les horaires de publication du tableur ne change pas). Par la suite, plusieurs idées pourraient être explorées. Je vais surement exploré l'une d'entre elle.

- **Créer un plugin QGIS** : Un plugin QGIS pourrait permettre de charger le classement, la dernière position des navires, et leur trajectoire. On pourrait imaginer que le post-traitement du fichier Excel vers des données SIG soit effectué par l’intégration continue (CI) et exporté en GeoJSON, et que le plugin charge ces GeoJSON hébergés dans le projet GitHub.
  
- **Fournir les données via une API** : On pourrait imaginer un projet qui récupère automatiquement ces données, les convertit et les structure, puis expose une API qui fournit une position ou une trajectoire en fonction du numéro d’un concurrent, par exemple.

- **Créer une application web cartographique** pour visualiser l'avancé des bateaux avec plus de pssibilité que ce que propose l'interface cartographique officiel. J'avais imaginé utilisé [mviewer](https://mviewer.github.io/fr/) pour cela.
