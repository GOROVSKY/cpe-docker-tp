
# TP docker Guillaume LAVILLE

  

## TP 1

  

### Database

  

* Création d'un Dockerfile, utilisation de l'image postgres:11.6-alpin.

* Build de l'image : `docker build -t guillaumelaville/databaseapp .`
* Démarrage du container : `docker run --name databaseapp guillaumelaville/databaseapp`

On récupère un client SQL qu'on démarre dans le même network que notre databaseapp :

* `docker pull adminer`
* Création d'un network : `docker network create app-network`
* Databaseapp : `docker run --name databaseapp -e POSTGRES_USER='usr' -e POSTGRES_PASSWORD='pwd' -e POSTGRES_DB='db' --network app-network guillaumelaville/databaseapp`
* Adminer : `docker run -p 8080:8080 --network app-network adminer`

*Le paramètre -e permet de passer des variables d'environnement pour éviter d'écrire les identifiants à la BD dans le Dockerfile mais cela complexifie la ligne de commande*

* Persistence des données en créant un volume : `-v /data:/var/lib/postgresql/data`
* Run le docker databaseapp : `docker run --name databaseapp -v /data:/var/lib/postgresql/data --network=app-network guillaumelaville/databaseapp`

**Pour se connecter à notre BD depuis adminer :**
* Server : **databaseapp**
* User ... (les info dans le Dockerfile)	

*Après modification/suppression des données, les changements sont conservés grâce au volume créé. Le volume permet de conserver les données attachées au docker pour éviter qu'elles soient supprimées quand on supprime le container* 

**Dockerfile**
```Dockerfile
# Image de base
FROM postgres:11.6-alpine

# Ajout des scripts sql à exucter au run
ADD 01-CreateScheme.sql /docker-entrypoint-initdb.d/01-CreateScheme.sql
ADD 02-InsertData.sql /docker-entrypoint-initdb.d/02-InsertData.sql

# Variables d'environnement
ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
``` 
 
 ### Backend API
 #### Basic
* Création d'un fichier Main.java basique
* Création du Dockerfile
```Dockerfile
# Choose a java JRE
FROM openjdk:11

# Add the compiled java (aka bytecode, aka .class)
COPY . /usr/src/backend-api
WORKDIR /usr/src/backend-api
  
# Run the Java with: “java Main” command.
RUN javac Main.java
CMD ["java", "Main"] 
```
On exécute `docker run --name backend-api backend-api` et `Hello world` s'affiche à l'écran.

#### Multistage builds
* Création d'un fichier Controlleur java

**Dockerfile**
```Dockerfile
# Build
# On nomme notre jdk via AS myapp-build qu'on pourra réutiliser ensuite pour le run
FROM maven:3.6.3-jdk-11 AS myapp-build
# On se place dans un dossier
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME

# On copie le fichier de conf pour build
COPY pom.xml .

# On copie tous les fichiers du projet à build
COPY src ./src

# On lance le build via maven
RUN mvn package -DskipTests dependency:go-offline

# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME

# On copie notre artefcat buildé précedemment  
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# Les commandes à exécuter au run
ENTRYPOINT java -jar myapp.jar
```

* On build notre projet : `docker build -t guillaumelaville/simple-api .`
* On run le docker en précisant un port (8090) pour accéder à l'API : `docker run -p 8090:8080 --name simple-api guillaumelaville/simple-api`
* On se rend sur http://localhost:8090/?name=Guillaume et l'API nous renvoie "hello, Guillaume!" 

#### Backend api
* Je récupère le projet simple-api sur le git, je copie le Dockerfile du projet précédent. Je modifie le fichier application.yml :
	* url: databaseapp *(le nom de mon container bd)*
	* username: usr
	* password: pwd 
* Je run mon container : `docker run --name backend-api -p 8095:8080 --net app-network guillaumelaville/backend-api`

### HTTP Server


