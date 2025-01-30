# qualimarc-docker

[![Docker Pulls](https://img.shields.io/docker/pulls/abesesr/qualimarc.svg)](https://hub.docker.com/r/abesesr/qualimarc/)

Qualimarc est l'outil de diagnostic des notices du Sudoc.

![qualimarc](https://user-images.githubusercontent.com/328244/203315079-4cabb49a-58a8-4778-80b5-d789e48fb94d.PNG)

Ce dépôt contient la configuration docker 🐳 pour déployer l'application qualimarc (cf sources de l'[api](https://github.com/abes-esr/qualimarc-api) et du [front](https://github.com/abes-esr/qualimarc-front)) en local sur le poste d'un développeur, ou bien sur les serveurs de dev, test et prod. 

## URLs de qualimarc

Les URLs correspondantes aux déploiements en local, dev, test et prod de qualimarc sont les suivantes :

- dev :
  - https://qualimarc-dev.sudoc.fr : homepage de qualimarc (qualimarc-front)
  - https://qualimarc-dev.sudoc.fr/api/v1/143519379 : API de qualimarc (qualimarc-api)
- test :
  - https://qualimarc-test.sudoc.fr : homepage de qualimarc (qualimarc-front)
  - https://qualimarc-test.sudoc.fr/api/v1/143519379 : API de qualimarc (qualimarc-api)
- prod
  - https://qualimarc.sudoc.fr : homepage de qualimarc (qualimarc-front)
  - https://qualimarc.sudoc.fr/api/v1/143519379 : API de qualimarc (qualimarc-api)

## Prérequis

Disposer de :
- ``docker``
- ``docker-compose``

## Installation

Déployer la configuration docker dans un répertoire :
```bash
# adaptez /opt/pod/ avec l'emplacement où vous souhaitez déployer l'application
cd /opt/pod/
git clone https://github.com/abes-esr/qualimarc-docker.git
```

Configurer l'application depuis l'exemple du [fichier ``.env-dist``](./.env-dist) (ce fichier contient la liste des variables avec des explications et des exemples de valeurs) :
```bash
cd /opt/pod/qualimarc-docker/
cp .env-dist .env
# personnaliser alors le contenu du .env
```

**Note : les mots de passe de la base de donnée xml de test ne sont pas présent dans le fichier au moment de la copie. Vous devez aller les renseigner manuellement en editant le fichier dans la console avec nano par exemple**

Démarrer l'application :
```bash
cd /opt/pod/qualimarc-docker/
docker-compose up -d
```

Pour information, une base de données postgresql vide sera alors automatiquement initialisée. Ses données binaires seront placées dans le répertoire persistant suivante (attention le user unix de ce répertoire est celui du conteneur postgresql qui n'est pas le même que celui que vous utilisez pour installer l'application) : ``/opt/pod/qualimarc-docker/volumes/qualimarc-db/pgdata/``

## Démarrage et arrêt

```bash
# pour démarrer l'application (ou pour appliquer des modifications 
# faites dans /opt/pod/qualimarc-docker/.env)
cd /opt/pod/qualimarc-docker/
docker-compose up -d
```

Remarque : retirer le ``-d`` pour voir passer les logs dans le terminal et utiliser alors CTRL+C pour stopper l'application

```bash
# pour stopper l'application
cd /opt/pod/qualimarc-docker/
docker-compose stop


# pour redémarrer l'application
cd /opt/pod/qualimarc-docker/
docker-compose restart
```

## Supervision

```bash
# pour visualiser les logs de l'appli
cd /opt/pod/qualimarc-docker/
docker-compose logs -f --tail=100
```

Cela va afficher les 100 dernière lignes de logs générées par l'application et toutes les suivantes jusqu'au CTRL+C qui stoppera l'affichage temps réel des logs.


## Configuration

Pour configurer l'application, vous devez créer et personnaliser un fichier ``/opt/pod/qualimarc-docker/.env`` (cf section [Installation](#installation)). Les paramètres à placer dans ce fichier ``.env`` et des exemples de valeurs sont indiqués dans le fichier [``.env-dist``](https://github.com/abes-esr/qualimarc-docker/blob/develop/.env-dist)

### Spécificité pour la mise à jour de POSTGRES_PASSWORD

Pour modifier la valeur du mot de passe de la base de données postgresql de qualimarc sur une base de données déjà initialisée (c'est à dire que le conteneur ``qualimarc-db`` a déjà été lancé une première fois), il est nécessaire de procéder en deux étapes :
1) modifier la valeur de ``POSTGRES_PASSWORD`` dans votre fichier ``.env``
2) lancer la commande suivante pour mettre à jour le mot de passe à l'interrieur de la base de données déjà initialisée (cela suppose que le conteneur ``qualimarc-db``  soit UP), et bien sur adaptez la valeur du mot de passe à la place de la chaine de caractères "qualimarcsecret2" donnée pour exemple ci-dessous :
   ```bash
   docker exec qualimarc-db psql -U qualimarc -c "alter user qualimarc with password 'qualimarcsecret2';"
   ```
3) relancer l'application pour que la variable ``POSTGRES_PASSWORD`` soit injectée dans tous les conteneurs qui en ont besoin : 
   ```bash
   docker-compose up -d
   ```

A noter que la valeur de ``POSTGRES_USER`` ne doit pas être modifiée car la mise à jour ne serait alors pas prise en compte par ``qualimarc-db``.

## Déploiement continu

Les objectifs des déploiements continus de qualimarc sont les suivants (cf [poldev](https://github.com/abes-esr/abes-politique-developpement/blob/main/01-Gestion%20du%20code%20source.md#utilisation-des-branches)) :
- git push sur la branche ``develop`` provoque un déploiement automatique sur le serveur ``diplotaxis1-dev``
- git push (le plus couramment merge) sur la branche ``main`` provoque un déploiement automatique sur le serveur ``diplotaxis1-test``
- git tag X.X.X (associé à une release) sur la branche ``main`` permet un déploiement (non automatique) sur le serveur ``diplotaxis1-prod``

Qualimarc est déployé automatiquement en utilisant l'outil watchtower. Pour permettre ce déploiement automatique avec watchtower, il suffit de positionner à ``false`` la variable suivante dans le fichier ``/opt/pod/qualimarc-docker/.env`` :
```env
QUALIMARC_WATCHTOWER_RUN_ONCE=false
```

Le fonctionnement de watchtower est de surveiller régulièrement l'éventuelle présence d'une nouvelle image docker de ``qualimarc-api`` et ``qualimarc-front``, si oui, de récupérer l'image en question, de stopper le ou les les vieux conteneurs et de créer le ou les conteneurs correspondants en réutilisant les mêmes paramètres ceux des vieux conteneurs. Pour le développeur, il lui suffit de faire un git commit+push par exemple sur la branche ``develop`` d'attendre que la github action build et publie l'image, puis que watchtower prenne la main pour que la modification soit disponible sur l'environnement cible, par exemple la machine ``diplotaxis1-dev``.

Le fait de passer ``QUALIMARC_WATCHTOWER_RUN_ONCE`` à false va faire en sorte d'exécuter périodiquement watchtower. Par défaut cette variable est à ``true`` car ce n'est pas utile voir cela peut générer du bruit dans le cas d'un déploiement sur un PC en local.

## Sauvegardes

Les éléments suivants sont à sauvegarder:
- ``/opt/pod/qualimarc-docker/.env`` : contient la configuration spécifique de notre déploiement
- ``/opt/pod/qualimarc-docker/volumes/qualimarc-db/dump/`` : contient les dumps quotidiens de la base de données postgresql de qualimarc

Le répertoire suivant est à exclure des sauvegardes :
- ``/opt/pod/qualimarc-docker/volumes/qualimarc-db/pgdata/`` : contient les données binaires de la base de données postgresql qualimarc

### Restauration depuis une sauvegarde

- #### Si on a plus aucun contenair ni de docker-compose.yml

Dans ce cas, cloner ce projet sur un serveur ayant acces au commande docker et git,
et demander au SIRE s'ils ont acces à une sauvegarde du ``.env`` afin de mettre ce fichier ``.env`` dans le repertoire ``qualimarc-docker``.  

- #### Si la base de donnée qualimarc-db est vide

Dans ce cas, si le repertoire ``../volumes/qualimarc-db/dump/`` n'est pas disponible, alors il faut lancer uniquement les conteneurs ``qualimarc-db`` et ``qualimarc-db-dumper`` pour qu'il créé et s'associe au repertoire :
   ```bash
   docker-compose up -d qualimarc-db qualimarc-db-dumper
   ```
Ensuite si le repertoire ``../volumes/qualimarc-db/dump/`` est disponible, demander au SIRE s'ils ont acces à une sauvegarde des dumps de la base de donnée. Ensuite prendre le dumps de la plus fiable et récente et la placer dans le repertoire ``../volumes/qualimarc-db/dump/`` (Le dump devrait se nommer sous cette forme là ``pgsql_qualimarc_qualimarc-db_20220801-143201.sql.gz``).
Une fois le dump placé, lancez uniquement les conteneurs ``qualimarc-db`` et ``qualimarc-db-dumper`` s'ils sont pas déjà lancé :
   ```bash
   docker-compose up -d qualimarc-db qualimarc-db-dumper
   ```
lancez le script de restauration ``restore`` comme ceci et suivez les instructions :
   ```bash
   docker exec -it qualimarc-db-dumper restore
   ```
pour pouvoir executer en ignorant le mode interactif en utilisant la syntaxe suivante:
   ```bash
   docker exec -it qualimarc-db-dumper bash -c 'restore <filename_du_dump> $DB_TYPE $DB_HOST $DB_NAME $DB_USER $DB_PASS 5432'
   ```
C'est bon, la base de données qualimarc est alors restaurée

- #### Verifier si tout marche
Lancez alors toute l'application qualimarc et vérifiez qu'elle fonctionne bien :
```bash
cd /opt/pod/qualimarc-docker/
docker-compose up -d
```

## Développements

### Admin de postgresql
Pour consulter l'interface d'admin web de postgresql (basée sur [Adminer](https://www.adminer.org/)) rendez vous sur cette URL : 
- local : http://127.0.0.1:11082/


Vous devriez obtenir une interface qui ressemble à ceci:  
![image](https://user-images.githubusercontent.com/328244/178326892-25ad4218-02e6-4f61-aab9-9ea1a94b8e6b.png)


### Mise à jour de la dernière version

Pour récupérer et démarrer la dernière version de l'application vous pouvez le faire manuellement comme ceci :
```bash
docker-compose pull
docker-compose up
```
Le ``pull`` aura pour effet de télécharger l'éventuelle dernière images docker disponible pour la version glissante en cours (ex: ``develop-api`` ou ``main-api``). Sans le pull c'est la dernière image téléchargée qui sera utilisée.

Ou bien [lancer le conteneur ``qualimarc-watchtower``](https://github.com/abes-esr/qualimarc-docker/blob/develop/README.md#d%C3%A9ploiement-continu) qui le fera automatiquement toutes les quelques secondes pour vous.

## Architecture

<img alt="schéma architecture" src="https://docs.google.com/drawings/d/e/2PACX-1vQE2ImlIkdPWhz_mUH3LeGg-tAUtiqTzJXx4zP5UjHY75Cl9Snw2gj1M0ww6iYJf_kM-gBtLGdJcCgb/pub?w=1332&amp;h=645">

([lien](https://docs.google.com/drawings/d/1ki8ih3egccbf1SBsW4uDTAp7v5hCThQijG3ghki_d9c/edit) pour modifier le schéma)

Les codes de source de qualimarc sont ici :
- https://github.com/abes-esr/qualimarc-api : code source de l'API de qualimarc
- https://github.com/abes-esr/qualimarc-front : code source de l'interface utilisateur de qualimarc
