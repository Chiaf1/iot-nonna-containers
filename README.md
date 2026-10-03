# iot-nonna-containers

Full deployment of the **iot-nonna** project with Docker Compose. It includes everything the system needs: MQTT broker, PostgreSQL database, ingest service, core API and frontend.

---

## What is iot-nonna

A home IoT system: ESP32 sensors publish temperature and humidity over MQTT, `iot-nonna-ingest` writes the data to PostgreSQL, `iot-nonna-core` exposes it through a REST API and `iot-nonna-frontend` shows it in a Next.js dashboard.

**Status:** home prototype, meant for a local network. The API has no authentication and MQTT uses plain TCP.

Project repositories:

- [iot-nonna-ingest](https://github.com/Chiaf1/iot-nonna-ingest) — subscribes to the MQTT topics and writes the readings to the database
- [iot-nonna-core](https://github.com/Chiaf1/iot-nonna-core) — REST API, migrations and database seeding
- [iot-nonna-frontend](https://github.com/Chiaf1/iot-nonna-frontend) — web dashboard

### Architecture

```
[ESP32 sensors]
      │ MQTT (tcp, 1883)
      ▼
[mqtt: Mosquitto]
      │
      ▼
[ingest] ──── writes readings ────▶ [pg_db: PostgreSQL]
                                            ▲
[core: REST API, :3030] ──── reads/writes ──┘
      ▲
      │ HTTP
[frontend: Next.js, :3000]
```

### Limitations

- No authentication on the `core` API (and therefore none on the frontend either).
- Plain MQTT (TCP, no TLS).
- Ports `1883`, `5432`, `3030` and `3000` are published on the host.
- Use only on a trusted local network. Do not expose it to the Internet.

---

## Services

| Service    | Image                            | Exposed port |
| ---------- | -------------------------------- | ------------ |
| `mqtt`     | `eclipse-mosquitto:2.1.2-alpine` | `1883`       |
| `pg_db`    | `postgres:18.2`                  | `5432`       |
| `ingest`   | `chiaf1/iot-nonna-ingest:1`      | —            |
| `core`     | `chiaf1/iot-nonna-core:1`        | `3030`       |
| `frontend` | `chiaf1/iot-nonna-frontend:1`    | `3000`       |

Once it is running, the frontend is available at [http://localhost:3000](http://localhost:3000) and the core API at [http://localhost:3030](http://localhost:3030).

---

## Configuration layout

```
.
└── configs/
    ├── core/
    │   ├── config.example.yaml
    │   └── config.yaml          ← to be created
    ├── ingest/
    │   ├── config.example.yaml
    │   └── config.yaml          ← to be created
    ├── mosquitto/
    │   └── mosquitto.conf
    └── postgres/
        ├── .env.example
        └── .env                 ← to be created
```

---

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Chiaf1/iot-nonna-containers.git
cd iot-nonna-containers
```

### 2. Configure the services

Each service needs its own configuration file. The `.example` files contain the available settings with sample values: copy them, rename them and adjust them as you like.

**Core:**

```bash
cp configs/core/config.example.yaml configs/core/config.yaml
```

**Ingest:**

```bash
cp configs/ingest/config.example.yaml configs/ingest/config.yaml
```

**PostgreSQL:**

```bash
cp configs/postgres/.env.example configs/postgres/.env
```

The examples use the Compose service names (`pg_db` and `mqtt`) instead of `localhost`, because the containers talk to each other over the Docker network. The database user, password and name are the same in the Postgres `.env` file and in the `DbURL` of core and ingest:

```yaml
# configs/core/config.yaml and configs/ingest/config.yaml
DB:
  DbURL: postgres://user:password@pg_db:5432/mydb?sslmode=disable
```

```yaml
# configs/ingest/config.yaml
MQTT:
  broker: tcp://mqtt:1883
```

```env
# configs/postgres/.env
POSTGRES_USER=user
POSTGRES_PASSWORD=password
POSTGRES_DB=mydb
```

If you change the database user, password or name, update all three files so they stay consistent. Edit the files you just created with the settings you prefer.

### 3. Configure Mosquitto

The file `configs/mosquitto/mosquitto.conf` is already in the repo. For a quick demo without user management, the examples use anonymous access:

```
allow_anonymous yes
```

#### Optional: enable username and password

If you prefer username and password authentication, set this in `mosquitto.conf`:

```
allow_anonymous false
password_file /mosquitto/config/passwordfile
```

Then create the password file with `mosquitto_passwd`, for example through the `eclipse-mosquitto` container (from the root of the repo). The command asks for the password interactively; `-c` creates the file (and overwrites it if it already exists):

```bash
docker run --rm -it \
  -v "$(pwd)/configs/mosquitto:/mosquitto/config" \
  eclipse-mosquitto:2.1.2-alpine \
  mosquitto_passwd -c /mosquitto/config/passwordfile USERNAME
```

To add more users to the same file, leave out `-c`. Finally, set the same credentials in `configs/ingest/config.yaml` (the `username` and `password` lines under `MQTT`, commented out in the example) and restart the broker and ingest.

References:

- [mosquitto_passwd](https://mosquitto.org/man/mosquitto_passwd-1.html)
- [mosquitto.conf (`password_file`, `allow_anonymous`)](https://mosquitto.org/man/mosquitto-conf-5.html)

> **Note:** even with a password, the credentials still travel in clear text over TCP, because the listener does not use TLS. The "trusted local network only" limitation still applies.

### 4. Start the services

```bash
docker compose up -d
```

Docker will download the required images on the first run. To check that all containers are running:

```bash
docker compose ps
```

To follow the logs in real time:

```bash
docker compose logs -f
```

#### Start order

`core` must have created the database schema before `ingest` starts: at startup `ingest` queries the `mqtt_topic_list_metadata` view (created by the `core` migrations) and exits with an error if the query fails. In `docker-compose.yaml`, `ingest` depends only on `mqtt` and `pg_db`, not on `core`, so on the first run it can start before the migrations have finished. The services use `restart: unless-stopped`, so Docker restarts `ingest`; if that does not happen, restart it by hand once `core` is up:

```bash
docker compose restart ingest
```

---

## Stopping the services

```bash
docker compose down
```

To also remove the volumes (database and MQTT data):

```bash
docker compose down -v
```

---

## Notes

- The `core` service runs the database migrations and seeding automatically at startup (variables `RUN_MIGRATIONS=true` and `RUN_SEEDING=true`).
- The `frontend` talks to `core` over the internal Docker network (`http://core:3030`, variable `API_URL`), so no extra ports need to be exposed for communication between services.
- PostgreSQL and MQTT data are stored in Docker volumes (`pgdata`, `mqtt-data`, `mqtt-log`) and survive container restarts.
