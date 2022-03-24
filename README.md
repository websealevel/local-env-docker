# Environnement de développement local Dockerisé derrière un reverse-proxy

- [Environnement de développement local Dockerisé derrière un reverse-proxy](#environnement-de-développement-local-dockerisé-derrière-un-reverse-proxy)
  - [Objectifs](#objectifs)
  - [Mise en place d'un dns local avec `dnsmasq`](#mise-en-place-dun-dns-local-avec-dnsmasq)
  - [Configuration de Traefik](#configuration-de-traefik)
    - [Lancement du conteneur Traefik](#lancement-du-conteneur-traefik)
    - [Intercepter uniquement les reqûetes vers nos conteneurs Docker](#intercepter-uniquement-les-reqûetes-vers-nos-conteneurs-docker)
  - [Tester le bon fonctionnement](#tester-le-bon-fonctionnement)
  - [En résumé](#en-résumé)

Un petit reverse proxy dockerisé et une configuration dns locale pour se faire un environnement de dev docker accueillant

Cet article est tiré d'un article plus complet sur la mise en place d'un starterpack utilisant ce reverse proxy que vous trouverez sur le [dépôt suivant](https://github.com/websealevel/starterpack-front-php-mysql-phpmyadmin).

## Objectifs

Nous l'avons dit dans la [section précédente](#aller-un-peu-plus-loin-acceder-à-nos-conteneurs-via-un-nom-de-domaine) notre starterpack est bien mais on peut faire mieux dans le cas où l'on souhaite travailler sur plusieurs projets en même temps sans avoir à toucher de la config et maintenir des états.

Pour régler ce problème on va utiliser un autre outil, le service [Traefik](https://doc.traefik.io/traefik/). On va s'en servir comme [reverse proxy](https://fr.wikipedia.org/wiki/Proxy_inverse). Il servira d'intermédiaire pour accéder à nos conteneurs Docker.

Illustrons concrètement ce que l'on cherche à faire: je démarre un projet `foobar` avec mon starterpack. J'ai déjà deux autres projets sur lesquels je travaille issus de mon pack. Je m'en soucie pas. J'accède par exemple à mon service `back` depuis mon navigateur en requêtant `back.foobar.test`. Le domaine `.test` [est recommandé](https://fr.wikipedia.org/wiki/.test) car il a été reservé pour offrir un domaine qui ne rentre pas en conflit avec des domaines réels d'Internet. Pour accéder à phpmyadmin de mon projet je tape `phpmyadmin.foobar.test`. Etc... Pratique non ? D'une part je n'ai plus besoin de savoir quels ports sont déjà réservés sur tel ou tel projet ni de les gérer. Enfin `back.foobar.test` est plus explicite que `localhost:90001`. Si je me donne une règle de syntaxe je peux retrouver n'importe quel conteneur de n'importe quel projet facilement six mois plus tard.

Pour y parvenir, on va se servir d'un serveur dns local et d'un reverse-proxy. On va mettre en place un service qui va essayer de résoudre le nom de domaine à partir d'une configuration sur votre machine avant d'interroger un vrai serveur dns d'internet. Quand on tapera l'url `back.foobar.test`, notre système de dns va donc regarder s'il trouve un pattern, ici le domaine `.test` et tous les [sous-domaines associés](https://fr.wikipedia.org/wiki/Nom_de_domaine), par exemple `back.foobar.test`. S'il le trouve, il va rediriger la requête faite depuis notre navigateur vers notre machine au lieu d'aller requêter l'Internet. C'est là que notre reverse proxy rentre en jeu: il va recevoir la requete, et s'il est bien configuré, va résoudre le nom de domaine pour nous servir le conteneur Docker de notre projet. Voilà le plan:

- utiliser le domaine reservé `.test` pour capter tous les sous-domaines (aka tous nos projets de dev) et renvoyer les reqûetes vers notre machine. C'est le job de notre service dns local
- intercepter les requêtes entrant sur notre machine pour les résoudre et les rediriger vers le bon conteneur Docker, par exemple le phpmyadmin d'un de nos projets. C'est le job du [reverse-proxy](https://fr.wikipedia.org/wiki/Proxy_inverse), il agit comme un portique par lequel les requêtes entrantes vont devoir passer pour être traitées selon nos besoins.

Pour mettre en place ce système on va avoir besoin de conteneurs Docker car on va conteneurisé le reverse proxy (et oui, encore, le minimum sur notre machine). Pour cela, on va créer un nouveau dépôt en dehors de notre starterpack. Un projet, un dépôt, c'est la règle. Ce projet vivra sa vie de manière indépendante sur votre machine et pourra servir à tous vos projets en local et non seulement à ceux réalisés avec votre starterpack. Quand on l'aura cablé, on le lancera une fois pour toute et vous n'y retoucherez plus jamais.

Créer donc un autre dépôt sur votre machine, par exemple `local-env-docker` et allons-y.

## Mise en place d'un dns local avec `dnsmasq`

Mettons en place ce système. Je le fais sur Linux, si vous êtes sur un autre OS il faudra trouver des façons équivalentes de faire la même chose. L'idée restera la même.

Dans tous les cas, pour notre dns local on va utiliser [dnsmasq](https://www.linuxtricks.fr/wiki/dnsmasq-le-serveur-dns-et-dhcp-facile-sous-linux) (disponible sous Linux/MacOS, pour Windows un équivalent semble être [Acrylic](http://mayakron.altervista.org/support/acrylic/Home.htm)).

Sur Linux, en tout cas sur Ubuntu, la configuration réseau est gérée par le processus `systemd`. Celui-ci définit `NetworkManager` comme application réseau par défaut. `NetworkManager` gère donc les DNS et le DHCP de votre machine. NetworkManager connait mais n'utilise pas `dnsmasq` par défaut donc il va falloir lui dire. On édite le fichier de configuration `/etc/NetworkManager/NetworkManager.conf` et on ajoute une nouvelle ligne `dns=dnsmasq` dans la section `[main]`. On enregistre la modification.

En principe, la résolution d'URL est gérée par `systemd-resolver`, mais, on va laisser `NetworkManager` s'en occuper afin de permettre à `dnsmasq` d'attraper les URLs qui nous concernent, celles en `.test`, en exécutant la commande suivante :

`sudo rm /etc/resolv.conf ; sudo ln -s /var/run/NetworkManager/resolv.conf /etc/resolv.conf`

Ici on supprime le fichier de configuration par défaut du resolver d'URL et on le remplace par le fichier de configuration de `NetworkManager` via un lien symbolique.

On crée ensuite un fichier de configuration dnsmasq `test-tld` dans le dossier `/etc/NetworkManager/dnsmasq.d/`, en ajoutant le pattern `.test` recherché

`echo 'address=/test/127.0.0.1' | sudo tee /etc/NetworkManager/dnsmasq.d/test-tld`

Ici on map le pattern `.test` à l'ip de notre machine pour qu'une requête comme `foobar.test` revienne vers nous. On redémarre `NetworkManager` pour prendre en compte les modifications

`sudo service NetworkManager reload`

A présent toutes les requêtes émises par notre machine vers les sous-domaines de `.test` devraient être interceptées et redirigées sur elle même (et non vers l'Internet). Testons cela

```
ping foobar.test
ping back.example.test
ping front.projet.test
```

Vous devriez voir que le ping vers n'importe quel sous-domaine de `.test` est bien redirigé vers notre machine, à savoir vers l'ip `127.0.0.1`, `localhost` en somme. Parfait, c'est exactement ce que nous voulions !

Donc dorénavant toutes vos requêtes depuis votre navigateur vers un sous domaine de `.test` n'ira jamais vers l'Internet. Mais si un site web est en `.test` ? Justement non et c'est tout l'intérêt d'utiliser ce nom de domaine: il est reservé pour le test et le développement. Vous êtes donc garantis par les standards de ne jamais perdre accès à un site en `.test` puisqu'il y'en aura jamais.

Notre seule config sur notre machine locale est terminée. A présent nous allons pouvoir mettre en place notre reverse proxy.

## Configuration de Traefik

Je ne suis pas un expert en reverse proxy mais voici en quelques mots à quoi va nous servir [Traefik](https://doc.traefik.io/traefik/getting-started/concepts/). `Traefik` est un service puissant et nous allons seulement en utiliser quelques fonctionnalités. Libre à vous d'explorer ce programme pour aller plus loin dans son usage.

En gros `Traefik` va intercepter les requêtes en `.test` qui retombent sur notre machine et les rediriger vers le bon conteneur. Mais ce qui est top avec [Traefik] c'est qu'il va détecter les nouveaux conteneurs automatiquement et créer les routes pour y accèder (en gros associer une URL à l'IP d'un conteneur Docker). Pas besoin de configurer des routes à chaque fois qu'on démarre un nouveau projet (ou qu'on en supprime un), [Traefik] gère ça pour nous !

### Lancement du conteneur Traefik

Je recommande déjà de suivre la page [Quick Start](https://doc.traefik.io/traefik/getting-started/quick-start/) de la doc de Traefik, vous aurez une base pour votre `docker-compose.yml` et une meilleure idée de son fonctionnement et de ses capacités.

Commençons donc à configurer. Ouvrez la page [Traefik & Docker](https://doc.traefik.io/traefik/providers/docker/), tout ce dont on aura besoin y est, vous pourrez vous y référer si c'est pas clair ce que j'écris. Si vous suivez le guide [Quick Start](https://doc.traefik.io/traefik/getting-started/quick-start/) et vous rendez à l'adresse `http://localhost:8080/api/rawdata` vous verez vos conteneurs docker reconnus par Traefik ainsi que leurs configurations (notamment leur IP). Si vous montez ou fermez des conteneurs vous les verrez ajoutés ou retirés de la liste. Plutôt cool.

Maintenant qu'on est convaincus que Traefik voit nos conteneurs et les prends en charge dynamiquement la question c'est : comment rediriger notre requête par exemple `back.foobar.test` vers le conteneur(service Compose) `back` du projet `foobar` monté sur notre machine locale ?

La première ligne de la page dit "_Attach labels to your containers and let Traefik do the rest!_". Voilà, donc faut qu'on comprenne comment arriver à ça.

Regardons aussi [cette page](https://doc.traefik.io/traefik/getting-started/configuration-overview/) qui donne un aperçu de la configuration de Traefik et comment la _magie opère_.

Déjà on ne veut pas que Traefik intercepte toutes les requêtes entrantes, seulement les requêtes HTTP (nos requêtes en `.test`). Pour cela on va utiliser la directive
`entryPoints` directement dans notre docker-compose.yml, c'est de la [configuration dynamique](https://doc.traefik.io/traefik/getting-started/configuration-overview/#the-dynamic-configuration) car elle va s'adapter à chaque situation. Voici notre service reverse-proxy, en s'inspirant directement de l'[exemple donné dans la doc](https://doc.traefik.io/traefik/user-guides/docker-compose/basic-example/)

```
services:
  reverse-proxy:
    # The official v2 Traefik docker image
    image: traefik:v2.6
    ports:
      # The HTTP port
      - "80:80"
      # The Web UI (enabled by --api.insecure=true)
      - "8080:8080"
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock

    command:
      #- "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
```

On map les ports 80 pour que Traefik écoute toutes les requêtes http entrantes sur notre machine. Le port 8080 est utilisé pour nous donner accès à des UI de Traefik (comme `http://localhost:8080/api/rawdata` que nous avons inspecté juste avant). La partie qui nous intéresse pour la configuration dynamique est sous la clef `command`. Ici on dit

- `--api.insecure=true`, on active l'API de Traefik pour exposer tout un tas d'UI et d'informations. Très utile pour le dev, à désactiver en prod (c'est le cas par défaut)
- `--providers.docker=true`, pas sûr de comprendre exactement mais en gros on dit à Traefik que Docker est utilisé. Donc Traefik va pouvoir requêter l'API de Docker pour pouvoir fonctionner correctement avec Docker
- `providers.docker.exposedbydefault=false`, on dit à Traefik que si un conteneur ne déclare pas explicitement (on le verra après) son envie d'être scanné pour mettre en place le routing automatiquement alors ignore le. Parfait
- `--entrypoints.web.address=:80`, très important. Les [entryPoints](https://doc.traefik.io/traefik/routing/entrypoints/) permettent de maper les ports de notre machine à Traefik pour qu'il se branche dessus. Ici on dit qu'on crée un `entryPoint` appelé `web` et que Traefik écoute le port `80` seulement.

Les `entryPoints` permettent à Traefik de récupérer les requêtes. Maintenant il faut lui dire vers où les diriger. Traefik crée pour chaque conteneur detecté un [routeur](https://doc.traefik.io/traefik/routing/routers/) et un [service](https://doc.traefik.io/traefik/routing/services/). Un _router_ est en charge de rediriger les requêtes entrantes vers le service Traefik qui peut les gérer. C'est un câble tiré entre l'`entryPoint` et le `service Traefik`. Oui il y a un peu de terminologie mais la documentation est vraiment bien faite et accompagnée de schémas en couleur. Un _service Traefik_, à ne pas confondre avec notre service Compose, est quand à lui responsable de définir _comment_ accéder rééelement à nos conteneurs. On verra ça juste après. Pour l'instant on met en place la partie interface entre Traefik et notre machine.

### Intercepter uniquement les reqûetes vers nos conteneurs Docker

Donc là, on a dit de récuperer les requêtes HTTP mais on veut être encore plus restrictif et ne pas interférer avec le trafic sur notre machine, on veut récupérer seulement les requêtes en `.test`.

Ajoutons les configurations suivantes

```
      - "--providers.docker.network=web"
      - "--providers.docker.defaultrule=HostRegexp(`{subdomain:[a-z]+}.test`)"
```

A présent on dit d'utiliser le réseau Docker `web` par défaut (car pourquoi pas). Et surtout on filtre les requêtes en fonction de l'hôte demandé (via la clef `defaultrule`) avec une regex pour ne capter que les noms de domaine en `.test`.

Relancez le projet avec `docker-compose up -d`. Si vous retournez sur `http://localhost:8080/api/rawdata` où sont exposées des infos de Traefik sur l'état des services, vous verrez que le service `whoami` n'est plus visible ! C'est bien ce qu'on voulait. D'ailleurs aucun de nos conteneurs ne sera visible car on a dit que par défaut Traefik ne devait pas les prendre en compte.

Comment ré-intégrer notre service whoami à Traefik ? Pour cela on va ajouter un peu de config sur notre service `whoami` sous la clef `labels`. Labeliser nos conteneurs permet à Traefik de retrouver sa configuration de routing, et donc au final le conteneur ciblé en retrouvant son adresse ip.

```
whoami:
  labels:
    # Explicitly tell Traefik to expose this container
    - "traefik.enable=true"
    # The domain the service will respond to
    - "traefik.http.routers.whoami.rule=Host(`whoami.test`)"
    # Allow request only from the predefined entry point named "web"
    - "traefik.http.routers.whoami.entrypoints=web"
```

Les commentaires sont assez clairs ici. Sur la première ligne on expose le conteneur de manière explicite. La ligne importante est `traefik.http.routers.whoami.rule=Host(whoami.test)`. Il attribue un nom de domaine au service de Traefik. On prendre bien soin de mettre ce nom en `.test`.

## Tester le bon fonctionnement

Visitez à présent `whoami.test` depuis votre navigateur favori. Vous devriez tomber sur la réponse du service comme attendu.

## En résumé

Donc pour résumer quand je taperai `whoami.test` dans mon navigateur:

- ma configuration dns locale va repérer le `.test` et rediriger la reqûete vers ma machine, sur le port `80`
- Traefik, qui écoute sur le port 80, va regarder si cette requête finit en `.test`. Si c'est le cas on continue, sinon Traefik ignore
- La requete `whoami.test` passe dans Traefik. Traefik regarde si il a un service labelisé `whoami.test`. Si c'est le cas, il retrouve sa configuration de routing et renvoie la requête vers l'adresse ip du conteneur.
- Le conteneur nous répond et on récupère le résultat dans le navigateur

Une fois lancé et cette configuration faite, vous n'aurez plus besoin d'y toucher.

Have fun.
