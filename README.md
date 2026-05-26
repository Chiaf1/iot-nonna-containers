# iot-nonna-container

Deploy completo del progetto **iot-nonna** tramite Docker Compose. Include tutti i servizi necessari: broker MQTT, database PostgreSQL, servizio di ingest, API core e frontend.

---

## Servizi

| Servizio   | Immagine                          | Porta esposta |
|------------|-----------------------------------|---------------|
| `mqtt`     | `eclipse-mosquitto:2.1.2-alpine`  | `1883`        |
| `pg_db`    | `postgres:18.2`                   | `5432`        |
| `ingest`   | `chiaf1/iot-nonna-ingest:1`       | —             |
| `core`     | `chiaf1/iot-nonna-core:1`         | `3030`        |
| `frontend` | `chiaf1/iot-nonna-frontend:1`     | `3000`        |

Una volta avviato, il frontend è raggiungibile su [http://localhost:3000](http://localhost:3000) e l'API core su [http://localhost:3030](http://localhost:3030).

---

## Struttura delle configurazioni

```
.
└── configs/
    ├── core/
    │   ├── config.example.yaml
    │   └── config.yaml          ← da creare
    ├── ingest/
    │   ├── config.example.yaml
    │   └── config.yaml          ← da creare
    ├── mosquitto/
    │   └── mosquitto.conf
    └── postgres/
        ├── .env.example
        └── .env                 ← da creare
```

---

## Setup

### 1. Clona la repository

```bash
git clone https://github.com/<tuo-utente>/iot-nonna-container.git
cd iot-nonna-container
```

### 2. Configura i servizi

Ogni servizio ha bisogno del proprio file di configurazione. I file `.example` contengono tutte le impostazioni disponibili con i valori di default: copiali, rinominali e personalizzali secondo le tue esigenze.

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

Edita i file appena creati con le impostazioni che preferisci.

### 3. Configura Mosquitto

Il file `configs/mosquitto/mosquitto.conf` è già presente nella repo. Per una demo rapida senza gestione degli utenti, assicurati che contenga la riga:

```
allow_anonymous true
```

> **Nota:** Se preferisci usare autenticazione con username e password, dovrai creare il file delle password all'interno del container del broker tramite bind mount e `mosquitto_passwd`. Per una demo locale è più pratico abilitare l'accesso anonimo.

### 4. Avvia i servizi

```bash
docker compose up -d
```

Docker scaricherà le immagini necessarie al primo avvio. Per verificare che tutti i container siano in esecuzione:

```bash
docker compose ps
```

Per visualizzare i log in tempo reale:

```bash
docker compose logs -f
```

---

## Fermare i servizi

```bash
docker compose down
```

Per rimuovere anche i volumi (database e dati MQTT):

```bash
docker compose down -v
```

---

## Note

- Il servizio `core` esegue automaticamente le migration del database e il seeding all'avvio (variabili `RUN_MIGRATIONS=true` e `RUN_SEEDING=true`).
- Il `frontend` comunica con il `core` tramite rete Docker interna (`http://core:3030`) — non è necessario esporre ulteriori porte per la comunicazione tra servizi.
- I dati di PostgreSQL e MQTT vengono persistiti in volumi Docker (`pgdata`, `mqtt-data`, `mqtt-log`) e sopravvivono ai riavvii dei container.