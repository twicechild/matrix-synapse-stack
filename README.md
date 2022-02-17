# A Matrix (Synapse) Stack with Traefik, bots, bridges and more

This is a stack in a single `docker-compose.yaml` file. The guide starts by preconfiguring the various services and finally bringing the stack up.

The stack follows some specific logic concerning the file organization and a couple "bad practices" (exposing ports and folders) that should not be a problem for a non production environment.

# Compoments (and images used)
- Postgres - `postgres:latest`
- Synapse homeserver - `matrixdotorg/synapse:latest`
- Element Web Client - `vectorim/element-web`
- Synapse Admin - `awesometechnologies/synapse-admin`
- Telegram Bridge - `dock.mau.dev/tulir/mautrix-telegram:latest`
- Facebook Bridge - `dock.mau.dev/tulir/mautrix-facebook:latest`
- Maubot bot manager - `dock.mau.dev/maubot/maubot:latest`
- Webhook Appservice - `turt2live/matrix-appservice-webhooks`


# Assuptions

<br/>

## Domain and subdomains

You should have a locally (at least) resolved domain (During the instructions we will use `ms.local`). We also use the following subdomains at various points:
- matrix.ms.local
- turn.ms.local
- webhooks.ms.local
- proxy.ms.local
- maubot.ms.local


<br/>


## Certificates

The guide assumes you have a wildcard ceritificate for your domain name (`WILDCARD.ms.local`) in `CERT_PATH` folder. 
```
/${CERT_PATH}/
    WILDCARD.ms.local.crt
    WILDCARD.ms.local.key
```

You can genarate a self-signed certificate folowing guide from @cecilemuller:
https://gist.github.com/cecilemuller/9492b848eb8fe46d462abeb26656c4f8

You can ofcource use diffrent certificates for every service. 

<br/>

## Folder hierarchy

The docker-compose.yaml file assumes the following hiecrasy:
```
${CONF_PATH}/
      db/
      homeserver/
      webchat/
      telegram-bridge/
      facebook-bridge/
      webhook-service/
      maubot/
${DATA_PATH}
      homeserver_media-store
      turn
${CERT_PATH}/
```
- `/configs/` : configuration persistent data

- `/certs/` : certificates

- `/data/` : other kind of persistent data (like synapse media store etc.)

<br/>


<br/>

# Initialization and preconfigurations

## Expsose ENV

Edit `.env` file to your liking. Then expose each ENV with `export VAR=VAL`. You will need:
```
export DOMAIN=ms.local
export CONF_PATH=/mnt/configs
```

Some of the services need to initialize some config files before you can finally start them.

<br/>

## Proxy
1. Create  file `traefik-ssl.toml` in `${CONF_PATH}/proxy/` and paste the following:
    ```
    [tls]
    [tls.stores]
    [tls.stores.default]
    [tls.stores.default.defaultCertificate]
    certFile = "/certs/WILDCARD.ms.local.crt"
    keyFile = "/certs/WILDCARD.ms.local.key"
    ```
    Change the file name of the certificate if you have to.

<br/>

## Prostgres

- Create a docker volume: `sudo docker volume create db-data`
- Change the values in `db.env` to your liking. You should at least change `POSTGRES_PASSWORD=`

<br/>

## Synapse
Generate a `homeserver.yaml` file in `${CONF_PATH}/homeserver/`. You can find a sample config at `sample_configs/homeserver/homeserver.yaml`


 __IMPORTANT: the subdomain (`matrix.${DOMAIN}`) CANNOT be changed later. Make sure you have decided correctly.__

```
sudo docker run -it --rm \
    -v=${CONF_PATH}/homeserver:/data \
    -e SYNAPSE_SERVER_NAME=matrix.${DOMAIN} \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest generate
```
Edit/Uncomment some important fields:
-  `server_name` will be autofilled
```
server_name: "matrix.ms.local"
```

- Add an https listener for secure connections, bind it to all addresses and enable federation.
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
    password: <SAME AS db.env>
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

- Enable registrations
```
enable_registration: true
```
- Enable user directory search. (This will help us find the bot accounts later)
```
user_directory:
    enabled: true
    search_all_users: true
    prefer_local_users: true
```
- Save the file (_We will edit more while configuring Turn, Bridges and Bots_)

<br/>

<br/>


## Bridges and Bots

<br/>

### Telegram Brige
_Source_: https://docs.mau.fi/bridges/python/setup/docker.html?bridge=telegram

1. Run the command to generate a `config.yaml`:
    ```
    sudo docker run --rm -v ${CONF_PATH}/telegram-bridge:/data:z dock.mau.dev/mautrix/telegram:latest
    ```
 

2. Edit the file (reference `sample_configs/telegram-bridge/config.yaml`):
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

<br/>


### Facebook Bridge (Almost identical to Telegram bridge)
_Source_: https://docs.mau.fi/bridges/python/setup/docker.html?bridge=facebook

1. Run the command to generate a `config.yaml`:
    ```
    sudo docker run --rm -v ${CONF_PATH}/facebook-bridge:/data:z dock.mau.dev/mautrix/facebook:latest
    ```
 

2. Edit the file (reference `sample_configs/facebookm-bridge/config.yaml`):
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

3. Run the docker command again to generate a 'registration.yaml'
    ```
    sudo docker run --rm -v ${CONF_PATH}/facebook-bridge:/data:z dock.mau.dev/mautrix/facebook:latest
    ```
    The `registration.yaml` file is mounted on the `homeserver` cotainer.
      
      
<br/>


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
    homeserver:
      url: "http://homeserver:8008"
      domain: "matrix.ms.local"

    webhookBot:
      localpart: "webhooks"
      appearance:
        displayName: "Webhook Bridge"
        avatarUrl: "http://i.imgur.com/IDOBtEJ.png"

    provisioning:
      secret: 'CHANGE_ME'

    web:
      hookUrlBase: 'https://webhooks.ms.local'

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

4. Run the command to check for errors:
    ```
    sudo docker run --rm -v ${CONF_PATH}/webhooks:/data turt2live/matrix-appservice-webhooks
    ```
    _If you get an `[ERROR] ConnectionError: request failed: getaddrinfo ENOTFOUND homeserver homeserver:8008`, this is normal since we don't have a working homeserver yet._

<br/>

### Maubot Manager
_Source_: https://docs.mau.fi/maubot/usage/setup/docker.html

1. Run the command to generate a `config.yaml`: 
    ```
    sudo docker run --rm -v ${CONF_PATH}/maubot:/data:z dock.mau.dev/maubot/maubot:latest
    ```
 

2. Update the file to add your homeserver:
    ```
    homeservers:
      matrix.ms.local
          url: https://homeserver:8448
          secret: <THE registration_shared_secret FROM homeserver.yaml>
    ```

3. Create an admin user
    ```
    admins:
      root: ''
      admin: '12345' #use a password you like
    ```
4. Save the file


<br/>

### Registering the new services to the home server:

Edit `homeserver.yaml` and add the following:
```
app_service_config_files:
  - /app_services/telegram-registration.yaml
  - /app_services/facebook-registration.yaml
  - /app_services/webhooks-registration.yaml
```
(in the docker-compose file we have mounted each file in the `homeserver` container)

<br/>


<br/>

<br/>

# Bringing up the Chat Server

If everything is correctly initialized we can bring up the stack with `sudo docker-compose up`. <br/>
After a while we should be able to visit the web element UI at `https://webchat.${DOMAIN}`, and register a new user.

<br/>

<br/>

# Final Notes

- This is by __no means__ a production ready setup. Some of the things that should be changed are:
  - Diffrent certificates for every service (plus for the bots)
  - Postgres for the bridges databases
  - No `--serverstransport.insecureskipverify=true` in traefik commands
  - Use `secrets` for sensitive information
- There are some more things to setup for the homeserver, bots and bridges. Please refer to their respective documentations.


# Disclaimer

It goes without saying that I'm not responsible for anything that might go wrong. __BUT__ I will be more than happy to help in any situation. If you have any suggestions on how this guide can be better (I'm sure there are a lot), please feel free to contact me!

# Sources and links

- Synapse
  - Github: @matrix-org | https://github.com/matrix-org/synapse
  - Documentation: https://matrix-org.github.io/synapse/latest/
  - Docker image: https://hub.docker.com/r/matrixdotorg/synapse/
- Postgres
  - Github: @postgres | https://github.com/postgres/postgres
  - Documentation: https://www.postgresql.org/docs/current/
  - Docker image: https://hub.docker.com/_/postgres/
- Element.io Web
  - Github: @vector-im
  - Docker image: https://hub.docker.com/r/vectorim/element-web/
- Synapse Admin
  - Github: @Awesome-Technologies | https://github.com/Awesome-Technologies/synapse-admin
  - Docker image: https://hub.docker.com/r/awesometechnologies/synapse-admin
- Telegram Bridge
  - Github: @mautrix | https://github.com/mautrix/telegram
  - Documentation: https://docs.mau.fi/bridges/python/setup/docker.html?bridge=telegram
- Facebook Bridge
  - Github: @mautrix | https://github.com/mautrix/facebook
  - Documentation: https://docs.mau.fi/bridges/python/setup/docker.html?bridge=facebook
- Maubot Manager
  - Github: @maubot | https://github.com/maubot/maubot
  - Documentation: https://docs.mau.fi/maubot/usage/setup/docker.html
- Webhook Appservice
  - Github: @turt2live | https://github.com/turt2live/matrix-appservice-webhooks
