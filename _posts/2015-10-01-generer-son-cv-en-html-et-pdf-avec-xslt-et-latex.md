---
layout: post
title:  "Générer son CV en HTML et en PDF avec XSLT et LaTeX"
date:   2015-10-01 21:26:00
comments: true
categories: []
tags: [docker, latex]
redirect_from: 2015/10/01/generer-son-cv-en-html-et-pdf-avec-xslt-et-late/x
permalink: generer-son-cv-en-html-et-pdf-avec-xslt-et-latex
---

Depuis que j'ai mis en ligne mon site personnel pour publier mon CV, chaque mise à jour du CV me demandait deux fois plus d'effort : je devais mettre à jour le site et il ne fallait pas oublier la version fichier (Word à l'époque).

Pour me faciliter la vie, je voulais modifier une seule fois mon CV et générer ensuite automatiquement la page du site et le fichier. Je suis arrivé à la solution suivante : un CV au format XML et 2 feuilles XSLT permettant de générer

* la page du site
* le fichier PDF

## Html
La transformation vers le HTML était la partie facile : un script PHP transforme le CV au format XML en un document HTML. Le code est visible [ici](https://github.com/jgiovaresco/julien.giovaresco.fr/blob/master/resume.php)

## Pdf
Pour générer le fichier, ne sachant pas générer directement du PDF à partir du XML, je suis passé par un document LaTeX. La feuille XSLT génère un document TeX et celui-ci est ensuite compilé pour obtenir un fichier PDF.

## Créer son CV en LaTeX 
J'ai utilisé la libraire [moderncv](https://github.com/xdanaux/moderncv) pour créer mon CV en LaTeX. Différents modèles sont disponibles et une fois qu'on a pris le coup, LaTeX nous facilite la vie question mise à en page...
Le plus pénible reste l'installation de l'environnement LaTeX, jusqu'à l'arrivée de Docker ! 

### Texmaker
Dans un premier temps, j'ai créé une image contenant le logiciel [TexMaker](http://www.xm1math.net/texmaker/index_fr.html). Pour cela je me suis inspiré du dépôt Git de [Jessie Frazzle](https://twitter.com/frazelledazzell) contenant pleins d'images permettant de "conteneuriser" des applications (Desktop ou autre) : [dockerfiles](https://github.com/jfrazelle/dockerfiles).

L'image se trouve sur DockerHub ([jgiovaresco/texmaker](https://hub.docker.com/r/jgiovaresco/texmaker/)) et son Dockerfile se trouve sur [GitHub](https://github.com/jgiovaresco/dockerfiles/blob/master/texmaker/Dockerfile).

Pour l'utiliser on peut créer un alias 

```
alias texmaker='docker run -it --rm=true -e USER=$USER -e USERID=$UID -v /tmp/.X11-unix:/tmp/.X11-unix -v $HOME:/home/texmaker -e DISPLAY=unix$DISPLAY  --device /dev/dri --name texmaker jgiovaresco/texmaker'
```

### Automatiser la génération
L'utilisation de TexMaker est pratique pour modifier le résultat de la feuille XSLT et ainsi faire des ajustements mais la génération PDF reste manuelle. Une fois que le fichier TeX généré par la feuille XSLT est correct il est intéressant d'automatiser la génération du PDF.

Pour cela j'ai créé un Makefile contenant 3 tâches :

* une tâche **clean** pour nettoyer la génération précédente
* une tâche **tex**  pour générer le fichier TeX à partir du fichier XML
* une tâche **pdf** pour générer le PDF

#### Target _tex_
Pour générer le fichier TeX, j'ai créé une image Docker ([jgiovaresco/saxonhe](https://hub.docker.com/r/jgiovaresco/saxonhe/)) dans laquelle [SaxonHE](http://saxon.sourceforge.net/) est installé et configuré comme ENTRYPOINT. L'exécution d'un conteneur basé sur cette image équivaut à l'exécution directe de SaxonHE.

```
docker run --rm=true -e USER=$USER -e USERID=$UID -v /tmp:/data --name saxonhe jgiovaresco/saxonhe \
    -ext:off -s:myfile.xml -xsl:myfile.xsl -o:myfile.tex
```

Avec le paramètre ``-v /tmp:/data``, le répertoire ``/tmp`` est bindé du host vers le répertoire ``/data`` du conteneur. Les fichiers XML et XSL devront se trouver dans ce répertoire. 
Les paramètres ``-ext:``, ``-s:``, ``-xsl:`` et ``-o:`` correspondent aux paramètres de SaxonHE.

#### Target _pdf_
Pour générer le fichier PDF, j'ai commencé par créer une image Docker contenant tout ce qu'il faut pour compiler du LaTeX ([jgiovaresco/lualatex](https://hub.docker.com/r/jgiovaresco/lualatex/)). Puis à partir de cette image, j'ai créé une autre image qui contient les packages nécessaires à la compilation du CV avec moderncv ([jgiovaresco/moderncv](https://hub.docker.com/r/jgiovaresco/moderncv/)). L'ENTRYPOINT de l'image étant ``lualatex``, il suffit de passer le fichier *.tex en paramètre lors de l'exécution de la commande ``docker run``.

On génère le CV LaTeX en PDF avec la commande

```
docker run --rm=true -e USER=$USER -e USERID=$UID -v /tmp:/data --name moderncv jgiovaresco/moderncv myfile.tex
```

Comme précédemment le répertoire ``/tmp`` est bindé du host vers le répertoire ``/data`` du conteneur, le fichier .tex devra se trouver dans ce répertoire.

### TravisCI
Pour finir, j'ai utilisé [TravisCI](https://travis-ci.org) pour générer le fichier PDF dès qu'un push est effectué sur le dépôt GitHub. Depuis peu, il est possible d'exécuter les commandes Docker dans un build Travis. 

Une fois la génération du PDF effectuée, je voulais déposer le fichier sur mon serveur afin de pouvoir mettre un lien de téléchargement sur mon site personnel. Après un détour sur la documentation de Travis, j'ai vu [ici](http://docs.travis-ci.com/user/deployment/custom/) qu'il était possible de lancer une commande FTP.
On évite de mettre le login/password en clair dans le fichier ``.travis.yml``. Heureusement il est possible de chiffrer ces valeurs à l'aide de l'outil fournit par Travis (voir la [doc](http://docs.travis-ci.com/user/encryption-keys/)).

```
sudo: required

services:
  - docker

env:
  global:
    - secure: "...."
    - secure: "...."

before_install:
  - chmod go+w  ./latex
  - ls -al latex

script:
  - make pdf

after_success:
  - "curl --ftp-create-dirs -T latex/resume.pdf -u $FTP_USER:$FTP_PASSWORD ftp://FTP.giovaresco.fr/"
```
