# Cognhacker

## Notes d’installation de la VM

- Déploiement Ubuntu 22.04.1 LTS avec le package OpenSSH + Docker
- Changement des MDP root et cognhacker
- Installation d’utilitaires : ``apt install -y curl wget git screen htop vim nano sudo tree iperf3 unzip sshfs nmap apache2 network-manager net-tools``
- MAJ et redémarrage
- Mise en service d’apache et clone du site
- ``echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf``

## Installation de WorkAdventure

Quelques tutos :

* https://shownotes.opensourceisawesome.com/virtual-office-to-virtual-meetings-with-workadventure/
* https://wiki.techinc.nl/Work-Adventure/install
* https://github.com/thecodingmachine/workadventure/tree/master/contrib/docker

Création du **docker-compose.yml**

````
version: "3.3"
services:
  reverse-proxy:
    image: traefik:v2.5
    command:
      - --log.level=WARN
      - --providers.docker
      - --entryPoints.web.address=:80
    ports:
      - "9999:80"
    depends_on:
      - pusher
      - front
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped

  front:
    #build:
    #  context: ./
    #  dockerfile: front/Dockerfile
    image: thecodingmachine/workadventure-front:v1.12.11
    environment:
      DEBUG_MODE: "$DEBUG_MODE"
      JITSI_URL: $JITSI_URL
      JITSI_PRIVATE_MODE: "$JITSI_PRIVATE_MODE"
      PUSHER_URL: /pusher
      ADMIN_URL: /admin
      TURN_SERVER: "${TURN_SERVER}"
      TURN_USER: "${TURN_USER}"
      TURN_PASSWORD: "${TURN_PASSWORD}"
      MAX_PER_GROUP: "${MAX_PER_GROUP}"
      MAX_USERNAME_LENGTH: "${MAX_USERNAME_LENGTH}"
      START_ROOM_URL: "${START_ROOM_URL}"
      DISABLE_NOTIFICATIONS: "${DISABLE_NOTIFICATIONS}"
      SKIP_RENDER_OPTIMIZATIONS: "${SKIP_RENDER_OPTIMIZATIONS}"
    labels:
      - "traefik.http.routers.front.rule=PathPrefix(`/`)"
      - "traefik.http.routers.front.entryPoints=web"
      - "traefik.http.services.front.loadbalancer.server.port=80"
      - "traefik.http.routers.front.service=front"
    restart: unless-stopped

  pusher:
    #build:
    #  context: ./
    #  dockerfile: pusher/Dockerfile
    image: thecodingmachine/workadventure-pusher:v1.12.11
    environment:
      SECRET_JITSI_KEY: "${SECRET_JITSI_KEY}"
      SECRET_KEY: ${SECRET_KEY}
      API_URL: back:50051
      ADMIN_API_URL: "${ADMIN_API_URL}"
      ADMIN_API_TOKEN: "${ADMIN_API_TOKEN}"
      JITSI_URL: ${JITSI_URL}
      JITSI_ISS: ${JITSI_ISS}
      FRONT_URL : ${FRONT_URL}
	  START_ROOM_URL: "${START_ROOM_URL}"
    labels:
      - "traefik.http.middlewares.strip-pusher-prefix.stripprefix.prefixes=/pusher"
      - "traefik.http.routers.pusher.rule=PathPrefix(`/pusher`)"
      - "traefik.http.routers.pusher.middlewares=strip-pusher-prefix@docker"
      - "traefik.http.routers.pusher.entryPoints=web"
      - "traefik.http.services.pusher.loadbalancer.server.port=8080"
      - "traefik.http.routers.pusher.service=pusher"
    restart: unless-stopped

  back:
    #build:
    #  context: ./
    #  dockerfile: back/Dockerfile
    image: thecodingmachine/workadventure-back:v1.12.11
    environment:
      SECRET_KEY: ${SECRET_KEY}
      STARTUP_COMMAND_1: yarn install
      SECRET_JITSI_KEY: "${SECRET_JITSI_KEY}"
      ADMIN_API_TOKEN: "${ADMIN_API_TOKEN}"
      ADMIN_API_URL: "${ADMIN_API_URL}"
      JITSI_URL: ${JITSI_URL}
      JITSI_ISS: ${JITSI_ISS}
      MAX_PER_GROUP: ${MAX_PER_GROUP}
      TURN_STATIC_AUTH_SECRET: "${TURN_STATIC_AUTH_SECRET}"
      REDIS_HOST: redis
    labels:
      - "traefik.http.middlewares.strip-api-prefix.stripprefix.prefixes=/api"
      - "traefik.http.routers.back.rule=PathPrefix(`/api`)"
      - "traefik.http.routers.back.middlewares=strip-api-prefix@docker"
      - "traefik.http.routers.back.entryPoints=web"
      - "traefik.http.services.back.loadbalancer.server.port=8080"
      - "traefik.http.routers.back.service=back"
    restart: unless-stopped

  redis:
    image: redis:6
    restart: unless-stopped

  wordpress:
    image: wordpress
    restart: unless-stopped
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: cognhacker
      WORDPRESS_DB_PASSWORD: **********************
      WORDPRESS_DB_NAME: cognhacker
    volumes:
      - /var/www/html:/var/www/html

  db:
    image: mysql:5.7
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: cognhacker
      MYSQL_USER: cognhacker
      MYSQL_PASSWORD: **********************
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
````

**.env**
````
# The base domain
DOMAIN=lab.cognhacker.net

DEBUG_MODE=false
JITSI_URL=meet.jit.si
# If your Jitsi environment has authentication set up, you MUST set JITSI_PRIVATE_MODE to "true" and you MUST pass a SECRET_JITSI_KEY to generate the JWT secret
JITSI_PRIVATE_MODE=false
JITSI_ISS=
SECRET_JITSI_KEY=

# URL of the TURN server (needed to "punch a hole" through some networks for P2P connections)
# TURN_SERVER=
# TURN_USER=
# TURN_PASSWORD=

# The URL used by default, in the form: "/_/global/map/url.json"
START_ROOM_URL=/_/global/cognhacker.github.io/wa/maps/lab.json

# The email address used by Let's encrypt to send renewal warnings (compulsory)
ACME_EMAIL=orga@cognhacker.net

# Set to true to allow using this instance as a target for the apiUrl property
FEDERATE_PUSHER=false

# Server settings
MAX_PER_GROUP=10
MAX_USERNAME_LENGTH=12
DISABLE_NOTIFICATIONS=false
SKIP_RENDER_OPTIMIZATIONS=false

# Enable / disable chat
ENABLE_CHAT=false

# Secrets
SECRET_KEY="**********************"
ADMIN_API_TOKEN="**********************"
ADMIN_API_URL=
````

Lancement :

````
sudo docker-compose -f /home/cognhacker/docker-compose.yml up -d --build
````
