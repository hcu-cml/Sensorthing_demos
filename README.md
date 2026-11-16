# IoT Server for Geospatial Data — OGC SensorThings API with FROST-Server

This repository sets up a complete **OGC SensorThings API** server on your own computer with Docker.
You get a running **FROST-Server** that stores sensor data with real geometries in **PostgreSQL/PostGIS**, plus two optional tools for ingesting data from devices: **Mosquitto** (MQTT broker) and **Node-RED** (low-code data flows).

> Course: Project Session — *OGC SensorThings API* · Lecturer: Badreddine Ouzougarh

---

## Contents

1. [Architecture](#1-architecture)
2. [Requirements](#2-requirements)
3. [Repository layout](#3-repository-layout)
4. [⚠ Check your ports first](#4--check-your-ports-first)
5. [Set up the environment step by step](#5-set-up-the-environment-step-by-step)
   - [5.1 Install Docker](#51-install-docker)
   - [5.2 Run PostgreSQL + PostGIS](#52-run-postgresql--postgis)
   - [5.3 Run FROST-Server](#53-run-frost-server)
   - [5.4 Stop, restart and clean up](#54-stop-restart-and-clean-up)
6. [Quick start with Docker Compose](#6-quick-start-with-docker-compose)
7. [The docker-compose.yml explained](#7-the-docker-composeyml-explained)
8. [The SensorThings data model](#8-the-sensorthings-data-model)
9. [Create your first data](#9-create-your-first-data)
10. [Query the data](#10-query-the-data)
11. [Optional: Mosquitto + Node-RED](#11-optional-mosquitto--node-red)
12. [View the data in QGIS](#12-view-the-data-in-qgis)
13. [Troubleshooting](#13-troubleshooting)
14. [References](#14-references)

> **Two ways to install.** Section 5 builds the stack by hand, one component at a time, so you see what each part does. Section 6 starts the same stack with a single command. Use **one or the other**, not both at the same time: they use the same container names and ports.

---

## 1. Architecture

```
 ┌──────────┐   MQTT   ┌────────────┐        ┌───────────┐  HTTP/MQTT  ┌──────────────┐  JDBC  ┌──────────┐
 │ Sensors  │ ───────▶ │ Mosquitto  │ ─────▶ │ Node-RED  │ ──────────▶ │ FROST-Server │ ◀────▶ │ PostGIS  │
 │ ESP32,RPi│          │ (optional) │        │ (optional)│             │ :8080 :1883  │        │  :5432   │
 └──────────┘          └────────────┘        └───────────┘             └──────┬───────┘        └──────────┘
                                                                               │ REST / MQTT
                                                         ┌─────────────────────▼──────────────────────┐
                                                         │ Clients: QGIS · Leaflet/OpenLayers · Python │
                                                         └─────────────────────────────────────────────┘
```

| Service | Role | Required | Default port |
|---|---|---|---|
| **FROST-Server** | Implements the SensorThings API (REST + built-in MQTT) | ✅ | 8080 (HTTP), 1883 (MQTT) |
| **PostgreSQL + PostGIS** | Stores all entities; locations as geometries | ✅ | 5432 |
| **Mosquitto** | Stand-alone MQTT broker for field devices | optional | 1884 on the host (see §4) |
| **Node-RED** | Converts device messages into SensorThings Observations | optional | 1880 |

The minimal setup is FROST + PostGIS. Clients can write to FROST directly over HTTP, so Mosquitto and Node-RED are only needed when real devices send data in their own format.

---

## 2. Requirements

- **Docker Desktop** (Windows / macOS) or **Docker Engine + Docker Compose plugin** (Linux)
- About 2 GB of free disk space for the images
- A tool for HTTP requests: your browser (GET), plus **curl**, **Postman** or **Bruno** (POST)
- Optional: **QGIS 3.36+**, **MQTT Explorer**

Check your installation:

```bash
docker --version
docker compose version
```

> **Windows users:** run the `curl` examples in **Git Bash** or WSL. PowerShell treats quotes differently, so use Postman there if a command fails.

---

## 3. Repository layout

```
parking_pykci/
├── docker-compose.yml        # all services
├── .env.example              # ports and passwords → copy to .env
├── mosquitto/
│   └── config/mosquitto.conf # broker configuration (optional service)
├── data/
│   ├── station.json          # example Thing with Location + Datastream
│   └── observation.json      # example Observation
└── README.md
```

---

## 4. ⚠ Check your ports first

Docker publishes each container port on your computer. If a port is already in use, the container won't start, or you end up talking to a different program. **Choose your ports before you start any container.**

| Service | Default | Typical clash | Change to |
|---|---|---|---|
| PostgreSQL | 5432 | PostgreSQL already installed on your laptop | 5433 |
| FROST HTTP | 8080 | GeoServer, Tomcat, another web app | 8081 |
| FROST MQTT | 1883 | Mosquitto | keep 1883 (Mosquitto uses 1884, see §11) |
| Node-RED | 1880 | another Node-RED | 1881 |

Find out what is using a port:

```bash
# Windows
netstat -ano | findstr :5432
# macOS / Linux
lsof -i :5432
```

**Manual setup (§5):** change the left number of each `-p` option, e.g. `-p 5433:5432`.

**Docker Compose (§6):** set the ports in `.env`:

```dotenv
DB_HOST_PORT=5433
FROST_HTTP_PORT=8081
FROST_MQTT_PORT=1883
MOSQUITTO_HOST_PORT=1884
NODERED_HOST_PORT=1880
POSTGRES_PASSWORD=ChangeMe
```

**Three rules:**

1. A mapping is `HOST:CONTAINER`. **Change only the left number.**
2. If you change the FROST HTTP port, `serviceRootUrl` must use the same port. The compose file (§7) builds it from `FROST_HTTP_PORT` automatically. Without that, the links in API responses point to the wrong port.
3. **Do not change** `database:5432` in `persistence_db_url`. FROST reaches the database by its service name **inside Docker's network**, whatever port you published on the host.

---

## 5. Set up the environment step by step

You install three components, in this order:

| Step | Component | Why this order |
|---|---|---|
| 5.1 | **Docker** | Runs the other two as containers |
| 5.2 | **PostgreSQL + PostGIS** | FROST needs a database to store its data |
| 5.3 | **FROST-Server** | Connects to the database and serves the API |

Each step ends with a check. Don't move on until the check passes.

### 5.1 Install Docker

**Windows 10 / 11**

1. Docker Desktop needs **WSL 2**. Open PowerShell **as administrator** and run:
   ```powershell
   wsl --install
   ```
   Restart the computer when asked.
2. Download **Docker Desktop for Windows** from https://www.docker.com/products/docker-desktop/ and run the installer. Keep the option **Use WSL 2 instead of Hyper-V** ticked.
3. Start Docker Desktop and wait until it shows **Engine running**.

**macOS**

1. Download **Docker Desktop for Mac** from https://www.docker.com/products/docker-desktop/. Choose **Apple silicon** (M1–M4) or **Intel** to match your Mac (Apple menu → About This Mac).
2. Drag Docker into Applications, start it and wait until it shows **Engine running**.

**Linux (Ubuntu / Debian)**

```bash
curl -fsSL https://get.docker.com | sudo sh   # installs Docker Engine + Compose plugin
sudo usermod -aG docker $USER                 # use docker without sudo
```

Log out and back in so the group change takes effect.

**✅ Check**

```bash
docker --version
docker compose version
docker run --rm hello-world
```

`hello-world` prints *Hello from Docker!* If you get *Cannot connect to the Docker daemon*, Docker Desktop is not running yet.

### 5.2 Run PostgreSQL + PostGIS

We use the official **postgis/postgis** image: PostgreSQL with the PostGIS extension already installed and enabled in the database it creates.

**1. Create a network and a volume**

```bash
docker network create sensorthings-net   # lets FROST find the database by name
docker volume create postgis_volume      # keeps the data when the container is removed
```

**2. Start the database container**

```bash
docker run -d --name database \
  --network sensorthings-net \
  -e POSTGRES_DB=sensorthings \
  -e POSTGRES_USER=sensorthings \
  -e POSTGRES_PASSWORD=ChangeMe \
  -p 5432:5432 \
  -v postgis_volume:/var/lib/postgresql/data \
  postgis/postgis:16-3.4-alpine
```

| Option | Meaning |
|---|---|
| `--name database` | Container name. FROST uses it as the host name. |
| `--network sensorthings-net` | Puts the database on the same network as FROST |
| `-e POSTGRES_DB / USER / PASSWORD` | Database, user and password created at the first start |
| `-p 5432:5432` | Publishes the port on your computer (for pgAdmin or QGIS). **Port busy? Use `-p 5433:5432`** (§4). |
| `-v postgis_volume:…` | Stores the data in the volume |

> **Windows PowerShell:** replace the `\` at the end of each line with a backtick `` ` ``, or write the whole command on one line.

**✅ Check**

```bash
docker ps                                   # "database" is listed as Up
docker logs database                        # ends with "database system is ready to accept connections"
docker exec -it database psql -U sensorthings -d sensorthings -c "SELECT postgis_full_version();"
```

The last command prints the PostGIS version. If it says *function postgis_full_version() does not exist*, enable the extension once:

```bash
docker exec -it database psql -U sensorthings -d sensorthings -c "CREATE EXTENSION IF NOT EXISTS postgis;"
```

> **Already have PostgreSQL installed on your computer?** You can use it instead of the container: install PostGIS with the Stack Builder (Windows) or your package manager, create the `sensorthings` user and database, and run `CREATE EXTENSION postgis;` in it. In step 5.3, FROST then reaches it at `host.docker.internal` instead of `database`. For the course, the container is simpler.

### 5.3 Run FROST-Server

We use the **all-in-one** image `fraunhoferiosb/frost-server`: HTTP API and MQTT in one container.

**1. Start the FROST container**

```bash
docker run -d --name web \
  --network sensorthings-net \
  -p 8080:8080 \
  -p 1883:1883 \
  -e serviceRootUrl=http://localhost:8080/FROST-Server \
  -e http_cors_enable=true \
  -e http_cors_allowed_origins=* \
  -e persistence_db_driver=org.postgresql.Driver \
  -e persistence_db_url=jdbc:postgresql://database:5432/sensorthings \
  -e persistence_db_username=sensorthings \
  -e persistence_db_password=ChangeMe \
  -e persistence_autoUpdateDatabase=true \
  fraunhoferiosb/frost-server:latest
```

| Option | Meaning |
|---|---|
| `-p 8080:8080` | HTTP API. **Port busy? Use `-p 8081:8080`** and change `serviceRootUrl` to `:8081` too. |
| `-p 1883:1883` | FROST's built-in MQTT |
| `serviceRootUrl` | The public address FROST writes into its links |
| `persistence_db_url` | `database` = the container name from 5.2. Always port **5432**, whatever you published on your computer. |
| `persistence_db_password` | Must match `POSTGRES_PASSWORD` from 5.2 |
| `persistence_autoUpdateDatabase=true` | Creates the SensorThings tables at start-up |

**2. Initialise the database (if needed)**

Open http://localhost:8080/FROST-Server/DatabaseStatus. With `persistence_autoUpdateDatabase=true` the tables already exist. If the page shows pending changes, click the update button.

**✅ Check**

```bash
docker ps                 # "database" and "web" are both Up
docker logs -f web        # no "Connection refused" errors (Ctrl+C to leave)
```

Open http://localhost:8080/FROST-Server/v1.1. You should see JSON listing `Things`, `Locations`, `Datastreams`, `Sensors`, `ObservedProperties`, `Observations`, `FeaturesOfInterest` and `HistoricalLocations`.

**Your server is running.** Continue with §9 to create your first data.

### 5.4 Stop, restart and clean up

```bash
docker stop web database          # stop (data is kept)
docker start database web         # start again: database first
docker rm -f web database         # remove the containers (data stays in the volume)
docker volume rm postgis_volume   # delete the data for good
docker network rm sensorthings-net
```

To switch to Docker Compose (§6), remove the containers first: `docker rm -f web database`.

---

## 6. Quick start with Docker Compose

This starts the same PostGIS and FROST containers as §5, configured in one file. If you followed §5, stop and remove those containers first (§5.4).

```bash
# 1. Get the project
git clone https://github.com/badreddineouzougarh/parking_pykci.git
cd parking_pykci

# 2. Create your local settings, then CHECK THE PORTS (§4)
cp .env.example .env

# 3. Start the core stack (FROST + PostGIS)
docker compose up -d

# 4. Check that every container is running
docker compose ps
```

Then open:

| What | URL |
|---|---|
| Landing page | http://localhost:8080/FROST-Server/ |
| Database status | http://localhost:8080/FROST-Server/DatabaseStatus |
| API root (v1.1) | http://localhost:8080/FROST-Server/v1.1 |

The API root lists the entity sets (`Things`, `Locations`, `Datastreams`, …). If you see that list, the server works.

> The compose file sets `persistence_autoUpdateDatabase=true`, so FROST creates its tables at start-up. If the list shows errors, open **DatabaseStatus** and click the update button.

To stop the stack:

```bash
docker compose down        # stops containers, KEEPS your data
docker compose down -v     # stops containers and DELETES the database volume
```

---

## 7. The docker-compose.yml explained

```yaml
services:

  # ── FROST-Server: the SensorThings API (HTTP + MQTT, all-in-one image) ──
  web:
    image: fraunhoferiosb/frost-server:latest   # pin a version (e.g. 2.x) for the course
    restart: unless-stopped
    ports:
      - "${FROST_HTTP_PORT:-8080}:8080"
      - "${FROST_MQTT_PORT:-1883}:1883"
    environment:
      - serviceRootUrl=http://localhost:${FROST_HTTP_PORT:-8080}/FROST-Server
      - http_cors_enable=true
      - http_cors_allowed_origins=*
      - persistence_db_driver=org.postgresql.Driver
      - persistence_db_url=jdbc:postgresql://database:5432/sensorthings   # ← container port, never change
      - persistence_db_username=sensorthings
      - persistence_db_password=${POSTGRES_PASSWORD:-ChangeMe}
      - persistence_autoUpdateDatabase=true
    depends_on:
      - database

  # ── PostgreSQL with PostGIS ──
  database:
    image: postgis/postgis:16-3.4-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_DB=sensorthings
      - POSTGRES_USER=sensorthings
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-ChangeMe}
    ports:
      - "${DB_HOST_PORT:-5432}:5432"            # only needed to connect with pgAdmin / QGIS
    volumes:
      - postgis_volume:/var/lib/postgresql/data

  # ── Optional: MQTT broker for devices ──
  mosquitto:
    image: eclipse-mosquitto:2
    profiles: ["iot"]
    restart: unless-stopped
    ports:
      - "${MOSQUITTO_HOST_PORT:-1884}:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config

  # ── Optional: low-code flows ──
  nodered:
    image: nodered/node-red:latest
    profiles: ["iot"]
    restart: unless-stopped
    ports:
      - "${NODERED_HOST_PORT:-1880}:1880"
    volumes:
      - nodered_data:/data

volumes:
  postgis_volume:
  nodered_data:
```

What the important lines do:

| Setting | Meaning |
|---|---|
| `serviceRootUrl` | Public address of the API. FROST writes it into every `@iot.selfLink` and `@iot.navigationLink`. |
| `http_cors_enable` | Lets browser apps (Leaflet, OpenLayers) on other origins call the API. |
| `persistence_db_url` | JDBC connection; `database` is the service name inside the Docker network. |
| `persistence_autoUpdateDatabase` | Creates/updates the tables automatically at start-up. |
| `profiles: ["iot"]` | Mosquitto and Node-RED start only when you ask for them (§11). |
| `postgis_volume` | Named volume, so the data survives `docker compose down`. |

> **Security:** this setup is for local teaching. It has no authentication and uses a default password. Do not expose it to the internet as it is.

---

## 8. The SensorThings data model

```
 Location ◀──▶ Thing ───▶ HistoricalLocation
                 │
                 ▼
 Sensor ───▶ Datastream ◀─── ObservedProperty
                 │
                 ▼
            Observation ◀─── FeatureOfInterest
```

| Entity | Meaning | Example |
|---|---|---|
| **Thing** | The object being monitored | Weather station on campus |
| **Location** | Where the Thing is (GeoJSON) | Point at the station |
| **HistoricalLocation** | Where the Thing was, and when (created automatically) | — |
| **Datastream** | One time series and its unit | Air temperature every 10 min, °C |
| **Sensor** | The instrument | DHT22 |
| **ObservedProperty** | What is measured | Air temperature |
| **Observation** | One result at one time | 21.4 at 10:00 |
| **FeatureOfInterest** | What the observation is about (geometry) | The air at the station, a field parcel |

**Location** and **FeatureOfInterest** hold geometry, which is what makes spatial queries possible (§10).

---

## 9. Create your first data

### 9.1 One request for Thing + Location + Datastream (deep insert)

`data/station.json`:

```json
{
  "name": "Campus Weather Station 01",
  "description": "Teaching station on campus",
  "properties": { "owner": "Project Session" },
  "Locations": [{
    "name": "Station 01 position",
    "description": "Roof of building A",
    "encodingType": "application/geo+json",
    "location": { "type": "Point", "coordinates": [-6.8650, 33.9716] }
  }],
  "Datastreams": [{
    "name": "Air temperature – Station 01",
    "description": "Air temperature every 10 minutes",
    "observationType": "http://www.opengis.net/def/observationType/OGC-OM/2.0/OM_Measurement",
    "unitOfMeasurement": {
      "name": "degree Celsius",
      "symbol": "°C",
      "definition": "http://unitsofmeasure.org/ucum.html#para-30"
    },
    "Sensor": {
      "name": "DHT22",
      "description": "Digital temperature and humidity sensor",
      "encodingType": "application/pdf",
      "metadata": "https://www.sparkfun.com/datasheets/Sensors/Temperature/DHT22.pdf"
    },
    "ObservedProperty": {
      "name": "Air temperature",
      "description": "Temperature of the ambient air",
      "definition": "http://vocabs.lter-europe.net/EnvThes/22035"
    }
  }]
}
```

> GeoJSON coordinates are **[longitude, latitude]**, not the other way round. Replace them with your own position.

```bash
curl -X POST -H "Content-Type: application/json" \
     -d @data/station.json \
     http://localhost:8080/FROST-Server/v1.1/Things
```

A `201 Created` response with a `Location` header means the Thing was created. Check it in the browser:
http://localhost:8080/FROST-Server/v1.1/Things?$expand=Locations,Datastreams

### 9.2 Post an Observation

`data/observation.json`:

```json
{
  "phenomenonTime": "2026-11-09T10:00:00Z",
  "result": 21.4,
  "Datastream": { "@iot.id": 1 }
}
```

```bash
curl -X POST -H "Content-Type: application/json" \
     -d @data/observation.json \
     http://localhost:8080/FROST-Server/v1.1/Observations
```

If you send no `FeatureOfInterest`, FROST creates one from the Thing's Location.

---

## 10. Query the data

Base URL: `http://localhost:8080/FROST-Server/v1.1/`

| Request | Result |
|---|---|
| `Things` | All Things |
| `Things?$expand=Locations,Datastreams` | Things with position and time series |
| `Datastreams(1)/Observations?$orderby=phenomenonTime desc&$top=10` | The 10 latest values |
| `Observations?$filter=result gt 30` | Values above a threshold |
| `Observations?$filter=phenomenonTime ge 2026-11-01T00:00:00Z` | Values since a date |
| `Things?$filter=geo.intersects(Locations/location, geography'POLYGON((-6.95 33.90, -6.75 33.90, -6.75 34.05, -6.95 34.05, -6.95 33.90))')` | **Spatial query:** Things inside a study area |
| `Datastreams?$select=name,unitOfMeasurement` | Only selected fields |
| `Things?$count=true&$top=5` | Paging, with the total count |

Other spatial functions: `geo.distance`, `st_within`, `st_contains`, `st_intersects`.
If the browser rejects a URL, encode the spaces as `%20`.

### Real-time with MQTT

FROST publishes every new entity on MQTT (port 1883):

```bash
mosquitto_sub -h localhost -p 1883 -t "v1.1/Datastreams(1)/Observations"
```

You can also **create** an Observation by publishing JSON to `v1.1/Datastreams(1)/Observations`.

---

## 11. Optional: Mosquitto + Node-RED

Start them with the `iot` profile:

```bash
docker compose --profile iot up -d
```

`mosquitto/config/mosquitto.conf` (Mosquitto 2 rejects external clients without a listener config):

```conf
listener 1883
allow_anonymous true
```

> Mosquitto is published on host port **1884** so it doesn't clash with FROST's MQTT on 1883. Devices outside Docker connect to `localhost:1884`.

Example flow in Node-RED (http://localhost:1880):

1. **mqtt in** node: server `mosquitto`, port `1883`, topic `devices/station01/temp`
2. **function** node to convert the device message into an Observation:
   ```js
   msg.payload = {
     phenomenonTime: new Date().toISOString(),
     result: Number(msg.payload),
     Datastream: { "@iot.id": 1 }
   };
   return msg;
   ```
3. **http request** node: method `POST`, URL `http://web:8080/FROST-Server/v1.1/Observations`, return a parsed JSON object

Inside Docker, containers reach each other by **service name and container port** (`mosquitto:1883`, `web:8080`), never `localhost`.

Test it without a real device:

```bash
mosquitto_pub -h localhost -p 1884 -t devices/station01/temp -m 22.7
```

---

## 12. View the data in QGIS

1. QGIS 3.36 or later: **Layer → Data Source Manager → SensorThings**
2. **New** connection, URL: `http://localhost:8080/FROST-Server/v1.1`
3. Entity type **Locations** (or Things), geometry type **Point** → **Add**
4. Style the layer, or join the latest Observation values to it for analysis

---

## 13. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `port is already allocated` / `address already in use` | Port clash. Change the host port (§4). |
| Links in responses point to `:8080` but you use `:8081` | `serviceRootUrl` doesn't match the published port. |
| FROST log: `Connection refused` to the database | The DB is still starting. Wait and run `docker compose restart web`, and check that `persistence_db_url` uses `database:5432`. |
| `password authentication failed` | You changed the password after the first start. The old one is stored in the volume: `docker compose down -v` (**deletes data**) and start again. |
| Empty list or SQL errors on `/v1.1/Things` | Tables are missing. Open `/DatabaseStatus` and run the update. |
| Mosquitto: "only local connections" | `mosquitto.conf` is missing or not mounted (§11). |
| `curl` errors on Windows | Use Git Bash, WSL or Postman. |
| `Conflict. The container name "/web" is already in use` | Containers from §5 still exist. Remove them: `docker rm -f web database`. |
| `docker: command not found` / cannot connect to the daemon | Docker Desktop is not started, or (Linux) log out and in after `usermod` (§5.1). |

Useful commands:

```bash
docker compose ps                 # status
docker compose logs -f web        # FROST logs
docker compose logs -f database   # database logs
docker compose restart web        # restart FROST only
docker compose pull               # update images
```

---

## 14. References

- OGC SensorThings API standard: https://www.ogc.org/standards/sensorthings/
- SensorThings API overview: https://ogcapi.ogc.org/sensorthings/
- FROST-Server documentation: https://fraunhoferiosb.github.io/FROST-Server/
- FROST-Server Docker deployment: https://fraunhoferiosb.github.io/FROST-Server/deployment/docker.html
- FROST-Server on Docker Hub: https://hub.docker.com/r/fraunhoferiosb/frost-server
- PostGIS Docker image: https://hub.docker.com/r/postgis/postgis
- Docker Desktop: https://docs.docker.com/desktop/
