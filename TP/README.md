# Commandes

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

## 2. Back du backendApi

```powershell
cd TP/backendAPI
javac Main.java # si modif dans main
docker build -t emmalehec/backendapi .
docker run --rm emmalehec/backendapi
```

## 2. Back du simpleapi

```powershell
cd TP/simpleapi
docker build -t emmalehec/simple-api .
docker run -d --name simple-api --network app-network -p 8080:8080 emmalehec/simple-api
```

## 3. HTTP
```powershell
cd TP/httpd
docker build -t emmalehec/httpd .
docker run -d --name httpd --network app-network -p 80:80 emmalehec/httpd
```

## 4. Link
```powershell
cd TP
docker compose up -d --build
```