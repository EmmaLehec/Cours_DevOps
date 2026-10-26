# Lancer

## 0. Réseau (une seule fois)

```powershell
docker network create app-network
```

## 1. Database

```powershell
cd TP/database

docker build -t emmalehec/tpdevops .

docker run -d --name tpdevops --network app-network -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd -v pgdata:/var/lib/postgresql/data emmalehec/tpdevops

docker run -d -p 8090:8080 --network app-network --name adminer adminer
```

http://localhost:8090
- Système : PostgreSQL
- Serveur : `tpdevops`
- Utilisateur : `usr`
- Mot de passe : `pwd`
- Base : `db`

## 2. Back du backendApi (basic) 

```powershell
cd TP/backendAPI
javac Main.java # si modif dans main
docker build -t emmalehec/backendapi .
docker run --rm emmalehec/backendapi
```

## 2. Back du simpleapi (mutlistage build)

```powershell
cd TP/simpleapi
docker build -t emmalehec/simple-api .
docker run -d --name simple-api --network app-network -p 8080:8080 -e SPRING_DATASOURCE_URL=jdbc:postgresql://tpdevops:5432/db -e SPRING_DATASOURCE_USERNAME=usr -e SPRING_DATASOURCE_PASSWORD=pwd emmalehec/simple-api
```

## 3. HTTP
```powershell
cd TP/httpd
docker build -t emmalehec/httpd .
docker run -d --name httpd --network app-network -p 80:80 -e BACKEND_HOST=simple-api -e BACKEND_PORT=8080  emmalehec/httpd
```

## 4. Link
```powershell
cd TP
docker compose up -d --build
```

# Rapport TP1

## 1-1 For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?

On ajoute le flag -e quand on run dans le container afin que les variables d'environnement ne soient pas inscrites dans le Dockerfile qui est push et donc partagé, mais simplement donné qu'au lancement.

## 1-2 Why do we need a volume to be attached to our postgres container?

On attache un volume afin de faire persister les données. Cela permet que les données modifiées dans la base soient conservées lorsque le container est stoppé (compose down) puis crée à nouveau (compose up). Elles sont conservées hors du container.

## 1-3 Document your database container essentials: commands and Dockerfile.

Les commandes :
- docker network create app-network
-> permet de créer le réseau Docker app-network

- docker build -t emmalehec/tpdevops .
-> permet de créer l'image du container à partir du dossier courant (.) avec le nom emmalehec/tpdevops (-t)

- docker run -d --name tpdevops --network app-network -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd -v pgdata:/var/lib/postgresql/data emmalehec/tpdevops
-> permet de créer le container tpdevops (--name) à partir de l'image emmalehec/tpdevops,
en mode detached (-d),
dans le réseau Docker app-network,
en passant les variables d'environnement (-e) (dont le nom de la bdd, le nom du user, le mot de passe),
et en attachant un volume afin que les données soient persistentes (-v)

- docker run -d -p 8090:8080 --network app-network --name adminer adminer
-> permet de créer le container adminer (--name) à partir de l'image adminer,
en mode detached (-d),
en reliant le port 8090 de la machine au port 8080 du container adminer,
dans le réseau Docker app-network

Les dockerfiles :
FROM postgres:17.2-alpine
-> image
COPY sql/ /docker-entrypoint-initdb.d/
-> on copie le contenu du dossier sql dans le dossier docker-entrypoint-initdb.d du docker. Ainsi, les fichiers CreateScheme.sql et InsertData.sql seront lancés automatiquement au run du container afin de créer et remplir les tables de la base.

## 1-4 Why do we need a multistage build? And explain each step of this dockerfile.

On utilise un multistage build pour compiler dans une première image (où on utilise le jdk qui est lourd), puis exécuter uniquement le .jar dans une deuxième image (où on utilise le jre qui est plus léger).

# Etape Build
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
-> image de build avec jdk 21

ENV MYAPP_HOME=/opt/myapp
-> crée variable MYAPP_HOME

WORKDIR $MYAPP_HOME
-> définit comme dossier de travail

RUN apk add --no-cache maven
-> installe maven

COPY pom.xml .
-> copie la liste de dépendances

RUN mvn dependency:go-offline
-> maven télécharge liste de dépendances

COPY src ./src
-> copie le code

RUN mvn package -DskipTests
-> compile (créer le .jar)

# Etape Run
FROM eclipse-temurin:21-jre-alpine
-> image d'exécution avec le jre 21

ENV MYAPP_HOME=/opt/myapp
-> crée variable MYAPP_HOME

WORKDIR $MYAPP_HOME
-> définit comme dossier de travail

COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
-> copie uniquement .jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
-> lance l'application

## 1-5 Why do we need a reverse proxy?

Il permet d'avoir un seul point d'entrée. Le backend et la database ne sont pas accessibles et sont cachés. On peut aussi répartir la charge en lançant plusieurs backend identiques et donc en répartissant les requêtes entre eux.

## 1-6 Why is docker-compose so important?

Lorsque qu'on lance le docker compose up, le fichier docker-compose.yml est lu. Ce fichier décrit toute l'application (services, réseau, volumes). Il permet de tout lancer en une seule fois, et d'éviter de lancer toutes les autres commandes.

## 1-7 Document docker-compose most important commands.

docker compose up -d --build 
-> build et run (en detached -d)

docker compose ps
-> liste des containers avec leur état

docker compose logs backend
-> liste des logs d'un service (ici backend)

docker compose stop
-> arrête le container sans le supprimer

docker compose down
-> arrête et supprime le container

docker compose down -v  
-> arrête et supprime le container et supprime aussi les volumes

## 1-8 Document your docker-compose file.

services:
    backend:
        build: ./simpleapi
        -> build avec l'image du dockerfile du dossier simpleapi

        environment:
            SPRING_DATASOURCE_URL: jdbc:postgresql://database:5432/${POSTGRES_DB}
            SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
            SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
            -> adresse de la base, identifiant et mot de passe de la base

        networks:
            - front-network
            - back-network
            -> réseau pour communiquer avec la database et celui avec le httpd

        depends_on:
            database:
                condition: service_healthy
                -> démarre après que le container de la databse soit lancé et prêt (healthy)

        restart: unless-stopped
        -> redémarré automatiquement tout le tps (en cas de crash) sauf si volontaire

    database:
        build: ./database
        -> build avec l'image du dockerfile du dossier database

        environment:
            POSTGRES_DB: ${POSTGRES_DB}
            POSTGRES_USER: ${POSTGRES_USER}
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
        -> nom de la base, identifiant et mot de passe de la base

        volumes:
            - pgdata:/var/lib/postgresql/data
            -> volume pour données persistentes
        
        networks:
            - back-network
            -> réseau pour communiquer avec le backend

        healthcheck:
            test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
            interval: 5s
            timeout: 5s
            retries: 10
        -> lance commande pour vérifier la base (ok si connection) toutes les 5s, avec 5s max sinon échec, et 10 échecs max sinon unhealthy

        restart: unless-stopped
        -> redémarré automatiquement tout le tps (en cas de crash) sauf si volontaire

    httpd:
        build: ./httpd
        -> build avec l'image du dockerfile du dossier httpd
        
        environment:
            BACKEND_HOST: ${BACKEND_HOST}
            BACKEND_PORT: ${BACKEND_PORT}

        ports:
            - "80:80"
            -> port 80 de la machine connecté au port 80 du conteneur

        networks:
            - front-network
            -> réseau pour communiquer avec le backend

        depends_on:
            - backend
            -> démarre après le backend

        restart: unless-stopped
        -> redémarré automatiquement tout le tps (en cas de crash) sauf si volontaire

networks:
    front-network:
    -> déclare le réseau commun où les containers backend et httpd communiquent

    back-network:
    -> déclare le réseau commun où les containers backend et database communiquent

volumes:
    pgdata:
    -> déclare le volume pour la base

## 1-9 Document your publication commands and published images in dockerhub.

docker login
-> se connecter à Docker Hub

docker tag tp-database emmalehec/tp-database:1.0
docker tag tp-backend emmalehec/tp-backend:1.0
docker tag tp-httpd emmalehec/tp-httpd:1.0
-> donne des noms complets (nom du dépôt + version) aux images du Docker

docker push emmalehec/tp-database:1.0
docker push emmalehec/tp-backend:1.0
docker push emmalehec/tp-httpd:1.0
-> publication des images sur Docker Hub

docker pull emmalehec/tp-backend:1.0
-> récupérer une image à partir du Docker Hub

## 1-10 Why do we put our images into an online repo?

On publie une image en ligne afin qu'elle soit partagée et utilisable par d'autres personnes. 
Cela permet aussi de la déployer après avoir été récupérée par un serveur de production, ce qui peut aussi être automatisé (CI/CD). 
On peut également suivre les versions des images.
Et l'image est aussi sauvegardé ailleurs que sur la machine. 

# Rapport TP2

## 2-1 What are testcontainers?

Les testcontainers sont une librairie Java qui permettent de lancer des containers Docker temporaires lors des tests. Un container PostgreSQL démarre automatiquement lors de "mvn clean verify" pour vérifier que les données sont bien insérées lors du test.

## 2-2 For what purpose do we need to use secured variables ?

Pour se connecter à Docker Hub la pipeline a besoin de l'identifiant et du token. On ne mets pas ces deux informations sur le main.yml puisqu'il sera push et elles seraient donc visibles par tout le monde. N'importe qui pourrait push et supprimer mes images sur mon compte Docker Hub.

## 2-3 Why did we put needs: build-and-test-backend on this job? Maybe try without this and you will see!

On ajoute la section needs, car sinon les jobs des test s'exécutent en parallèle. Needs permet de s'assurer que le job de test-backend est terminé avant de lancer le job de build-and-push-docker-image.

## 2-4 For what purpose do we need to push docker images?

On publie les images sur Docker Hub afin qu'elles soient sauvegardé en dehors de la pipeline, puisque GitHub détruit les images après les avoir créées.
Elles peuvent être ensuite récupérées (docker pull) par un quelqu'un de l'équipe ou par le serveur de production afin d'être déployée.
A chaque fois qu'on push sur main sur GitHub, nos images seront publiées automatiquement sur le Docker Hub, c'est le Continuous Delivery (CD). 

# Rapport TP3

## 3-1 Document your inventory and base commands

L'inventaire (setup.yml) décrit les serveurs de Ansible et leurs paramètres de connexion.

all:
  vars: -> variables communes à tous les serveurs
    ansible_user: admin
    -> utilisateur SSH

    ansible_ssh_private_key_file: /home/emma/.ssh/id_rsa
    -> chemin de la clé privée SSH

  children:
    prod:
    -> groupe prod pour regrouper pls serveurs par rôle

      hosts:
        emma.lehec.takima.school:

Commandes de base :

ansible all -i inventories/setup.yml -m ping
-> vérifie que serveur est joignable et authentification fonctionne

ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
-> récupère facts/infos découvertes sur le serveur, filtrées par des facts uniquement sur la distribution Linux

ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
-> on désinstalle Apache (état voulu : absent). Au 1er lancement, on a le résultat changed puisqu'Apache est désinstallé, et au 2e lancement, on a le résultat SUCCESS puisqu'Apache était déjà désinstallé, il ne fait donc rien.

