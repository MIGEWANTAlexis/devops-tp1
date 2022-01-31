# TP1 - Docker

# Database

## Basics

```
FROM postgres:11.6-alpine
ENV POSTGRES_DB=db \ 
		POSTGRES_USER=usr \ 
		POSTGRES_PASSWORD=pwd
```

`**Image**`

```bash
docker build -t alxs39/postgres-data-base .
```

**`-t`** : nommer une image avec la commande `**docker build**`

`**alxs39/postgres-data-base**`: nom de l’image en fonction du `**USERNAME**` sur Docker Hub

`**.**` : l’endroit où est le Dockerfile → dans le répertoire courant

### **Conteneurs**

`**Postgres**`

```bash
docker run -p 5432:5432 -e POSTGRES_DB=db -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=admin --network=app-network --name postgres-data-base -d alxs39/postgres-data-base
```

`**-e**` : pour indiquer des variables d’environnements → pour le nom de la base de données, le user et son mot de passe

<aside>
ℹ️ Why should we run the container with a flag -e to give the environment variables ?

</aside>

→ pour utiliser nos propres variables d’environnements et ne pas utiliser celle par défaut dans le Dockerfile. `**C’est mieux de passer avec -e pour sécuriser**`

→ Alternative : créer un fichier .env et ajouter la clause `**—env-file .env (répertoire du .env)**`

```bash
docker run -p 5432:5432 --env-file .env --network=app-network --name postgres-data-base -d alxs39/postgres-data-base
```

`**--network**` : pour lier le conteneur à un réseau → pour moi `**app-network**` partagé avec adminer

`**--name**` : pour nommer le conteneur → pour moi `**postgres-data-base**`

`**-d**` : mode détaché → pour avoir la main sur le terminal et lancer le conteneur tout de même

**`-p`** : exposer les ports → pour moi `**5432**` (host) sur `**5432**` (conteneur)

`**Adminer**`

```bash
docker pull adminer:4.8.1
```

```bash
docker run --name=adminer --network=app-network -p 8080:8080 -d adminer:4.8.1
```

Pour se connecter depuis l’interface http.//localhost:8080 :

```bash
Système	        PostgreSQL
Serveur	        postgres-data-base
Utilisateur	    admin
Mot de passe	  admin
Base de données	db
```

## Init database

Ajouter la ligne dans le Dockerfile et créer les scripts dans le dossier le travail

```bash
COPY ["./01-CreateScheme.sql", "./02-InsertData.sql", "/docker-entrypoint-initdb.d/"]
```

```
FROM postgres:11.6-alpine
COPY ["./01-CreateScheme.sql", "./02-InsertData.sql", "/docker-entrypoint-initdb.d/"]

ENV POSTGRES_DB=db \ 
    POSTGRES_USER=usr \ 
    POSTGRES_PASSWORD=pwd
```

```sql
.
└── Database
    ├── 01-CreateScheme.sql
    ├── 02-InsertData.sql
    ├── Dockerfile
    └── postgres-data
```

```sql
CREATE TABLE public.departments
(
    id      SERIAL      PRIMARY KEY,
    name    VARCHAR(20) NOT NULL
);

CREATE TABLE public.students
(
    id              SERIAL      PRIMARY KEY,
    department_id   INT         NOT NULL REFERENCES departments (id),
    first_name      VARCHAR(20) NOT NULL,
    last_name       VARCHAR(20) NOT NULL
);
```

```sql
INSERT INTO departments (name) VALUES ('IRC');
INSERT INTO departments (name) VALUES ('ETI');
INSERT INTO departments (name) VALUES ('CGP');
INSERT INTO students (department_id, first_name, last_name) VALUES (1, 'Eli', 'Copter');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Emma', 'Carena');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Jack', 'Uzzi');
INSERT INTO students (department_id, first_name, last_name) VALUES (3, 'Aude', 'Javel');
```

## Persist data

Ajouter la ligne lors de la commande docker run :

```bash
-v '/Users/alexis/Documents/IRC/4IRC/S8/Devops/Séance 1/TP1/Database/postgres-data':/var/lib/postgresql/data
```

```bash
docker run -p 5432:5432 --env-file .env --network=app-network -v '/Users/alexis/Documents/IRC/4IRC/S8/Devops/Séance 1/TP1/Database/postgres-data':/var/lib/postgresql/data --name postgres-data-base -d alxs39/postgres-data-base
```

**`-v`** : pour indiquer un répertoire pour stocker les données du conteneurs et les rendre persistantes → pour moi le chemin absolu de mon répertoire est `**/Users/alexis/Documents/IRC/4IRC/S8/Devops/Séance 1/TP1/Database/postgres-data**`

<aside>
ℹ️ Why do we need a volume to be attached to our postgres container ?

</aside>

→ Pour rendre les données persistantes et ne pas perdre les données lors de l’extinction du conteneur.

# **Backend API**

## Basics

```
├── Backend
│   ├── hello
│   │   ├── Dockerfile
│   │   └── Main.java
```

`**Java**`

```java
public class Main {
    public static void main(String[] args) { 
        System.out.println("Hello World!");
    }
}
```

`**Dockerfile**`

```
FROM openjdk:11
WORKDIR /app
COPY ./Main.java .
RUN ["javac", "Main.java"]
CMD ["java", "Main"]
```

`**Commande**`

```
docker run --name=java-hello alxs39/java-hello
$ Hello World!
```

## Multistage build

```
├── Backend
│   ├── hello
│   │   ├── Dockerfile
│   │   └── Main.java
│   └── simple-api
│       ├── Dockerfile
│       ├── HELP.md
│       ├── mvnw
│       ├── mvnw.cmd
│       ├── pom.xml
│       ├── src
│       └── target
```

<aside>
ℹ️ Why do we need a multistage build ? And explain each steps of this dockerfile

</aside>

→ Il faut deux étapes, une première étape de build pour compilé le projet maven et une deuxième étape qui exécute l’application java. De plus nous ne voulons pas des restes du conteneurs de build d’où l’utilité de faire un build et un run.

`**Dockerfile**`

```
# Build
# On définit l'image de base maven jdk 11 et on fait un alias myapp-build
FROM maven:3.6.3-jdk-11 AS myapp-build

# On définit la variable d'environnement du répertoire de travail avec un alias MYAPP_HOME
ENV MYAPP_HOME /opt/myapp

# On définit l'espace de travail dans MYAPP_HOME donc dans /opt/myapp 
WORKDIR $MYAPP_HOME

# On copie le fichier pom.xml dans le répertoire courant donc /opt/myapp 
COPY pom.xml .

# On copie le dossier src dans le répertoire courant donc /opt/myapp
COPY src ./src

# On compile et empaquete le code
RUN mvn package -DskipTests

# Run
# On définit l'image de base jdk 11
FROM openjdk:11-jre

# On définit la variable d'environnement du répertoire de travail avec un alias MYAPP_HOME
ENV MYAPP_HOME /opt/myapp

# On définit l'espace de travail dans MYAPP_HOME donc dans /opt/myapp 
WORKDIR $MYAPP_HOME

# On copie l'application compilé avec les .jar
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# La première commande qui sera lancé au start du conteneur : on lance l'app java
ENTRYPOINT java -jar myapp.jar
```

`**Commandes**`

```bash
docker build -t alxs39/simple-api .
docker run --name=simple-api -p 8080:8080 -d alxs39/simple-api
```

## Backend API

```
.
├── api
│   └── simple-api
│       ├── Dockerfile
│       ├── README.md
│       └── simple-api
│           ├── HELP.md
│           ├── pom.xml
│           └── src
```

Dans le fichier `**controller/DepartmentController.java**`

```java
import org.springframework.web.bind.annotation.*;
```

`**Dockerfile**`

```
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build 
ENV MYAPP_HOME /opt/myapp 
WORKDIR $MYAPP_HOME
COPY ./simple-api/pom.xml .
RUN mvn dependency:go-offline
COPY ./simple-api/src ./src
RUN mvn package -DskipTests
# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```

`**Commandes**`

```bash
docker build -t alxs39/api .
docker run --name=api --network=app-network -p 8080:8080 -d alxs39/api
```

# Http server

## Basics

```
.
├── Http-server
│   ├── Dockerfile
│   ├── index.html
```

`**Dockerfile**`

```
FROM httpd:2.4.52
WORKDIR /usr/local/apache2/htdocs/
COPY index.html .
EXPOSE 80
CMD ["httpd-foreground"]
```

`**Commandes**`

```bash
docker build -t alxs39/http-server .
docker run --name=http-server -p 80:80 -d alxs39/http-server
```

## Configuration

```bash
docker cp http-server:/usr/local/apache2/conf/httpd.conf .
```

## Reverse proxy