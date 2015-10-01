---
layout: post
title:  "Accès aux conteneurs Docker par leur nom"
date:   2015-09-27 14:10:00
categories: []
tags: [docker]
redirect_from: 2015/09/27/access-docker-container-with-their-name/
permalink: acces-aux-conteneurs-Docker-par-leur-nom
---

Interroger les conteneurs Docker peut être pénible, en effet il n'est pas possible de fixer une ip statique pour ces conteneurs. 
Le top serait de pouvoir utiliser un nom pour accéder à ces conteneurs, genre avec DNS... Et bien c'est possible ! J'ai découvert cette possibilité en déployant [Tuleap](http://tuleap.net) sur mon poste.

## dnsdock
La solution consiste à utiliser un conteneur Docker ([dnsdock](https://github.com/tonistiigi/dnsdock)) qui va repérer la création et la suppression d'autres conteneurs pour enregistrer leur nom dans un service DNS.

### Installation
On lance le conteneur avec la commande suivante : 

```
docker run -d -v /var/run/docker.sock:/var/run/docker.sock --name dnsdock -p 172.17.42.1:53:53/udp tonistiigi/dnsdock
```

L'ip fournie à la commande correspond à l'ip de l'interface ``docker0``. Cette ip n'étant pas statique, elle est susceptible de changer. Cependant il est possible de la fixer en modifiant la configuration service docker.

Sous Debian/Jessie, on modifie le script d'initialisation du service systemd associé à Docker. 
Modifier directement le fichier ``/lib/systemd/system/docker.service`` n'est pas recommandé car celui-ci sera écrasé lors des mises à jour du paquet. Il faut donc copier le fichier dans ``/etc/systemd/system``, systemd utilisera cette configuration à la place de celle par défaut.

Pour fixer l'adresse ip de l'interface ``docker0``, on ajoute les arguments suivants au démarrage du service ``--bip=172.17.42.1/24 --dns=172.17.42.1`` dans la section ``ExecStart``.

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket
Requires=docker.socket

[Service]
Type=notify
ExecStart=/usr/bin/docker daemon -H fd:// --bip=172.17.42.1/24 --dns=172.17.42.1
MountFlags=slave
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

### Lancement automatique du conteneur
Ce conteneur doit être systématiquement démarré car la résolution DNS dans les conteneurs passera par lui. Pour lancer automatiquement le conteneur, on peut

* utiliser l'option ``--restart=always`` lorsqu'on démarre le conteneur
* créer un nouveau service systemd

Comme précisé dans la [doc](https://docs.docker.com/articles/host_integration/#using-a-process-manager), l'option ``--restart`` peut entrer en conflit avec notre process manager (systemd). Nous allons donc créer un nouveau service en créant un fichier ``dnsdock.service`` dans le répertoire ``/etc/systemd/system``

```
[Unit]
Description=dnsdock container
Documentation=https://github.com/tonistiigi/dnsdock
After=docker.service
Requires=docker.service

[Service]
# Turn off timeouts 
TimeoutStartSec=0
# Enable automatic restart when container stops
Restart=always 
# Start dnsdock container when service starts
ExecStart=/usr/local/bin/systemd-docker run --rm -v /var/run/docker.sock:/var/run/docker.sock --name %n -p 172.17.42.1:53:53/udp tonistiigi/dnsdock 
Type=notify
NotifyAccess=all

[Install]
WantedBy=multi-user.target
``` 

Nous n'utilisons pas directement la commande ``docker`` pour lancer le conteneur car sinon systemd surveillerait le client au lieu de surveiller le conteneur. 
Ainsi si le client est détaché du conteneur pour n'importe quelle raison (par exemple un problème réseau), systemd tuera le conteneur même si celui-ci fonctionne correctement. Reciproquement si le conteneur s'arrête mais que le client fonctionne toujours, systemd ne fera rien. 
En utilisant le script [systemd-docker](https://github.com/ibuildthecloud/systemd-docker) nous nous assurons que systemd surveille correctement le conteneur.

De plus il ne faut surtout pas utiliser l'option `-d` car cela empêcherait de lancer le conteneur en tant que processus enfant du pid associé à l'exécution de la commande définie par ``ExecStart``. systemd ne pourra pas déterminer la bonne exécution de la commande et pensera donc que le processus s'est terminé.

### Démarrer le service systemd

Le service se démarre avec 

```
sudo systemctl start dnsdock.service 
```

Pour activer le service au démarrage, il faut lancer 

```
sudo systemctl enable dnsdock.service
```


### Modifier sa configuration réseau
Ne faut pas oublier de modifier sa configuration réseau pour ajouter l'ip ``172.17.42.1`` en tant que serveur DNS.

### Plus d'informations 
[Getting started with systemd](https://coreos.com/docs/launching-containers/launching/getting-started-with-systemd/)

[Running docker containers with systemd](http://container-solutions.com/running-docker-containers-with-systemd/)