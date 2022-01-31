# Build
#On nomme notre jdk via AS myapp-build qu'on pourra réutiliser ensuite pour le run
FROM maven:3.6.3-jdk-11 AS myapp-build

#On se place dans un dossier
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

# On construie notre .jar via les fichiers qu'on vient de build au stage précédent
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# Les commandes à exécuter au run
ENTRYPOINT java -jar myapp.jar