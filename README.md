
# TP docker Guillaume LAVILLE

  

## TP 1

  

### Database

  

* Création d'un Dockerfile, utilisation de l'image postgres:11.6-alpin.

* Build de l'image : `docker build -t guillaumelaville/databaseapp .`
* Démarrage du container : `docker run -p 5432:5432 --name databaseapp guillaumelaville/databaseapp`

On récupère un client SQL qu'on démarre dans le même network que notre databaseapp :

* `docker pull adminer`
* Création d'un network : `docker network create app-network`
* Databaseapp : `docker run -p 5432:5432 --name databaseapp --network=app-network guillaumelaville/databaseapp`
* Adminer : `docker run --link some_database:db -p 8080:8080 --network=app-network adminer`


  
