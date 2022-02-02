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

→ Testcontainers est une librairie qui permet de facilement démarrer un conteneur Docker au sein des tests unitaires. Il fournit une extension JUnit 5 et une extension permettant de facilement lancer des bases de données. Son utilisation est donc idéale pour lancer notre base de données Postgres pour nos tests unitaires.

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
ℹ️ Why did we put needs: build-and-test-backend on this job?

</aside>

→ on doit avoir le backend de build et testé, avant de build et de push les images sur le hub.

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

→ pour rendre disponible les images à jour pour les autres développeurs et la production.

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
        env:
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=MIGEWANTAlexis_devops-tp1 -Dsonar.organization=migewantalexis -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./Backend/api/simple-api/simple-api/pom.xml
  
  build-and-push-api:
    needs: test-backend # run only when code is compiling and tests are passing 
    runs-on: ubuntu-latest
    env:
      working-directory: ./Backend/api/simple-api/
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
      working-directory: ./Database/
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PWD}}
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          context: ./Database/
          tags: ${{secrets.DOCKERHUB_USERNAME}}/postgres-data-base:1.0
          push: ${{ github.ref == 'refs/heads/master' }}

  build-and-push-http-server:
    needs: test-backend # run only when code is compiling and tests are passing 
    runs-on: ubuntu-latest
    env:
      working-directory: ./Http-server/
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PWD}}
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          context: ./Http-server/
          tags: ${{secrets.DOCKERHUB_USERNAME}}/http-server:1.0
          push: ${{ github.ref == 'refs/heads/master' }}
```

```
.github
│   └── workflows
│       ├── .api-push.yml
│       ├── .database-http-server.yml
│       └── .test-backend.yml
```

**`.api-push.yml`**

```yaml
name: Api push
on: 
  workflow_run:
    workflows: [Test backend]
    types:
      - completed
    branches: [master]
jobs:
  build-and-push-api:
    runs-on: ubuntu-latest
    env:
      working-directory: ./Backend/api/simple-api/
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
```

**`.database-http-server.yml`**

```yaml
name: Database and HTTP server push
on: 
  push:
    branches:
      - master
  pull_request:
    branches:
      - master
jobs:
  build-and-push-database:
    runs-on: ubuntu-latest
    env:
      working-directory: ./Database/
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PWD}}
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          context: ./Database/
          tags: ${{secrets.DOCKERHUB_USERNAME}}/postgres-data-base:1.0
          push: ${{ github.ref == 'refs/heads/master' }}

  build-and-push-http-server:
    runs-on: ubuntu-latest
    env:
      working-directory: ./Http-server/
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_PWD}}
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          context: ./Http-server/
          tags: ${{secrets.DOCKERHUB_USERNAME}}/http-server:1.0
          push: ${{ github.ref == 'refs/heads/master' }}
```

**`.test-backend.yml`**

```yaml
name: Test backend
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
        env:
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=MIGEWANTAlexis_devops-tp1 -Dsonar.organization=migewantalexis -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./Backend/api/simple-api/simple-api/pom.xml
```