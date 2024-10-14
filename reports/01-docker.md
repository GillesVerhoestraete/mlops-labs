# Lab Report: SUBJECT

## Student information

- Student name: Gilles Verhoestraete
- Student code: 202289627

## Assignment description

Describe the assignment in your own words. What did you have to do? What was the goal of the assignment?

## Proof of work done

Include screenshots, code snippets, etc. to prove that you completed the assignment. If you have a lot of code, you can link to a separate file or repository.

### 1.1 Set up the lab environment

Docker is geinstalleerd.

![Bewijs-Docker-Geinstalleerd](../reports/img/Bewijs-Docker.png)

### 1.2 Configure Portainer

Ik heb de docker compose file uitgevoerd via het commando `docker-compose -f docker-compose.portainer.yml up -d`. Dit commando zorgt ervoor dat Portainer word opgestart in een docker container. Hierna ben ik via google naar `https://localhost:9443` gegaan waar ik een Portainer account heb gemaakt.

![Ingelogt-Portainer](../reports/img/Portainer-Ingelogt.png)

### 1.3 Create a Docker image for a simple web application

Ik heb een docker file aangemaakt met de volgende code in:

```a
FROM node:20

WORKDIR /app

COPY . .

RUN yarn install --frozen-lockfile

EXPOSE 3000

CMD ["yarn", "start"]

```

Daarna heb ik de docker image aangemaakt door `docker build -t webapp .` Dit werkt alleen maar als je in de directory van Webapp zit.

Daarna kan je de container starten met deze code: `docker run -p 3000:3000 webapp`

Om te kijken of dit werkt heb ik het eerste dier opgevraagt in mijn terminal en dit werkt via deze code `curl http://localhost:3000/animals/1`

Heb docker gestopt met deze code: `docker stop id-container`

### 1.4 Create a Docker Compose file

docker compose installeren via `sudo apt-get install docker-compose-plugin`

Ik maak een docker-compose file aan en zet de volgende code er in: 

```a
version: '3'
services:
  webapp:
    build: .
    ports:
      - "3000:3000"
    container_name: webapp-container
```

Ik gebruik volgende code om de docker compose file te starten: `docker-compose up -d`

### 1.5 Backup the database

Een shell in de container maken via deze code: `docker exec -it webapp-container sh`

Dit werkt als je in je Terminal een # vooraan ziet staan.

aan ons docker compose de volgende lijn code toevoegen zodat de database op het host systeem word opgeslagen: `volumes: - ./database:/app/database`

Voeg `database/` toe aan het gitignore bestand om te voorkomen dat de database wordt toegevoegd aan je Git repository.

Dan starten we de docker compose file opnieuw op door eerst hem af te sluiten via: `docker-compose down`  en dan hem opnieuw te builden: `docker-compose up build -d`

![database-sqlite](../reports/img/database-sqlite.png)

### 1.6 Add a database service

De inhoud van mijn nieuwe docker compose file:

```a
version: '3'
services:
  webapp:
    build: .
    ports:
      - "3000:3000"
    container_name: webapp-container
    environment:
      - MYSQL_URL=mysql://root:hogent23@database:3306/docker-opdracht-1  
    depends_on:
      database:
        condition: service_healthy  # Wacht tot de database gezond is
    command: ["sh", "-c", "sleep 10 && yarn start"]  # Voeg een vertraging toe

  database:
    image: mysql:latest
    environment:
      MYSQL_ROOT_PASSWORD: hogent23 
      MYSQL_DATABASE: docker-opdracht-1                
      MYSQL_USER: gilles                       
      MYSQL_PASSWORD: hogent23    
      MYSQL_ROOT_HOST: '%'           
    ports:
      - "3307:3306"  
    volumes:
      - db_data:/var/lib/mysql   
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:

```

Nu start ik docker compose opnieuw op: `docker-compose up`

ik heb depends on moeten toevoegen aan mijn docker compose file zodat het pas verbinding probeert te maken als de mysql databank is opgestart.

Je kan zien dat dit werkt door te surfen naar `http://localhost:3000/animals` of via portainer.

![database-sqlite](../reports/img/mysql-gelukt.png)

### 1.7 Backup the database

Om de data op te slaan in de docker area in plaats van de docker container zelf heb ik deze code toegevoegd aan mijn docker compose file:

```a
volumes:
      - db_data:/var/lib/mysql   
```

### 1.8 Optimizing the Docker image

De grote van mijn image is 1.24GB.

![image size](../reports/img/Size-Images.png)

Ik heb mijn versie van mijn docker file aangepast van `FROM node:20` naar `FROM node:20-alpine`, en daarna heb ik mijn docker compose file opnieuwt gebuild.

Nu is de size van mijn image 276MB.

![reduced-size](../reports/img/image-Size-Reduced.png)

Ik heb 15 lagen in mijn image.

![layers](../reports/img/layers-image.png)

Ik heb mijn dockerfile aangepast naar:

```a
FROM node:20-alpine

WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install --frozen-lockfile

COPY . .

EXPOSE 3000

CMD ["yarn", "start"]

```

Daarna heb ik hem opnieuw gebuild en mijn image size is nog steeds 276MB en ik heb ook nog steeds 15 layers. Hij voert de build rapper uit, je optimaliseert hier vooral de volgorde van de lagen voor caching.

Daarna heb ik een `.dockerignore` bestand gemaakt en hier heb ik de volgende bestanden in geplaatst

```a
node_modules
Dockerfile
docker-compose.yml
docker-compose-sqlite.yml
```

De docker image is nu 234MB groot en heeft nog steeds 15 lagen.

![Einde reducing](../reports/img/einde-reducing.png)

### 1.9 Testing the application

Ik heb volgende code toegevoegd aan mijn docker compose file

```a
test:
    build: .  
    environment:
      - API_URL=http://webapp:3000  
    depends_on:
      webapp:
        condition: service_started  
    command: ["yarn", "test"]  
```

Om dit dan allemaal uit te voeren en te testen voer ik eerst `docker-compose up -d webapp database`, dit zorgt ervoor dat de web services eerst draaien voordat we de test gaan uitvoeren. Daarna voeren we deze code uit `docker-compose up -d test`, dit voert dan de testen uit.

### 1.10 Pushing the Docker image to Docker Hub

De nieuwe docker-compose file ziet er nu zo uit, er is `image: gillesverhoestraete/webapp` toegevoegd:

```a
version: '3.8'

services:
  webapp:
    image: gillesverhoestraete/webapp  
    ports:
      - "3000:3000"
    container_name: webapp-container
    environment:
      - MYSQL_URL=mysql://root:hogent23@database:3306/docker-opdracht-1
    depends_on:
      database:
        condition: service_started  
    command: ["sh", "-c", "sleep 10 && yarn start"]

  database:
    image: mysql:latest
    environment:
      MYSQL_ROOT_PASSWORD: hogent23
      MYSQL_DATABASE: docker-opdracht-1
      MYSQL_USER: gilles
      MYSQL_PASSWORD: hogent23
      MYSQL_ROOT_HOST: '%'
    ports:
      - "3307:3306"
    volumes:
      - db_data:/var/lib/mysql

  test:
    image: gillesverhoestraete/webapp  
    environment:
      - API_URL=http://webapp:3000  
    depends_on:
      webapp:
        condition: service_started  
    command: ["yarn", "test"]  

volumes:
  db_data:
```

## Evaluation criteria

- [x] Show that you created a Docker image for the API
- [x] Show that you can start the API using the SQLite database
- [x] Show that you can start the API using the MySQL database
- [x] Show that you can access the API on port 3000
- [x] Show that you optimized the Docker image size
  - [x] You've used an Alpine version of Node.js
  - [x] You've copied and installed the dependencies in a separate layer
- [x] Show all running containers in the Portainer dashboard
- [x] Show that all tests are passing
- [x] Show that you pushed the Docker image to Docker Hub and that you can pull it from Docker Hub
- [x] Show that you wrote an elaborate lab report in Markdown and pushed it to the repository
- [x] Show that you updated the cheat sheet with the commands you need to remember

## Issues

Toen ik de webapp wou starten kreeg ik een foutmeldingen dat Yarn niet was geinstalleerd en gaven mij direct een mogelijke oplossing dat dit zou kunnen oplossen: ´sudo apt install cmdtest´. Dit blijkt niet de juiste oplossing te zijn waardoor ik nog eens appart moest opzoeken hoe ik Yarn moest installeren.

![Yarn-Installeren-foutieve](../reports/img/yarnInstalleren-cmdTest.png)

Ik heb dan uiteindelijk Yarn geinstalleerd via deze code:

```a
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt update && sudo apt install yarn 
```

Ik heb dat ook nog mijn node.js moeten updaten want het was een te oude versie via ´sudo apt-get install -y nodejs´
Daarna heb ik volgende code uitgevoerd:
- sudo apt-get remove nodejs
- curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
- sudo apt-get install -y nodejs

Achteraf gezien had ik dit allemaal niet nodig want ik had de opdracht verkeerd gelezen en moest ik gewoon een dockerfile aanmaken.

Er draait al iets op poort 3306 dus moet ik mijn mysql database draaien op poort 3307. Dit heb ik gewoon gedaan doormiddel van mijn docker compose file aan te passen.

Mijn mysql server starte te laat op waardoor ik via mijn docker compose geen verbinding kon maken, ik heb dit opgelost door een delay en een healthcheck toe te voegen aan mijn docker compose file.

## Reflection

What was difficult? What was easy? What did you learn from the assignment? What would you do differently if you had to do it over? If you have any other remarks, feel free to share them.

## Resources

List all sources of useful information that you encountered while completing this assignment: books, manuals, HOWTO's, blog posts, etc.
