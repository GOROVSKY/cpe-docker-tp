
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
# Le jdk permet de compiler le java
FROM  maven:3.6.3-jdk-11  AS  myapp-build

#On se place dans un dossier
ENV  MYAPP_HOME  /opt/myapp
WORKDIR  $MYAPP_HOME

# On copie le fichier de conf pour build
COPY  pom.xml  .

# On copie tous les fichiers du projet à build
COPY  src  ./src

# On lance le build via maven
RUN  mvn  package  -DskipTests  dependency:go-offline

# Run
# On part d'une image plus simple pour avoir une empreinte mémoire plus légère
FROM  openjdk:11-jre
ENV  MYAPP_HOME  /opt/myapp
WORKDIR  $MYAPP_HOME

# On construie notre .jar via les fichiers qu'on vient de build au stage précédent
COPY  --from=myapp-build  $MYAPP_HOME/target/*.jar  $MYAPP_HOME/myapp.jar

# Les commandes à exécuter au run
ENTRYPOINT  java  -jar  myapp.jar
```

* On build notre projet : `docker build -t guillaumelaville/simple-api .`
* On run le docker en précisant un port (8090) pour accéder à l'API : `docker run -p 8090:8080 --name simple-api guillaumelaville/simple-api`
* On se rend sur http://localhost:8090/?name=Guillaume et l'API nous renvoie "hello, Guillaume!" 

#### Backend api
* Je récupère le projet simple-api sur le git, je copie le Dockerfile du projet précédent. Je modifie le fichier application.yml :
	* url: databaseapp *(le nom de mon container bd)*
	* username: usr
	* password: pwd 
* Je run mon container : `docker run --name backend-api -p 8080:8080 --net app-network guillaumelaville/backend-api`
* Je me rends sur `http://localhost:8080/departments/IRC/students` pour constater sur mon api backend lancée interroge bien ma BD et affiche le résultat

### HTTP Server
#### Basic
* Création d'un Dockerfile
```Dockerfile
# Image de base httpd
FROM  httpd:2.4

# On copie nos fichiers dans le container
COPY  .  /usr/local/apache2/htdocs/
```
* Après création d'un index.html, build du container : `docker build -t guillaumelaville/http-server .`
* Lancement du container : `docker run --name http-server -p 8090:80 --network app-network guillaumelaville/http-server`
* Mon index.html s'affiche bien sur `localhost:8090`


#### Commandes
`docker stats {id}` : des indicateurs de consommation mémoire, CPU et autres
`docker inspect {id}` : toutes les infos du container, les ports mobilisés, les connections tcp autorisées 
`docker logs {id}` : les logs affichés par le container (visibles si pas -d)


#### Configuration
* J'utilise *docker exec* pour exécuter une commande dans mon container existant (nommé http-server) : `docker exec -it http-server cat /usr/local/apache2/conf/httpd.conf`
* La configuration d'apache2 s'affiche dans le terminal car on se trouve dans le container
* On peut copier cette configuration dans notre dossier local après avoir créé le fichier httpd.conf : `docker cp id_container:/usr/local/apache2/conf/httpd.conf httpd.conf`

#### Reverse proxy
* Après avoir récupéré la configuration et l'avoir modifiée, je modifie le Dockerfile pour copier cette nouvelle configuration dans le container :
```Dockerfile
# Image de base httpd
FROM  httpd:2.4

# On copie nos fichiers dans le container
COPY  index.html  /usr/local/apache2/htdocs/
  
# On rajoute notre configuration perso dans le container
COPY  httpd.conf  /usr/local/apache2/conf/httpd.conf
```
* On a besoin d'un reverse proxy pour éviter que les clients interrogent directement notre API backend, cela sécurise en entrée notre application.

**Penser à mettre le container proxy dans le même network ...** 

#### Docker-compose
* Docker-compose est un manager au-dessus de docker, on lui fourni des images à compiler
```Docker-compose
version: '3.3'  # La version de notre Docker compose
services: # Déclaration des services à builddpc
	backend-api:
		build: ./backend-api  # Le dossier où se situe notre Dockerfile à build
		networks: # On assigne notre container à un network
			- app-network
		depends_on: # On précise que notre service dépend d'un autre
			- database
	database:
		build: ./database-app
		networks:
			- app-network
	httpd:
		build: ./http-server
		ports: # La conversion de port
			- "8080:80"
		networks:
			- app-network
		depends_on:
			- backend-api
networks: # Déclaration des networks
	app-network:
```

* Lancer un docker-compose : `docker-compose up`
* Les ports du backend et de la database ne sont pas ouverts

#### Publish
* Je me connecte à docker hub : `docker login`
* Je tag mon image de database : `docker tag database guillaumelaville/database:1.0`
* Je mets en ligne mon image : `docker push guillaumelaville/database`
* Je me connecte sur mon profil docker hub et je constate bien que mon image a été publiée : grâce à cela j'aurais accès à mon image sur différents PC et créer des containers dans différents environnements



## TP 2
### Setup Github Actions
* *Testcontainers* est une classe Java qui permet de réaliser des test unitaires et autres en instanciant par exemple des bases de données conteneurisées.
* Build et tester une application : `mvn clean verify` à l'endroit où se trouve le fichier pom.xml qui contient les dépendances et les types de tests réalisables
```yml
name: CI devops 2022 CPE
on:
	push: # le déclencheur
		branches: # les branches concernées
			- main
			- develop
	pull_request:
jobs:
	test-backend: # le nom du job
		runs-on: ubuntu-18.04 # on spécifie l'OS
		steps: # les étapes du job
			- uses: actions/checkout@v2.3.3 # on fait un using pour spécifier comme chargé
			- name: Set up JDK 11
				uses: actions/setup-java@v2 
				with: # on met en place une jdk 11
					java-version: '11'
					distribution: 'adopt'
			- name: Build and test with Maven
				working-directory: ./tp/backend-api # on se place dans le dossier de l'appli
				run: mvn clean verify # on build et test notre aplication 
```
* Ajout des variables secrètes dans github actions pour ne pas faire trainer les login/mot de passe en clair dans les commit : 
	* Settings -> Secrets -> Actions : **DOCKERHUB_USERNAME**  
	* J'enregistre aussi mon token généré sur Dockerhub : `41f14573-b06a-4af3-9b3d-b899e0d844e0`

* On build nos images grâce à la pipeline : 
	* On écrit le nouveau job `needs: build-and-test-backend`
	* On rajoute une étape de login à Docker hub puis les étapes de push des images avec en tag les credentials Dockerhub (enregistrés en secret sur github)
	* Au moment de commit, la pipeline est lancée 
		* D'abbord le job test-backend qui réussi
		* Puis build and push docker images : quand terminé, mes images sont publiées sur mon Dockerhub !

#### Setup Quality Gate
 * Je me conncte sur SonerCloud via mon compte GitHub et je choisis mon projet à analyser
	 *	Project key : `GOROVSKY_cpe-docker-tp`
	 *	Sonar token : `0e6bf54ac126083a02ad89df8325fe220964e53b`

```yml
- name: Build and test with Maven
	env:
		GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
		SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
	working-directory: ./tp/backend-api
	run: mvn -B verify sonar:sonar -Dsonar.projectKey=GOROVSKY_cpe-docker-tp
		-Dsonar.organization=gorovsky
		-Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./pom.xml
```

## TP 3
### Intro
* Création du chemin ansible/inventories
* Ajout du fichier setup.yml :
 ```yml
 all:
	vars:
		ansible_user: centos # On indique l'utilisateur qui souhaite établir une connexion avec les serveurs
		ansible_ssh_private_key_file: ../id_rsa # On indique notre clef rsa
	children:
		prod:
			hosts: guillaume.laville.takima.cloud # On spécifie l'adresse de notre serveur en pro
 ```
 * Test de ping : `ansible all -i inventories/setup.yml -m ping`

 ### Facts
* Les facts sont des informations déduites par ansible lors de la communication avec le server : `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"`
	* ansible_distribution : CentOS
	* ansible_distribution_version : 8 
* On peut supprimer httpd en indiquant l'état du server (avec httpd de supprimé) : `ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become`

