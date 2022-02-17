# A Matrix (Synapse) Stack with coturn, bots, bridges and more
A docker-compose stack with Synapse, Postgres, Element-Web, Turn and more

This is how I was "serving" a small chat server on an Ubuntu virtual machine.

The stack follows some specific logic concerning the file organization and a couple "bad practices" (exposing ports and folders) that should not be a problem for a non production environment.

# Compoments (and images used)
- Postgres - `postgres:latest`
- Synapse homeserver - `matrixdotorg/synapse:latest`
- Element Web Client - `vectorim/element-web`
- Synapse Admin - `awesometechnologies/synapse-admin`
- Turn Server - `instrumentisto/coturn`
- Telegram Bridge - `dock.mau.dev/tulir/mautrix-telegram:latest`
- Facebook Bridge - `dock.mau.dev/tulir/mautrix-facebook:latest`
- Maubot bot manager - `dock.mau.dev/maubot/maubot:latest`
- Webhook Appservice - `turt2live/matrix-appservice-webhooks`


# Assuptions
## Domain and subdomains

You should have a locally (at least) resolved domain (During the instructions we will use `domain.ltd`). We also use the following subdomains at various points:
- matrix.ms.local
- turn.ms.local
- webhooks.ms.local
- proxy.ms.local
- maubot.ms.local



## Certificates

The guide assumes you have a wildcard ceritificate for your domain name (`WILDCARD.ms.local`) in `CERT_PATH` folder. 
```
/mnt/
  certs/
    WILDCARD.domain.ltd.crt
    WILDCARD.domain.ltd.key
```

You can ofcource use diffrent certificates for every service. 
_Certificate generation is outside of the scope of this guide, for now._
## Folder hiercacy

The docker-compose.yaml file assumes the following hiecrasy:
```
/BASE_FOLDER/
    configs/
      db/
      homeserver/
      webchat/
      turn/
      telegram-bridge/
      facebook-bridge/
      webhook-service/
      maubot/
    data/
      homeserver_media-store
      turn
    certs/
```
- `/configs/` : configuration persistent data

- `/certs/` : certificates

- `/data/` : other kind of persistent data (like synapse media store etc.)

## Docker volumes and networks

- Create a docker volume for postgres: `sudo docker volume create db-data`
- The three required networks (`db`,`bots` and `ms`) will be created automatically. If the names overlap with anything already running, you should edit `docker-compose.yaml`


# Initialization

## Expsose ENV

Edit `.env` file to your liking. Then expose each ENV with `export VAR=VAL`. You will need:
```
export DOMAIN=ms.local
export CONF_PATH=/mnt/configs
```

Some of the services need to initialize some config files before you can finally start them. Below are the steps and a reasoning behind them:

## Synapse
Use the following command to generate a `homeserver.yaml` file in `${CONF_PATH}/homeserver/`. __IMPORTANT: the subdomain (`matrix.${DOMAIN}`) CANNOT be changed later. Make sure you have decided correctly.__

```
sudo docker run -it --rm \
    -v=${CONF_PATH}/homeserver:/data \
    -e SYNAPSE_SERVER_NAME=matrix.${DOMAIN} \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest generate
```
After that you can edit the file however you want. Some important fields are:

- `server_name` will be autofilled
```
server_name: "matrix.ms.local"
```

- We add an https listener for secure connections, bind it to all addresses and enable federation.
```
listeners:
  - port: 8448
    type: http
    tls: true
    bind_addresses: ['0.0.0.0']
    x_forwarded: true

    resources:
      - names: [client]
        compress: true
      - names: [federation]
        compress: false


  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    bind_addresses: ['0.0.0.0']
    resources:
      - names: [client]
        compress: true

```

- Add the postgress info to connect to `db` container
```
database:
  name: psycopg2
  args:
    user: synapse
    password: 12345
    database: synapse_db
    host: db
    cp_min: 5
    cp_max: 10
```

- Change the default `media_store` path to the that will be mounted in `docker-compose.yaml`
```
media_store_path: "/media_store"
```

- Specify the path to our certificate
```
tls_certificate_path: "/certs/WILDCARD.ms.local.crt"
tls_private_key_path: "/certs/WILDCARD.ms.local.key"
```

- Finally, enable registrations
```
enable_registration: true
```

- Save the file (_We will edit more while configuring Turn, Bridges and Bots_)

## Bridges and Bots

### Telegram Brige
_Source_: https://docs.mau.fi/bridges/python/setup/docker.html?bridge=telegram

1. Run:
```
sudo docker run --rm -v ${CONF_PATH}/telegram-bridge:/data:z dock.mau.dev/mautrix/telegram:latest
```
This will generate a `config.yaml` that you should edit. 

2. You need to set at least the following:
- Main connection configurations (_Since this is a dev/testing server we will use HTTPS but we won't verify any certificates between the bridge and the homeserver. Same goes for othe bridges and services_)
```
homeserver:
    # The address that this appservice can use to connect to the homeserver.
    address: https://homeserver:8448
    # The domain of the homeserver (for MXIDs, etc).
    domain: matrix.ms.local
    # Whether or not to verify the SSL certificate of the homeserver.
    # Only applies if address starts with https://
    verify_ssl: false

appservice:
    # The address that the homeserver can use to connect to this appservice.
    address: http://telegram-bridge:29317
    database: sqlite:////data/telegram-bridge.db
```
- Bridge permissions

We should also give permission to some users to use the bridge. Since we don't even have a homeserver yet we will give admin permissions to all users that share the domain `matrix.ms.local` . Edit the following:
```
permissions:
        "*": relaybot
        "matrix.ms.local": admin
```
- Telegram API key
```
telegram:
    # Get your own API keys at https://my.telegram.org/apps
    api_id: 12345
    api_hash: tjyd5yge35lbodk1xwzw2jstp90k55qz

```

3. Run the docker command again to generate a 'registration.yaml'
```
sudo docker run --rm -v ${CONF_PATH}/telegram-bridge:/data:z dock.mau.dev/mautrix/telegram:latest
```
 The `registration.yaml` file is mounted on the `homeserver` cotainer. 


### Facebook Bridge (Almost identical to Telegram bridge)
_Source_: https://docs.mau.fi/bridges/python/setup/docker.html?bridge=facebook

Run:
```
 docker run --rm -v ${CONF_PATH}/facebook-bridge:/data:z dock.mau.dev/mautrix/facebook:latest
```
This will generate a `config.yaml` that you should edit. You need to set at least the following:
- Main connection configurations (_Since this is a dev/testing server we will use HTTPS but we won't verify any certificates between the bridge and the homeserver. Same goes for othe bridges and services_)

```
homeserver:
    # The address that this appservice can use to connect to the homeserver.
    address: https://homeserver:8448
    # The domain of the homeserver (for MXIDs, etc).
    domain: matrix.ms.local
    # Whether or not to verify the SSL certificate of the homeserver.
    # Only applies if address starts with https://
    verify_ssl: false

appservice:
    # The address that the homeserver can use to connect to this appservice.
    address: http://facebook-bridge:29317
    database: sqlite:////data/facebook-bridge.db
```
- Bridge permissions

We should also give permission to some users to use the bridge. Since we don't even have a homeserver yet we will give admin permissions to all users that share the domain `matrix.ms.local` . Edit the following:
```
permissions:
        "*": "relay"
        "matrix.ms.local": "admin"
```



Run the docker command again to generate a 'registration.yaml'
```
sudo docker run --rm -v ${CONF_PATH}/facebook-bridge:/data:z dock.mau.dev/mautrix/facebook:latest
```


 The `registration.yaml` file is mounted on the `homeserver` cotainer.

### Webhook App Service
Source: https://github.com/turt2live/matrix-appservice-webhooks#docker 

1. Create an `appservice-registration-webhooks.yaml` file in `${CONF_PATH}/webhooks` and copy the following (make sure you generate `hs_token` and `as_token`):

```
id: webhooks
hs_token: A_RANDOM_ALPHANUMERIC_STRING  # CHANGE THIS
as_token: ANOTHER_RANDOM_ALPHANUMERIC_STRING  # CHANGE THIS
namespaces:
  users:
    - exclusive: true
      regex: '@_webhook.*'
url: 'http://webhook-service:9000'  
sender_localpart: webhooks
rate_limited: false
```

2. Create an `config.yaml` file in `${CONF_PATH}/webhooks` and copy/edit the following:
```
# Configuration specific to the application service. All fields (unless otherwise marked) are required.
homeserver:
  # The domain for the client-server API calls.
  url: "http://homeserver:8008"

  # The domain part for user IDs on this home server. Usually, but not always, this is the same as the
  # home server's URL.
  domain: "matrix.ms.local"

# Configuration specific to the bridge. All fields (unless otherwise marked) are required.
webhookBot:
  # The localpart to use for the bot. May require re-registering the application service.
  localpart: "webhooks"

  # Appearance options for the Matrix bot
  appearance:
    displayName: "Webhook Bridge"
    avatarUrl: "http://i.imgur.com/IDOBtEJ.png" # webhook icon

# Provisioning API options
provisioning:
  # Your secret for the API. Required for all provisioning API requests.
  secret: 'CHANGE_ME'

# Configuration related to the web portion of the bridge. Handles the inbound webhooks
web:
  hookUrlBase: 'https://webhooks.domain.ltd'

logging:
  file: logs/webhook.log
  console: true
  consoleLevel: debug
  fileLevel: verbose
  writeFiles: true
  rotate:
    size: 52428800 # bytes, default is 50mb
    count: 5

```

3. Create a `database.json` file in `${CONF_PATH}/webhooks` and copy the following:
```

{
  "defaultEnv": {
    "ENV": "NODE_ENV"
  },
  "development": {
    "driver": "sqlite3",
    "filename": "/data/development.db"
  },
  "production": {
    "driver": "sqlite3",
    "filename": "/data/production.db"
  }
}

```

4. Run:
```
sudo docker run --rm -v ${CONF_PATH}/webhooks:/data turt2live/matrix-appservice-webhooks
```
Check the logs for any errors. If you get an `[ERROR] ConnectionError: request failed: getaddrinfo ENOTFOUND homeserver homeserver:8008`, this is normal since we don't have a working homeserver yet.

### Maubot Manager
_Source_: https://docs.mau.fi/maubot/usage/setup/docker.html

1. Run: 
```
sudo docker run --rm -v ${CONF_PATH}/maubot:/data:z dock.mau.dev/maubot/maubot:latest
```

This will generate a `config.yaml` file. 

2. Update the file to your liking. You should at least add your homeserver:
```
homeservers:
   matrix.ms.local
      url: https://homeserver:8448
      secret: <THE registration_shared_secret FROM homeserver.yaml>

```
3. Save the file

### Registering the new services to the home server:

Edit `homeserver.yaml` and add the following:
```
app_service_config_files:
  - /app_services/telegram-registration.yaml
  - /app_services/facebook-registration.yaml
  - /app_services/webhooks-registration.yaml
```
(in the docker-compose file we have mounted each file in the `homeserver` container)

## Turn server (for audio and video calls)

Create a new file `turnserver.conf` in `${CONF_PATH}/turn/`. Copy and paste the sample file from: https://github.com/coturn/coturn/blob/master/docker/coturn/turnserver.conf

Edit the following in the file:

- Specify and external ip
```
external-ip=<YOUR PUBLIC IP,  IF YOU PLAN TO USE IT FROM THE INTERNET>
external-ip=<YOUR DOCKER HOST IP>
```
- Specify a port range
```
min-port=64000
max-port=65535
```
This range worked perfectly for me but you should define your own depending on your network setup

- Certificates:
```
cert=/certs/WILDCARD.ms.local.crt
pkey=/certs/WILDCARD.ms.local.key
```
- Define a realm
```
realm=turn.domain.ltd
```

- Uncomment `use-auth-secret`. Generate a alphanumeric and fill `static-auth-secret=`. 
- In `homeserver.yaml` in `##TURN##` secrion paste the same alpanumeric at `turn_shared_secret: "ALPHANUMERIC"` and add the following `turn_uris`
```
turn_uris:
 - "turn:turn.domain.ltd?transport=udp"
 - "turn:turn.domain.ltd?transport=tcp"
 - "turns:turn.domain.ltd:5349?transport=udp"
 - "turns:turn.domain.ltd:5349?transport=tcp"
```




# Bringing up the Chat Server

If everything is correctly initialized we can bring up the stack with `sudo docker-compose up`

After a while we should be able to visit the web element UI at `http://<DOCKER-HOST-IP>:10000`, and register a new user.
