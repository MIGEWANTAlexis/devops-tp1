# TP1 - Docker

# Database

## Basics

```
FROM postgres:11.6-alpine
ENV POSTGRES_DB=db \ 
    POSTGRES_USER=usr \ 
    POSTGRES_PASSWORD=pwd
```

---

**`Image`**

```bash
docker build -t alxs39/postgres-data-base .
```

**`-t`** : nommer une image avec la commande **`docker build`**

**`alxs39/postgres-data-base`**: nom de l’image en fonction du **`USERNAME`** sur Docker Hub

**`.`** : l’endroit où est le Dockerfile → dans le répertoire courant

---

### **Conteneurs**

**`Postgres`**

```bash
docker run -p 5432:5432 -e POSTGRES_DB=db -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=admin --network=app-network --name postgres-data-base -d alxs39/postgres-data-base
```

**`-e`** : pour indiquer des variables d’environnements → pour le nom de la base de données, le user et son mot de passe

<aside>
ℹ️ Why should we run the container with a flag -e to give the environment variables ?

</aside>

→ pour utiliser nos propres variables d’environnements et ne pas utiliser celle par défaut dans le Dockerfile. **`C’est mieux de passer avec -e pour sécuriser`**

→ Alternative : créer un fichier .env et ajouter la clause **`—env-file .env (répertoire du .env)`**

```bash
docker run -p 5432:5432 --env-file .env --network=app-network --name postgres-data-base -d alxs39/postgres-data-base
```

**`--network`** : pour lier le conteneur à un réseau → pour moi **`app-network`** partagé avec adminer

**`--name`** : pour nommer le conteneur → pour moi **`postgres-data-base`**

**`-d`** : mode détaché → pour avoir la main sur le terminal et lancer le conteneur tout de même

**`-p`** : exposer les ports → pour moi **`5432`** (host) sur **`5432`** (conteneur)

---

**`Adminer`**

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

---

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

---

## Persist data

Ajouter la ligne lors de la commande docker run :

```bash
-v '/Users/alexis/Documents/IRC/4IRC/S8/Devops/Séance 1/TP1/Database/postgres-data':/var/lib/postgresql/data
```

```bash
docker run -p 5432:5432 --env-file .env --network=app-network -v '/Users/alexis/Documents/IRC/4IRC/S8/Devops/Séance 1/TP1/Database/postgres-data':/var/lib/postgresql/data --name postgres-data-base -d alxs39/postgres-data-base
```

**`-v`** : pour indiquer un répertoire pour stocker les données du conteneurs et les rendre persistantes → pour moi le chemin absolu de mon répertoire est **`/Users/alexis/Documents/IRC/4IRC/S8/Devops/Séance 1/TP1/Database/postgres-data`**

<aside>
ℹ️ Why do we need a volume to be attached to our postgres container ?

</aside>

→ Pour rendre les données persistantes et ne pas perdre les données lors de l’extinction du conteneur.

---

# **Backend API**

## Basics

```
├── Backend
│   ├── hello
│   │   ├── Dockerfile
│   │   └── Main.java
```

**`Java`**

```java
public class Main {
    public static void main(String[] args) { 
        System.out.println("Hello World!");
    }
}
```

**`Dockerfile`**

```
FROM openjdk:11
WORKDIR /app
COPY ./Main.java .
RUN ["javac", "Main.java"]
CMD ["java", "Main"]
```

**`Commande`**

```
docker run --name=java-hello alxs39/java-hello
$ Hello World!
```

---

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

→ Il faut deux étapes, une première étape de build pour compilé le projet maven (avec les **`jdk`** → outil de développement java qui sont relativement lourd) et une deuxième étape qui exécute l’application java (avec les **`jre`** qui sont moins lourd et n’exécute que l’application java). De plus nous ne voulons pas des restes du conteneurs de build d’où l’utilité de faire un build et un run.

**`Dockerfile`**

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

**`Commandes`**

```bash
docker build -t alxs39/simple-api .
docker run --name=simple-api -p 8080:8080 -d alxs39/simple-api
```

---

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

Dans le fichier **`controller/DepartmentController.java`**

```java
// corrigé sur le github de la prof
import org.springframework.web.bind.annotation.*;
```

Dans le fichier **`ressources/application.yml`**

```yaml
datasource:
    url: jdbc:postgresql://postgres-data-base:5432/db
    username: admin
    password: admin
    driver-class-name: org.postgresql.Driver
```

**`Dockerfile`**

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

**`Commandes`**

```bash
docker build -t alxs39/api .
docker run --name=api --network=app-network -p 8080:8080 -d alxs39/api
```

---

# Http server

## Basics

```
.
├── Http-server
│   ├── Dockerfile
│   ├── index.html
```

**`Dockerfile`**

```
FROM httpd:2.4.52
WORKDIR /usr/local/apache2/htdocs/
COPY index.html .
EXPOSE 80
CMD ["httpd-foreground"]
```

**`Commandes`**

```bash
docker build -t alxs39/http-server .
docker run --name=http-server --network=app-network -p 80:80 -d alxs39/http-server
```

---

## Configuration

```bash
docker cp http-server:/usr/local/apache2/conf/httpd.conf .
```

---

## Reverse proxy

**`httpd.conf`**

```
# ligne à décommenter
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

```
# en haut du fichier
ServerName localhost
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://api:8080/ 
    ProxyPassReverse / http://api:8080/
</VirtualHost>
ServerRoot "/usr/local/apache2"
```

**`Dockerfile`**

```
FROM httpd:2.4.52
WORKDIR /usr/local/apache2/htdocs/
COPY index.html .
WORKDIR /usr/local/apache2/conf/
COPY httpd.conf .
EXPOSE 80
CMD ["httpd-foreground"]
```

<aside>
ℹ️ Why do we need a reverse proxy ?

</aside>

→ Httpd ne génère pas lui-même les données, mais le contenu est obtenu par un ou plusieurs serveurs dorsaux, qui n'ont normalement pas de connexion directe avec le réseau externe. Lorsque httpd reçoit une demande d'un client, la demande elle-même est transmise par procuration à l'un de ces serveurs dorsaux, qui traite ensuite la demande, génère le contenu, puis renvoie ce contenu à httpd, qui génère ensuite la réponse HTTP réelle au client. 

→ Pour des raisons liées à la sécurité, à la haute disponibilité, à l'équilibrage des charges et à l'authentification/autorisation centralisée. Donc le serveur reverse proxy est la seule source de tout le contenu (point d’entrée unique).

---

# Link application

```bash
.
├── Backend
│   ├── api
│   ├── hello
│   └── simple-api
├── Database
│   ├── 01-CreateScheme.sql
│   ├── 02-InsertData.sql
│   ├── Dockerfile
│   └── postgres-data
├── Http-server
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
├── README.md
├── TP1.pdf
└── docker-compose.yml
```

## Docker-compose

**`docker-compose.yml`**

```yaml
version: '3.7' 
services:
  backend: 
    build: ./Backend/api/simple-api/
    image: alxs39/api
    container_name: api
    networks:
      - app-network
    depends_on:
      - database
  database: 
    build: ./Database/
    image: alxs39/postgres-data-base
    env_file:
      - ./Database/.env
    container_name: postgres-data-base
    volumes:
      - /Users/alexis/Documents/IRC/4IRC/S8/Devops/Séance 1/TP1/Database/postgres-data:/var/lib/postgresql/data
    networks:
      - app-network
  httpd: 
    build: ./Http-server/
    image: alxs39/http-server
    container_name: http-server
    ports:
      - 80:80
    networks:
      - app-network
    depends_on:
      - backend
networks: 
  app-network:
    external: true
```

**`services`** : déclaration des services, regroupement de conteneur

**`env_file`** : pour indiquer le fichier de variables d’environnements au conteneur postgres

**`build`** : indiquer l’endroit du Dockerfile à utiliser

**`container_name`**: nom du conteneur dans le service

**`networks`** : réseau à utiliser par le conteneur

**`depends_on`** : dépendre d’un autre conteneur

**`ports`** : gestion de la redirection de port du conteneur sur le host

**`networks`** : déclaration des réseaux externes pour cet étape

**`volumes`** : indiquer le volume, soit dans les dossiers perso, soit un volume docker, ici c’est un dossier de ma machine

**`image`** : nommer l’image du build de l’image

**`Commandes`**

```bash
docker-compose -p 'tp1' up -d
```

**`-p`** : donner un nom au service, ici tp1

**`up`** : lancer le service, donc les conteneurs dans ce service

**`-d`** : mode détaché, pour nous laisser la main dans le terminal exécutant la commande

---

## Publish

**`Commandes`**

```bash
# tag des trois images (pour donner une version) : la base de données, l'api et le http serveur
docker tag alxs39/postgres-data-base alxs39/postgres-data-base:1.0
docker tag alxs39/api alxs39/api:1.0
docker tag alxs39/http-server alxs39/http-server:1.0
```

```bash
# Connexion au docker hub
docker login -u 'alxs39' -p 'password' docker.io
```

```bash
# push des trois images sur le hub
docker push alxs39/postgres-data-base:1.0
docker push alxs39/api:1.0
docker push alxs39/http-server:1.0
```

<aside>
ℹ️ Why do we put our images into an online repository ?

</aside>

→ Pour que d’autres développeurs puissent utiliser nos images custom. Ou sur un repo privé pour être utilisé par des collaborateurs d’une entreprise.

# TP2 - CI/CD

# **Setup Github Actions**

## First steps into the CI world

```
.
├── Backend
│   ├── api
│   ├── hello
│   └── simple-api
├── Database
│   ├── 01-CreateScheme.sql
│   ├── 02-InsertData.sql
│   ├── Dockerfile
│   └── postgres-data
├── Http-server
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
├── README.md
├── TP1.pdf
├── docker-compose.yml
└── .main.yml

```

```bash
create .main.yml at root project
```

## Build and test your application

```bash
mvn clean verify
```

```bash
# bug -> remplacé par.
# corrigé dans le github de la prof
mvn clean verify -DskipTests
```

<aside>
ℹ️ What are testcontainers?

</aside>

➡️ Testcontainers est une librairie qui permet de facilement démarrer un conteneur Docker au sein des tests unitaires. Il fournit une extension JUnit 5 et une extension permettant de facilement lancer des bases de données. Son utilisation est donc idéale pour lancer notre base de données Postgres pour nos tests unitaires.

<aside>
ℹ️ Document your Github Actions configurations

</aside>

```yaml
name: CI devops 2022 CPE 
on:
  #to begin you want to launch this job in main (master for me i don't rename it) and develop
  push:
    branches:
      - master
      - develop
  pull_request:
    branches:
      - master
      - develop
jobs: 
  test-backend:
    runs-on: ubuntu-18.04 
    steps:
      - uses: actions/checkout@v2.3.3 #checkout your github code using actions/checkout@v2.3.3
      - name: Set up JDK 11
        uses: actions/setup-java@v2 #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
        with:
          distribution: 'temurin'
          java-version: '11'
          cache: 'maven'
      - name: Build and test with Maven #finally build your app with the latest command
        run: mvn clean verify --file ./Backend/api/simple-api/simple-api/pom.xml
```

**`on`** : pour déclencher automatiquement la pipeline

**`push`** : pour déclencher lorsqu’un push sur une branche est détecté

**`pull_request`** : comme push, mais lors d’un pull request sur les branches listées. Pour moi master et develop

**`branches`** : pour déclencher lorsqu’un push est détecté sur les branches listées. Pour moi develop et master

**`jobs`** : travail à effectuer dans la pipeline 

**`test-backend`** : l’id du job

**`runs-on`** : définir le type de machine sur laquelle exécuter la tâche

**`steps`** : étapes de la pipeline (les tâches à réaliser)

**`uses`** : action à exécuter

**`name`** : nom l’étape à afficher sur GitHub.

**`with`** : carte des paramètres d'entrée définis par l'action

**`distribution`** : distribution pour java

**`java-version`** : la version de java

**`cache`** : pour mettre en cache les dépendances maven

**`run`** : exécuter des commande avec le shell du système d’exploitation

## First steps into the CD world

```yaml
name: CI devops 2022 CPE 
on:
  #to begin you want to launch this job in main (master for me i don't rename it) and develop
  push:
    branches:
      - master
      - develop
  pull_request:
    branches:
      - master
      - develop
jobs: 
  test-backend:
    runs-on: ubuntu-18.04 
    steps:
      - uses: actions/checkout@v2.3.3 #checkout your github code using actions/checkout@v2.3.3
      - name: Set up JDK 11
        uses: actions/setup-java@v2 #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
        with:
          distribution: 'temurin'
          java-version: '11'
          cache: 'maven'
      - name: Build and test with Maven #finally build your app with the latest command
        run: mvn clean verify --file ./Backend/api/simple-api/simple-api/pom.xml
  build-and-push-docker-image:
    needs: test-backend # run only when code is compiling and tests are passing 
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          context: ./Backend/api/simple-api/ # relative path to the place where source code with Dockerfile is located
          tags: ${{secrets.DOCKERHUB_USERNAME}}/api:1.0 # Note: tags has to be all lower-case 
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          context: ./Database/
          tags: ${{secrets.DOCKERHUB_USERNAME}}/postgres-data-base:1.0
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          context: ./Http-server/
          tags: ${{secrets.DOCKERHUB_USERNAME}}/http-server:1.0
```

<aside>
ℹ️ Why did we put **needs: build-and-test-backend** on this job?

</aside>

➡️ on doit avoir le backend de build et testé, avant de build et de push les images sur le hub.

### **Publish your docker images when there is a commit on the main branch**

```yaml
name: CI devops 2022 CPE 
on:
  #to begin you want to launch this job in main (master for me i don't rename it) and develop
  push:
    branches:
      - master
      - develop
  pull_request:
    branches:
      - master
      - develop
jobs: 
  test-backend:
    runs-on: ubuntu-18.04 
    steps:
      - uses: actions/checkout@v2.3.3 #checkout your github code using actions/checkout@v2.3.3
      - name: Set up JDK 11
        uses: actions/setup-java@v2 #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
        with:
          distribution: 'temurin'
          java-version: '11'
          cache: 'maven'
      - name: Build and test with Maven #finally build your app with the latest command
        run: mvn clean verify --file ./Backend/api/simple-api/simple-api/pom.xml
  build-and-push-docker-image:
    needs: test-backend # run only when code is compiling and tests are passing 
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          context: ./Backend/api/simple-api/ # relative path to the place where source code with Dockerfile is located
          tags: ${{secrets.DOCKERHUB_USERNAME}}/api:1.0 # Note: tags has to be all lower-case
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          context: ./Database/
          tags: ${{secrets.DOCKERHUB_USERNAME}}/postgres-data-base:1.0
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          context: ./Http-server/
          tags: ${{secrets.DOCKERHUB_USERNAME}}/http-server:1.0
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PWD}}
```

<aside>
ℹ️ For what purpose do we need to push docker images?

</aside>

➡️ pour rendre disponible les images à jour pour les autres développeurs et la production.

# **Setup Quality Gate**

## Quality gate configuration

Attention à désactiver dans **`Administration → Analysis Method → SonarCloud Automatic Analysis = OFF`**

**Coverage** is less than **`80.0%`**

**Duplicated Lines (%)** is greater than **`3.0%`**

**Maintainability Rating** is worse than **`A`**

**Reliability Rating** is worse than **`A`**

**Security Hotspots Reviewed** is less than **`100%`**

**Security Rating** is worse than **`A`**

# **Going further: Split pipeline (Optional)**

**`Ne fonctionne pas`**

```yaml
name: CI devops 2022 CPE 
on:
  #to begin you want to launch this job in main (master for me i don't rename it) and develop
  push:
    branches:
      - master
      - develop
  pull_request:
    branches:
      - master
      - develop
jobs: 
  test-backend:
    runs-on: ubuntu-18.04
    env:
      working-directory: ./Backend/api/simple-api
    steps:
      - uses: actions/checkout@v2.3.3 #checkout your github code using actions/checkout@v2.3.3
      - name: Set up JDK 11
        uses: actions/setup-java@v2 #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
        with:
          distribution: 'temurin'
          java-version: '11'
          cache: 'maven'
      - name: Build and test with Maven #finally build your app with the latest command
        env:
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=MIGEWANTAlexis_devops-tp1 -Dsonar.organization=migewantalexis -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./Backend/api/simple-api/simple-api/pom.xml
  
  build-and-push-api:
    needs: test-backend # run only when code is compiling and tests are passing 
    runs-on: ubuntu-latest
    env:
      working-directory: ./Backend/api/simple-api
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PWD}}
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
          context: ./Backend/api/simple-api/ # relative path to the place where source code with Dockerfile is located
          tags: ${{secrets.DOCKERHUB_USERNAME}}/api:1.0 # Note: tags has to be all lower-case
          push: ${{ github.ref == 'refs/heads/master' }}

  build-and-push-database:
    needs: test-backend # run only when code is compiling and tests are passing 
    runs-on: ubuntu-latest
    env:
      working-directory: ./Database
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PWD}}
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          context: ./Database
          tags: ${{secrets.DOCKERHUB_USERNAME}}/postgres-data-base:1.0
          push: ${{ github.ref == 'refs/heads/master' }}

  build-and-push-http-server:
    needs: test-backend # run only when code is compiling and tests are passing 
    runs-on: ubuntu-latest
    env:
      working-directory: ./Http-server
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PWD}}
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          context: ./Http-server
          tags: ${{secrets.DOCKERHUB_USERNAME}}/http-server:1.0
          push: ${{ github.ref == 'refs/heads/master' }}
```