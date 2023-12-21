# Projet "HumansBestFriend"

## Description

Ce projet est une application distribuée composée de plusieurs services interconnectés, chacun remplissant un rôle spécifique. L'application permet de gérer un système de vote pour élire le meilleur ami de l'homme.

## Architecture

L'architecture du projet est basée sur des services Dockerisés et orchestre l'ensemble à l'aide de Docker Compose. Voici une brève description de chaque service :

- **vote**: Gère le système de vote. Les utilisateurs peuvent voter pour leur animal préféré.
- **result**: Affiche les résultats du vote. Dépend du service de base de données (`db`) pour récupérer les données de vote.
- **seed-data**: Charge les données initiales dans le système, en dépendant du service de vote pour s'assurer de sa disponibilité.
- **worker**: Effectue des tâches de fond, dépendant des services Redis et Postgres.
- **postgres**: Base de données PostgreSQL pour stocker les données de vote.
- **redis**: Système de messagerie pour les tâches asynchrones et la coordination entre les services.

## Prérequis

Assurez-vous d'avoir installé les outils suivants sur votre machine avant de démarrer le projet :

- Docker
- Docker Compose

## Instructions d'utilisation

1. Clonez ce dépôt sur votre machine locale :

    ```bash
    git clone https://github.com/votre-utilisateur/humansbestfriend.git
    ```

2. Naviguez vers le répertoire du projet :

    ```bash
    cd humansbestfriend
    ```

3. Création du  registry pour deployer nos images :

    ```bash
    docker run -d -p 5000:5000 --restart always --name registry registry:2
    ```

    Cela construira une image registry qui nous permettra de stocker nos images chez nous 
4. Tag des images :

    ```bash
    docker tag nom_image localhost:5000/nom_image:version
    ```
5. push de l'image dans le registry :

    ```bash
    docker push localhost:5000/nom_image:version
    ```

5. Vérifier si l'image est bien push :

    ```bash
    curl localhost:5000/v2/_catalog
    ```

5. Création d'un fichier docker-compose.build.yml  et copier coller le code suivant :

    ```bash
    
    services:
    worker:
        build:
        context: ./worker
        depends_on:
        redis:
            condition: service_healthy
        postgres:
            condition: service_healthy
        networks:
        - humansbestfriend-network

    vote:
        build:
        context: ./vote
        volumes:
        - ./vote:/usr/local/app
        ports:
        - "5002:80"
        networks:
        - humansbestfriend-network
        healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost"]
        interval: 15s
        timeout: 5s
        retries: 3
        start_period: 10s

    seed-data:
        build:
        context: ./seed-data
        depends_on:
        vote:
            condition: service_healthy
        restart: "no"
        networks:
        - humansbestfriend-network

    result:
        build:
        context: ./result
        depends_on:
        postgres:
            condition: service_healthy
        volumes:
        - ./result:/usr/local/app
        ports:
        - "5001:80"
        - "127.0.0.1:9229:9229"
        networks:
        - humansbestfriend-network

    postgres:
        image: postgres:15-alpine
        volumes:
        - "db-data:/var/lib/postgresql/data"
        - "./healthchecks:/healthchecks"
        healthcheck:
        test: /healthchecks/postgres.sh
        interval: "5s"
        networks:
        - humansbestfriend-network

    redis:
        image: redis
        networks:
        - humansbestfriend-network
        volumes:
        - "./healthchecks:/healthchecks"
        healthcheck:
        test: /healthchecks/redis.sh
        interval: "5s"

    networks:
    humansbestfriend-network:

    volumes:
    db-data:

    ```

6. buil des images :
     ```bash
    docker compose -f docker-compose.build.yml
    ```
7. Creation d'un fichier compose.yml et copier coller le code  :
     ```bash
    
        services:
        vote:
            image: localhost:5000/vote:v1
            ports:
            - "5001:80"
        result:
            image: localhost:5000/result:v1
            ports:
            - "5002:80"
        seed-data:
            image: localhost:5000/seed-data:v1
        worker:
            image: localhost:5000/worker:v1
    ```
8. build des containers  :
     ```bash
        docker compose up -d
    ```

9. Accédez à l'interface utilisateur de vote à l'adresse [http://192.168.21.21:5002] et profitez du système de vote.

10. Accédez à l'interface utilisateur de resultat à l'adresse [http://192.168.21.21:5001] et profitez du système de vote.
