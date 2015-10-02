---
layout: post
title:  "Construire une image Docker dans une intégration continue"
date:   2015-08-17 18:45:00
comments: true
categories: [ci]
tags: [java, docker, shippable]
redirect_from: 2015/08/17/build-docker-image-with-ci/
permalink: construire-image-docker-dans-une-ic
---

Je voulais voir s'il était possible de construire une image Docker à la suite d'un build réussi en utilisant des outils disponibles sur le net. 
J'ai enfin réussi à faire ce que je voulais et je présente ici ce que j'ai fait.

# Objectif
Mon objectif était de construire une image Docker à la suite d'un build CI réussi. Le build CI est déclenché à chaque push. De plus je 
voulais pouvoir construire l'image sur mon poste pour des tests local.

# Outils
Les outils que je vais utiliser sont

  * [gradle-docker](https://github.com/Transmode/gradle-docker) 
  * [Shippable](http://www.shippable.com) 
  * [tutum.co](https://www.tutum.co/)

## Gradle-docker
Il s'agit d'un plugin Gradle permettant de construire une image Docker à partir du script de build. On peut fournir un fichier ``Dockerfile``
au plugin ou en générer un à partir du script.

## Shippable
[Shippable](http://www.shippable.com) est une plateforme d'intégration continue. Par rapport à [Travis](https://travis-ci.org/) cette 
plateforme se base sur Docker : l'environnement créé pour le build est une image Docker. Il est possible d'utiliser les images Docker par défaut 
mais on peut aussi fournir notre propre image. De plus, il est possible de construire une image Docker pendant le build.

L'option gratuite permet de lancer qu'un seul build à la fois.

## Tutum.co
[Tutum](https://www.tutum.co) est une plateforme qui permet la gestion de déploiement de containers Docker dans le cloud. Le produit permet 
de déployer des conteneurs dans différents cloud comme [Amazon](https://aws.amazon.com/fr/), [Digital Ocean](https://www.digitalocean.com/), ... 

L’inscription est gratuite pour l’instant et le restera par la suite lors de la fin de la beta.

# Construction de l'image Docker en local

L'image Docker est construite par Gradle grâce au plugin [gradle-docker](https://github.com/Transmode/gradle-docker). 
La déclaration dans le fichier ``build.gradle``
 
```groovy
buildscript {
    ...
    dependencies {
        ...
        classpath 'se.transmode.gradle:gradle-docker:1.2'
    }
}
...
apply plugin: 'docker'
...

task buildDocker(type: Docker, dependsOn: build) {
    project.group = "tutum.co/jgiovaresco"
    applicationName = jar.baseName
    dockerfile = file('Dockerfile')
    doFirst {
        copy {
            from jar
            into "${stageDir}/buildoutput"
        }
    }
}
```

Une nouvelle tâche ``buildDocker`` est ajoutée et celle-ci dépend de la tâche ``build``. Pour construire l'image Docker, le plugin va d'abord copier le fichier ``Dockerfile`` fournit dans le répertoire de travail ``build/docker``. Le plugin lance ensuite la commande ``docker build`` depuis ce répertoire.

L'image à construire doit contenir le jar de l'application construit par Gradle. Ce jar est donc copié avant de construire l'image grâce au bloc ``doFirst{}``. Le jar est copié dans un sous répertoire ``buildoutput`` pour que la construction avec Shippable fonctionne.

On construit l'image sur son poste avec ``gradle buildDocker``

# Build avec Shippable

On commence par lier notre compte Shippable avec notre compte Tutum pour que l'image Docker soit publiée dans le registry privé. Le doc décrit la démarche : [Use a Docker registry](http://docs.shippable.com/docker_registries/)

La création d'un build est aussi très bien expliquée dans la documentation : [Run A Build With Test And Coverage Reports](http://docs.shippable.com/build_case2/). Pour rappel, le process de build fournit par Shippable se trouve [ici](http://docs.shippable.com/basic_flow/).

Pour construire un image Docker dans un build, il est nécessaire de configurer le projet dans Shippable. Comme le décrit la documentation [Docker build](http://docs.shippable.com/docker_build/), il y a 2 workflows  : 

* Pre-CI :
  * Construction de l'image Docker en utilisant le fichier Dockerfile présent à la racine du repository.
  * Récupération du code depuis GitHub/Bitbucket et lance les tests dans le conteneur issue de l'image construite précédemment.
  * Pousse le conteneur dans le registry (Docker Hub, GCR, Quay.io, Private Registry)

* Post-CI :
  * Récupération de l'image Docker spécifiée dans la configuration du projet depuis le registry (Dockery Hub, GCR, Quay.io)
  * Récupération du code depuis GitHub/Bitbucket et lance les tests dans le conteneur issue de l'image récupéré précédemment.
  * Si le build réussi, construit le conteneur Docker  depuis le fichier Dockerfile présent à la racine du repository.
  * Pousse le conteneur dans le registry (Docker Hub, GCR, Quay.io, Private Registry)

J'ai utilisé le 2ème workflow.  L'image utilisée pour lancer mon build est une image fournie par Shippable ``shippableimages/ubuntu1404_java``. Dans un second temps, je pense que je vais créer ma propre image dans laquelle je pourrais installer les outils nécessaires pour accélérer le build en simplifiant la phase de préparation.

En utilisant l'image par défaut, il faut commencer par configurer la version de Java à utiliser. Pour cela on ajoute dans le bloc ``before_script`` les lignes suivantes : 

```yaml
before_script:
  - if [[ $SHIPPABLE_JDK_VERSION == "oraclejdk8" ]] ; then export JAVA_HOME="/usr/lib/jvm/java-8-oracle"; export PATH="$PATH:/usr/lib/jvm/java-8-oracle/bin"; export java_path="/usr/lib/jvm/java-8-oracle/jre/bin/java"; fi
  - update-alternatives --set java $java_path
  - java -version
```

Pour inclure le jar construit dans l'image, il est nécessaire de le copier dans un répertoire particulier ``shippable/buildoutput``. Il faut donc créer ce répertoire dans la phase ``before_script`` et copier le jar construit dans la phase ``after_script``.
La construction de l'image Docker se fera automatiquement à la fin du build.

Pour finir je rappel le contenu du fichier ``shippable.yml`` que j'utilise

```yaml
language: java

jdk:
  - oraclejdk8

before_script:
  - if [[ $SHIPPABLE_JDK_VERSION == "oraclejdk8" ]] ; then export JAVA_HOME="/usr/lib/jvm/java-8-oracle"; export PATH="$PATH:/usr/lib/jvm/java-8-oracle/bin"; export java_path="/usr/lib/jvm/java-8-oracle/jre/bin/java"; fi
  - update-alternatives --set java $java_path
  - java -version
  - mkdir -p shippable/buildoutput
  - mkdir -p shippable/testresults
  - mkdir -p shippable/codecoverage
  - wget -O shippable/cover2cover.py https://raw.githubusercontent.com/rix0rrr/cover2cover/master/cover2cover.py

script:
  - ./gradlew build

after_script:
  - cp ./build/test-results/*.xml ./shippable/testresults
  - cp ./build/libs/sm-identityaccess.jar ./shippable/buildoutput

after_success:
  - ./gradlew jacocoTestReport
  - python shippable/cover2cover.py ./build/reports/jacoco/test/jacocoTestReport.xml src/main/java > shippable/codecoverage/coverage.xml
```

On peut trouver un exemple sur mon [Github](https://github.com/jgiovaresco/sm-identityaccess)
