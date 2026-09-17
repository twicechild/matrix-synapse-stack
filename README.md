# A Matrix (Synapse) Stack with Traefik, bots, bridges and more

This is a stack in a single `docker-compose.yaml` file. The guide starts by preconfiguring the various services and finally bringing the stack up.

The stack follows some specific logic concerning the file organization and a couple "bad practices" (exposing ports and folders) that should not be a problem for a non production environment.

# Components (and images used)

- Postgres - `postgres:18.6`
- Synapse homeserver - `matrixdotorg/synapse:v1.161.0`
- Element Web Client - `vectorim/element-web:v1.12.28`
- Synapse Admin - `awesometechnologies/synapse-admin:0.11.4`
- Traefik proxy - `traefik:v3.7.13`
- Telegram Bridge - `dock.mau.dev/mautrix/telegram:v0.2609.0`
- Instagram Bridge - `dock.mau.dev/mautrix/meta:ig-v0.2609.0`
- Messenger Bridge - `dock.mau.dev/mautrix/meta:v0.2609.0`
- Hookshot (webhooks and integrations) - `ghcr.io/matrix-org/matrix-hookshot:7.4.4`
- Maubot bot manager - `dock.mau.dev/maubot/maubot:v0.6.0`

Everything is pinned on purpose. When you want to upgrade, bump the tag yourself — `latest` tags are how stacks rot.

<br/>

# Assumptions

<br/>

## Domain and subdomains

You should have a locally (at least) resolved domain (During the instructions we will use `ms.local`). We also use the following subdomains at various points:
- matrix.ms.local
- webchat.ms.local
- admin.ms.local
- webhooks.ms.local
- maubot.ms.local
- proxy.ms.local

<br/>

## Certificates

The guide assumes you have a wildcard certificate for your domain name (`WILDCARD.ms.local`) in `CERT_PATH` folder.
```
/${CERT_PATH}/
    WILDCARD.ms.local.crt
    WILDCARD.ms.local.key
```

You can generate a self-signed certificate following the guide from @cecilemuller:
https://gist.github.com/cecilemuller/9492b848eb8fe46d462abeb26656c4f8

You can of course use different certificates for every service.

<br/>

## Folder hierarchy

The docker-compose.yaml file assumes the following hierarchy:
```
${CONF_PATH}/
      homeserver/
      webchat/
      telegram-bridge/
      instagram-bridge/
      messenger-bridge/
      hookshot/
      maubot/
      proxy/
${DATA_PATH}/
      homeserver-media-store/
${CERT_PATH}/
```
- `${CONF_PATH}` : configuration and persistent data

- `${CERT_PATH}` : certificates

- `${DATA_PATH}` : other kind of persistent data (like synapse media store etc.)

Create them before anything else:
```
mkdir -p ${CONF_PATH}/{homeserver,webchat,telegram-bridge,instagram-bridge,messenger-bridge,hookshot,maubot,proxy} ${DATA_PATH}/homeserver-media-store ${CERT_PATH}
```
Create them as your own user. If you let Docker create missing folders on first mount, it makes them root-owned — and the services run as non-root users, so they won't be able to write to them.

<br/>

<br/>

# Initialization and preconfigurations

## Expose ENV

Copy the example env files and edit them to your liking:
```
cp .env.example .env
cp db.env.example db.env
cp synapse.env.example synapse.env
```

Then expose each ENV with `export VAR=VAL`. You will need:
```
export DOMAIN=ms.local
export CONF_PATH=/mnt/configs
```
(These exports are for the `docker run` setup commands below — `docker compose` reads `.env` on its own.)

Also edit `db.env` and change `POSTGRES_PASSWORD` (please). And `synapse.env` if you want different stats reporting.

Some of the services need to initialize some config files before you can finally start them.

<br/>

## Proxy

1. Create file `traefik-ssl.toml` in `${CONF_PATH}/proxy/` and paste the following (there is also a sample in `sample_configs/proxy/`):
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

## Postgres

Nothing to configure here — the compose file creates the volume for you. Just the password in `db.env`.

Start only the database now, we will need it for the bridge setups below:
```
sudo docker compose up -d db
```

<br/>

## Synapse

Generate a `homeserver.yaml` file in `${CONF_PATH}/homeserver/`. You can find a sample config at `sample_configs/homeserver/homeserver.yaml`

__IMPORTANT: the subdomain (`matrix.${DOMAIN}`) CANNOT be changed later. Make sure you have decided correctly.__

```
sudo docker run -it --rm \
    -v=${CONF_PATH}/homeserver:/data \
    -e SYNAPSE_SERVER_NAME=matrix.${DOMAIN} \
    -e SYNAPSE_REPORT_STATS=no \
    matrixdotorg/synapse:v1.161.0 generate
```

__IMPORTANT 2: do not skip the generate step.__ It also creates the log config and the signing key. If the log config file is missing, synapse falls back to its built-in default, which tries to write `/homeserver.log` at the root of the container. The image runs as UID 991 (not root), so it will crash-loop with `PermissionError: [Errno 13] Permission denied: '/homeserver.log'` — and no amount of `chmod 777` on your host folders will fix it, because the crash happens before any mounted folder is touched.

Edit/Uncomment some important fields:

-  `server_name` will be autofilled
```
server_name: "matrix.ms.local"
```

- One plain HTTP listener is enough — Traefik unwraps TLS for us. Note we keep `federation` in the resources, it goes through the same listener.
```
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    bind_addresses: ['0.0.0.0']
    resources:
      - names: [client, federation]
        compress: false
```

- Add the postgres info to connect to `db` container (same password as `db.env`)
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

- Change the default `media_store` path to the one that will be mounted in `docker-compose.yaml`
```
media_store_path: "/media_store"
```

- Enable registrations (and set a shared secret — maubot will use it to register bot accounts)
```
enable_registration: true
enable_registration_without_verification: true
registration_shared_secret: "<generate one, e.g. openssl rand -base64 32>"
```
(Recent Synapse versions refuse to start with open registration unless you explicitly allow it without verification — fine for a lab stack, use tokens or captcha for anything exposed.)

- Enable user directory search. (This will help us find the bot accounts later)
```
user_directory:
    enabled: true
    search_all_users: true
    prefer_local_users: true
```

- Save the file (_We will edit more while configuring the Bridges and Bots_)

<br/>

## Element Web

Copy `sample_configs/webchat/config.json` to `${CONF_PATH}/webchat/config.json` and point it to your homeserver. Without this, Element defaults to matrix.org and you'd have to change the homeserver by hand on the login screen — the old guide never mentioned this, sorry about that.

<br/>

<br/>

## Bridges and Bots

<br/>

### Telegram Bridge
_Source_: https://docs.mau.fi/bridges/general/docker-setup.html?bridge=telegram

Heads up: the bridge was rewritten in Go since the original version of this guide (they call it "bridgev2"). The config format is completely different from the old Python one — if you're upgrading an old setup, start from a fresh config, don't try to patch the old file.

1. Run the command to generate a `config.yaml`:
    ```
    sudo docker run --rm -v ${CONF_PATH}/telegram-bridge:/data:z dock.mau.dev/mautrix/telegram:v0.2609.0
    ```

2. Edit the file (reference `sample_configs/telegram-bridge/config.yaml`):
    - Main connection configurations (_Since everything lives in the same compose network, the bridge talks to the homeserver over plain HTTP by container name. No certificates to verify between them — same goes for the other bridges and services_)
        ```
        network:
            # Get your own API keys at https://my.telegram.org/apps
            api_id: 12345
            api_hash: tjyd5yge35lbodk1xwzw2jstp90k55qz

        database:
            type: postgres
            uri: postgres://synapse:<SAME AS db.env>@db/telegram_bridge?sslmode=disable

        homeserver:
            address: http://homeserver:8008
            domain: matrix.ms.local
            software: standard

        appservice:
            address: http://telegram-bridge:29317
            hostname: 0.0.0.0
            port: 29317
        ```
        (`hostname` must be `0.0.0.0` in Docker, not localhost)

    - Bridge permissions

      We should also give permission to some users to use the bridge. Since we don't even have a homeserver yet we will give full access to all users that share the domain `matrix.ms.local`. Edit the following:
      ```
      bridge:
          permissions:
              "*": relay
              "matrix.ms.local": user
      ```

3. Create the bridge database on the shared postgres:
    ```
    sudo docker exec -it db psql -U synapse -d synapse_db -c "CREATE DATABASE telegram_bridge OWNER synapse;"
    ```

4. Run the docker command again to generate a `registration.yaml`
    ```
    sudo docker run --rm -v ${CONF_PATH}/telegram-bridge:/data:z dock.mau.dev/mautrix/telegram:v0.2609.0
    ```
      The `registration.yaml` file is mounted on the `homeserver` container. The bridge images write it as root, so make it readable:
      ```
      sudo chmod 644 ${CONF_PATH}/telegram-bridge/registration.yaml
      ```

<br/>

### Instagram Bridge
_Source_: https://docs.mau.fi/bridges/general/docker-setup.html?bridge=meta

This replaces the old facebook bridge, which was archived upstream in 2024. The new bridge (`mautrix-meta`) covers Instagram DMs and Facebook Messenger — but only one mode per instance, and the mode is baked into the image tag: `ig-v0.2609.0` is Instagram, a plain tag like `v0.2609.0` is Messenger. We set up both below; if you only want one, just skip the other.

The setup is almost identical to the Telegram bridge:

1. Run the command to generate a `config.yaml`:
    ```
    sudo docker run --rm -v ${CONF_PATH}/instagram-bridge:/data:z dock.mau.dev/mautrix/meta:ig-v0.2609.0
    ```

2. Edit the file (reference `sample_configs/instagram-bridge/config.yaml`):
    - Main connection configurations (_same logic as the Telegram bridge_)
        ```
        database:
            type: postgres
            uri: postgres://synapse:<SAME AS db.env>@db/instagram_bridge?sslmode=disable

        homeserver:
            address: http://homeserver:8008
            domain: matrix.ms.local
            software: standard

        appservice:
            address: http://instagram-bridge:29330
            hostname: 0.0.0.0
            port: 29330
        ```

    - Bridge permissions
      ```
      bridge:
          permissions:
              "*": relay
              "matrix.ms.local": user
      ```

3. Create the bridge database:
    ```
    sudo docker exec -it db psql -U synapse -d synapse_db -c "CREATE DATABASE instagram_bridge OWNER synapse;"
    ```

4. Run the docker command again to generate a `registration.yaml`
    ```
    sudo docker run --rm -v ${CONF_PATH}/instagram-bridge:/data:z dock.mau.dev/mautrix/meta:ig-v0.2609.0
    ```
      The `registration.yaml` file is mounted on the `homeserver` container. The bridge images write it as root, so make it readable:
      ```
      sudo chmod 644 ${CONF_PATH}/instagram-bridge/registration.yaml
      ```

<br/>

### Messenger Bridge (almost identical to Instagram bridge)
_Source_: https://docs.mau.fi/bridges/general/docker-setup.html?bridge=meta

Same bridge, Messenger mode — hence the plain image tag. Second instance, own config, own registration, own database:

1. Run the command to generate a `config.yaml`:
    ```
    sudo docker run --rm -v ${CONF_PATH}/messenger-bridge:/data:z dock.mau.dev/mautrix/meta:v0.2609.0
    ```

2. Edit the file (reference `sample_configs/messenger-bridge/config.yaml`):
    - Main connection configurations (_same logic as the Instagram bridge, different port and IDs_)
        ```
        database:
            type: postgres
            uri: postgres://synapse:<SAME AS db.env>@db/messenger_bridge?sslmode=disable

        homeserver:
            address: http://homeserver:8008
            domain: matrix.ms.local
            software: standard

        appservice:
            address: http://messenger-bridge:29319
            hostname: 0.0.0.0
            port: 29319
        ```

    - Bridge permissions
      ```
      bridge:
          permissions:
              "*": relay
              "matrix.ms.local": user
      ```

3. Create the bridge database:
    ```
    sudo docker exec -it db psql -U synapse -d synapse_db -c "CREATE DATABASE messenger_bridge OWNER synapse;"
    ```

4. Run the docker command again to generate a `registration.yaml`
    ```
    sudo docker run --rm -v ${CONF_PATH}/messenger-bridge:/data:z dock.mau.dev/mautrix/meta:v0.2609.0
    ```
      The `registration.yaml` file is mounted on the `homeserver` container. The bridge images write it as root, so make it readable:
      ```
      sudo chmod 644 ${CONF_PATH}/messenger-bridge/registration.yaml
      ```

<br/>

### Hookshot (webhooks and integrations)
_Source_: https://matrix-org.github.io/matrix-hookshot/latest/setup.html

This replaces the old webhook appservice, which its own author described as "planned obsolescence" (fair enough). Hookshot does generic webhooks like the old one, and also GitHub, GitLab, Jira, RSS feeds and more — enable whatever you need in its config.

1. Generate a passkey (hookshot uses it to encrypt its tokens):
    ```
    openssl genpkey -out ${CONF_PATH}/hookshot/passkey.pem -outform PEM -algorithm RSA -pkeyopt rsa_keygen_bits:4096
    ```

2. Create a `config.yml` file in `${CONF_PATH}/hookshot/` (reference `sample_configs/hookshot/config.yml`). The important bits:
    ```
    bridge:
      domain: matrix.ms.local
      url: http://homeserver:8008
      port: 9993
      bindAddress: 0.0.0.0

    passFile: /data/passkey.pem

    listeners:
      - port: 9000
        bindAddress: 0.0.0.0
        resources:
          - webhooks

    permissions:
      - actor: matrix.ms.local
        services:
          - service: "*"
            level: admin

    generic:
      enabled: true
      urlPrefix: https://webhooks.ms.local/
      userIdPrefix: _webhooks_
    ```

3. Create a `registration.yml` file in `${CONF_PATH}/hookshot/` (reference `sample_configs/hookshot/registration.yml`). Make sure you generate `hs_token` and `as_token` (e.g. `openssl rand -hex 32`):
    ```
    id: matrix-hookshot
    as_token: A_RANDOM_ALPHANUMERIC_STRING  # CHANGE THIS
    hs_token: ANOTHER_RANDOM_ALPHANUMERIC_STRING  # CHANGE THIS
    namespaces:
      rooms: []
      users:
        - regex: "@_webhooks_.*:matrix.ms.local"
          exclusive: true
      aliases: []
    sender_localpart: hookshot
    url: 'http://hookshot:9993'
    rate_limited: false
    receive_ephemeral: true
    ```
    The `registration.yml` file is mounted on the `homeserver` container. Same as the bridges — make it readable:
    ```
    sudo chmod 644 ${CONF_PATH}/hookshot/registration.yml ${CONF_PATH}/hookshot/passkey.pem
    ```

<br/>

### Maubot Manager
_Source_: https://docs.mau.fi/maubot/usage/setup/docker.html

1. Run the command to generate a `config.yaml`:
    ```
    sudo docker run --rm -v ${CONF_PATH}/maubot:/data:z dock.mau.dev/maubot/maubot:v0.6.0
    ```

2. Update the file to add your homeserver (reference `sample_configs/maubot/config.yaml`):
    ```
    homeservers:
      matrix.ms.local:
          url: http://homeserver:8008
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
  - /app_services/instagram-registration.yaml
  - /app_services/messenger-registration.yaml
  - /app_services/hookshot-registration.yml
```
(in the docker-compose file we have mounted each file in the `homeserver` container)

<br/>

<br/>

<br/>

# Bringing up the Chat Server

If everything is correctly initialized we can bring the stack up with `sudo docker compose up -d`. <br/>
The bridges and bots wait for the homeserver to pass its health check before they start, so don't panic if they take a minute.

After a while we should be able to visit the web element UI at `https://webchat.${DOMAIN}`, and register a new user.

The admin UI lives at `https://admin.${DOMAIN}` — log in with your (admin) user there.

To use the bridges, open a DM with the bridge bot (`@telegrambot:matrix.ms.local`, `@instagrambot:matrix.ms.local`, `@messengerbot:matrix.ms.local`) and send `help`. For hookshot, invite `@hookshot:matrix.ms.local` to a room.

<br/>

<br/>

# Final Notes

- This is by __no means__ a production ready setup. Some of the things that should be changed are:
  - Real certificates (Let's Encrypt instead of self-signed)
  - `enable_registration: false` once your users exist
  - Actually strong passwords everywhere — the ones in this repo are placeholders
  - Restrict or disable the Traefik dashboard
  - Backups for `${CONF_PATH}`, `${DATA_PATH}` and the `db-data` volume. The signing key in `${CONF_PATH}/homeserver/` is irreplaceable — lose it and your server identity is gone
- Upgrading is just bumping the image tags in `docker-compose.yaml` and then `docker compose pull && docker compose up -d`. Check the release notes of each component first — the bridges in particular sometimes change config format between versions.
- There are some more things to setup for the homeserver, bots and bridges. Please refer to their respective documentations.


# Disclaimer

It goes without saying that I'm not responsible for anything that might go wrong. __BUT__ I will be more than happy to help in any situation. If you have any suggestions on how this guide can be better (I'm sure there are a lot), please feel free to open an issue or a PR!

# Sources and links

- Synapse
  - Github: @element-hq | https://github.com/element-hq/synapse
  - Documentation: https://matrix-org.github.io/synapse/latest/
  - Docker image: https://hub.docker.com/r/matrixdotorg/synapse/
- Postgres
  - Github: @postgres | https://github.com/postgres/postgres
  - Documentation: https://www.postgresql.org/docs/current/
  - Docker image: https://hub.docker.com/_/postgres/
- Element.io Web
  - Github: @element-hq | https://github.com/element-hq/element-web
  - Docker image: https://hub.docker.com/r/vectorim/element-web/
- Synapse Admin
  - Github: @Awesome-Technologies | https://github.com/Awesome-Technologies/synapse-admin
  - Docker image: https://hub.docker.com/r/awesometechnologies/synapse-admin
- Traefik
  - Github: @traefik | https://github.com/traefik/traefik
  - Documentation: https://doc.traefik.io/traefik/
- Telegram Bridge
  - Github: @mautrix | https://github.com/mautrix/telegram
  - Documentation: https://docs.mau.fi/bridges/general/docker-setup.html?bridge=telegram
- Instagram Bridge
  - Github: @mautrix | https://github.com/mautrix/meta
  - Documentation: https://docs.mau.fi/bridges/general/docker-setup.html?bridge=meta
- Messenger Bridge
  - Github: @mautrix | https://github.com/mautrix/meta
  - Documentation: https://docs.mau.fi/bridges/general/docker-setup.html?bridge=meta
- Hookshot
  - Github: @matrix-org | https://github.com/matrix-org/matrix-hookshot
  - Documentation: https://matrix-org.github.io/matrix-hookshot/latest/setup.html
- Maubot Manager
  - Github: @maubot | https://github.com/maubot/maubot
  - Documentation: https://docs.mau.fi/maubot/usage/setup/docker.html