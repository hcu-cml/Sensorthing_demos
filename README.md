# IoT Server for Geospatial Data — OGC SensorThings API with FROST-Server

In this project you build a complete **OGC SensorThings API** server on your own computer.
You will run **FROST-Server**, which stores sensor data with real geometries in **PostgreSQL/PostGIS**, and then add data, query it and show it on a map.

> Course: Project Session — *OGC SensorThings API* · Lecturer: Badreddine Ouzougarh

> 📘 **This repository contains only this guide.** There is no code to download. You create every file yourself, following the steps below.

---

## Contents

1. [Architecture](#1-architecture)
2. [Requirements](#2-requirements)
3. [Your project folder](#3-your-project-folder)
4. [⚠ Check your ports first](#4--check-your-ports-first)
5. [**Set up the environment: Docker → PostGIS → FROST**](#5-set-up-the-environment-docker--postgis--frost)
   - [Step 1 · Install Docker](#step-1--install-docker)
   - [Step 2 · Run PostgreSQL + PostGIS](#step-2--run-postgresql--postgis)
   - [Step 3 · Run FROST-Server](#step-3--run-frost-server)
   - [Stop, restart and clean up](#stop-restart-and-clean-up)
6. [The same stack with Docker Compose](#6-the-same-stack-with-docker-compose)
7. [The SensorThings data model](#7-the-sensorthings-data-model)
8. [Create your first data](#8-create-your-first-data)
9. [Query the data](#9-query-the-data)
10. [Optional: Mosquitto + Node-RED](#10-optional-mosquitto--node-red)
11. [View the data in QGIS](#11-view-the-data-in-qgis)
12. [Build a simple web viewer](#12-build-a-simple-web-viewer)
13. [Troubleshooting](#13-troubleshooting)
14. [References](#14-references)

---

## 1. Architecture

```
 ┌──────────┐   MQTT   ┌────────────┐        ┌───────────┐  HTTP/MQTT  ┌──────────────┐  JDBC  ┌──────────┐
 │ Sensors  │ ───────▶ │ Mosquitto  │ ─────▶ │ Node-RED  │ ──────────▶ │ FROST-Server │ ◀────▶ │ PostGIS  │
 │ ESP32,RPi│          │ (optional) │        │ (optional)│             │ :8080 :1883  │        │  :5432   │
 └──────────┘          └────────────┘        └───────────┘             └──────┬───────┘        └──────────┘
                                                                               │ REST / MQTT
                                                         ┌─────────────────────▼──────────────────────┐
                                                         │ Clients: QGIS · Web viewer · Python         │
                                                         └─────────────────────────────────────────────┘
```

| Service | Role | Required | Default port |
|---|---|---|---|
| **FROST-Server** | Implements the SensorThings API (REST + built-in MQTT) | ✅ | 8080 (HTTP), 1883 (MQTT) |
| **PostgreSQL + PostGIS** | Stores all entities; locations as geometries | ✅ | 5432 |
| **Mosquitto** | Stand-alone MQTT broker for field devices | optional | 1884 on the host (see §4) |
| **Node-RED** | Converts device messages into SensorThings Observations | optional | 1880 |

The minimal setup is **FROST + PostGIS**, both running in Docker. Mosquitto and Node-RED are only needed when real devices send data in their own format.

---

## 2. Requirements

- A computer with **8 GB RAM** or more and about **3 GB of free disk space**
- **Admin rights** on your computer (to install Docker)
- A tool for HTTP requests: your browser (GET), plus **curl**, **Postman** or **Bruno** (POST)
- Optional: **QGIS 3.36+**, **MQTT Explorer**, a code editor such as **VS Code**

> **Windows users:** run the commands in **PowerShell**. Each step gives you a PowerShell version where it differs from macOS / Linux.

---

## 3. Your project folder

Create this folder on your computer. You will fill it step by step; the section that creates each file is shown on the right.

```
sensorthings-project/
├── docker-compose.yml        # §6  – whole stack in one file (optional)
├── .env                      # §6  – your ports and password
├── data/
│   ├── station.json          # §8  – example Thing with Location + Datastream
│   └── observation.json      # §8  – example Observation
├── mosquitto/                # §10 – optional
│   └── config/mosquitto.conf
└── Client_viewer/            # §12 – web map
    ├── index.html
    └── script.js
```

Open a terminal **inside this folder** for every command in this guide.



## 5. Set up the environment: Docker → PostGIS → FROST

> 🧭 **This is the part where most people get stuck.** Go slowly, run the check at the end of each step, and **don't move on until it passes.** Each step has a *Stuck?* table with the most common errors.

You install three components, in this order:

```
 Step 1            Step 2                     Step 3
┌────────┐       ┌──────────────────┐       ┌──────────────┐
│ Docker │ ────▶ │ PostgreSQL+PostGIS│ ────▶ │ FROST-Server │
└────────┘       └──────────────────┘       └──────────────┘
 runs the         stores the data            serves the API,
 containers                                  needs the database
```

**Your progress:**

- [ ] **Step 1** · `docker run --rm hello-world` prints *Hello from Docker!*
- [ ] **Step 2** · PostGIS answers with its version number
- [ ] **Step 3** · http://localhost:8080/FROST-Server/v1.1 lists the entity sets

---

### Step 1 · Install Docker

**🎯 Goal:** Docker runs on your computer and can start containers.

#### 1a. Install

**Windows 10 / 11**

1. Docker Desktop needs **WSL 2**. Open PowerShell **as administrator** and run:
   ```powershell
   wsl --install
   wsl --update
   ```
   Restart the computer when asked.
2. Download **Docker Desktop for Windows** from https://www.docker.com/products/docker-desktop/ and run the installer. Keep **Use WSL 2 instead of Hyper-V** ticked.
3. Start Docker Desktop, accept the terms and wait until the bottom-left corner shows **Engine running** (green).

**macOS**

1. Download **Docker Desktop for Mac** from https://www.docker.com/products/docker-desktop/. Choose **Apple silicon** (M1–M4) or **Intel** to match your Mac (Apple menu → About This Mac).
2. Drag Docker into Applications, start it and wait until it shows **Engine running**.

**Linux (Ubuntu / Debian)**

```bash
curl -fsSL https://get.docker.com | sudo sh   # installs Docker Engine + Compose plugin
sudo usermod -aG docker $USER                 # use docker without sudo
```

Log out and back in so the group change takes effect.

#### 1b. ✅ Check

```bash
docker --version
docker compose version
docker run --rm hello-world
```

**Passed** when `hello-world` prints *Hello from Docker!*

#### 1c. Stuck?

| You see | Do this |
|---|---|
| `docker: command not found` / *not recognized* | Docker Desktop isn't installed or the terminal was open during install. Close and reopen the terminal. |
| *Cannot connect to the Docker daemon* / *error during connect* | Docker Desktop isn't running. Start it and wait for **Engine running**. |
| *WSL 2 installation is incomplete* / *WSL update required* (Windows) | Run `wsl --update` as administrator, then restart Docker Desktop. |
| *Virtualization support not detected* / *Hardware assisted virtualization* (Windows) | Enable virtualization in the BIOS/UEFI (**Intel VT-x** or **AMD-V / SVM**), then restart. |
| Docker Desktop stays on *Starting…* | Quit it, run `wsl --shutdown` (Windows), start it again. |
| *permission denied while trying to connect to the Docker daemon socket* (Linux) | You skipped the log-out after `usermod`. Log out and back in, or run `newgrp docker`. |

---

### Step 2 · Run PostgreSQL + PostGIS

**🎯 Goal:** a database called `sensorthings` with the PostGIS extension, running in a container.

We use the official **postgis/postgis** image: PostgreSQL with PostGIS already installed and enabled in the database it creates.

#### 2a. Create a network and a volume

```bash
docker network create sensorthings-net   # lets FROST find the database by name
docker volume create postgis_volume      # keeps the data when the container is removed
```

#### 2b. Start the database container

**macOS / Linux**

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

**Windows (PowerShell, one line)**

```powershell
docker run -d --name database --network sensorthings-net -e POSTGRES_DB=sensorthings -e POSTGRES_USER=sensorthings -e POSTGRES_PASSWORD=ChangeMe -p 5432:5432 -v postgis_volume:/var/lib/postgresql/data postgis/postgis:16-3.4-alpine
```

| Option | Meaning |
|---|---|
| `--name database` | Container name. **FROST uses it as the host name** in Step 3. |
| `--network sensorthings-net` | Puts the database on the same network as FROST |
| `-e POSTGRES_DB / USER / PASSWORD` | Database, user and password, created **only at the very first start** |
| `-p 5432:5432` | Publishes the port on your computer (for pgAdmin or QGIS). **Port busy? Use `-p 5433:5432`** (§4). |
| `-v postgis_volume:…` | Stores the data in the volume |

> The first start downloads the image (~150 MB) and takes a minute.

#### 2c. ✅ Check

```bash
docker ps
docker logs database
docker exec -it database psql -U sensorthings -d sensorthings -c "SELECT postgis_full_version();"
```

**Passed** when:
- `docker ps` lists `database` with status **Up**
- the log ends with *database system is ready to accept connections*
- the last command prints a line starting with `POSTGIS="3.4…`

If the last command says *function postgis_full_version() does not exist*, enable the extension once and check again:

```bash
docker exec -it database psql -U sensorthings -d sensorthings -c "CREATE EXTENSION IF NOT EXISTS postgis;"
```

#### 2d. Stuck?

| You see | Do this |
|---|---|
| *port is already allocated* / *address already in use* | Something already uses 5432 (often a PostgreSQL you installed earlier). Remove the container with `docker rm -f database` and start it again with `-p 5433:5432`. |
| *The container name "/database" is already in use* | You ran the command before. Start the existing one with `docker start database`, or remove it with `docker rm -f database` and run again. |
| *network sensorthings-net not found* | Run step 2a first. |
| `docker ps` doesn't show `database` | It stopped. Read the reason with `docker logs database`. |
| *password authentication failed* later on | The password is set only at the first start and kept in the volume. To reset: `docker rm -f database`, `docker volume rm postgis_volume`, then repeat 2a and 2b. **This deletes the data.** |
| *platform (linux/amd64) does not match* (Apple silicon Mac) | Only a warning: the image runs in emulation. If the container keeps stopping, enable **Settings → General → Use Rosetta** in Docker Desktop. |
| PowerShell: *unexpected token* or options ignored | Use the one-line PowerShell version. The `\` line breaks only work in bash. |

> **Already have PostgreSQL installed on your computer?** You can use it instead of the container: install PostGIS (Stack Builder on Windows, or your package manager), create the `sensorthings` user and database, and run `CREATE EXTENSION postgis;` in it. In Step 3, FROST then reaches it at `host.docker.internal` instead of `database`, and you leave out `--network`. For the course, the container is simpler.

---

### Step 3 · Run FROST-Server

**🎯 Goal:** FROST runs, connects to the database and answers on http://localhost:8080/FROST-Server/v1.1.

We use the **all-in-one** image `fraunhoferiosb/frost-server`: HTTP API and MQTT in one container.

> ⚠ **Before you run:** the database from Step 2 must be **Up** (`docker ps`). If you changed ports in §4, change them in this command too.

#### 3a. Start the FROST container

**macOS / Linux**

```bash
docker run -d --name web \
  --network sensorthings-net \
  -p 8080:8080 \
  -p 1883:1883 \
  -e serviceRootUrl=http://localhost:8080/FROST-Server \
  -e http_cors_enable=true \
  -e "http_cors_allowed_origins=*" \
  -e persistence_db_driver=org.postgresql.Driver \
  -e persistence_db_url=jdbc:postgresql://database:5432/sensorthings \
  -e persistence_db_username=sensorthings \
  -e persistence_db_password=ChangeMe \
  -e persistence_autoUpdateDatabase=true \
  fraunhoferiosb/frost-server:latest
```

**Windows (PowerShell, one line)**

```powershell
docker run -d --name web --network sensorthings-net -p 8080:8080 -p 1883:1883 -e serviceRootUrl=http://localhost:8080/FROST-Server -e http_cors_enable=true -e "http_cors_allowed_origins=*" -e persistence_db_driver=org.postgresql.Driver -e persistence_db_url=jdbc:postgresql://database:5432/sensorthings -e persistence_db_username=sensorthings -e persistence_db_password=ChangeMe -e persistence_autoUpdateDatabase=true fraunhoferiosb/frost-server:latest
```

| Option | Meaning |
|---|---|
| `--network sensorthings-net` | Same network as the database, otherwise FROST can't find it |
| `-p 8080:8080` | HTTP API. **Port busy? Use `-p 8081:8080`** and change `serviceRootUrl` to `:8081` too. |
| `-p 1883:1883` | FROST's built-in MQTT. **Port busy? Use `-p 1885:1883`.** |
| `serviceRootUrl` | The public address FROST writes into its links |
| `http_cors_enable`, `http_cors_allowed_origins` | Lets your web viewer (§12) call the API from the browser |
| `persistence_db_url` | `database` = the container name from Step 2. **Always port 5432**, whatever you published on your computer. |
| `persistence_db_username` / `_password` | **Must match** `POSTGRES_USER` / `POSTGRES_PASSWORD` from Step 2 |
| `persistence_autoUpdateDatabase=true` | Creates the SensorThings tables at start-up |

> FROST needs **30–60 seconds** to start. Give it time before you open the browser.

#### 3b. ✅ Check

```bash
docker ps            # "database" and "web" are both Up
docker logs web      # read the last lines: no "Connection refused", no "password authentication failed"
```

Then open in your browser:

| URL | You should see |
|---|---|
| http://localhost:8080/FROST-Server/ | The FROST landing page |
| http://localhost:8080/FROST-Server/DatabaseStatus | Database status. If it shows pending changes, click the update button. |
| http://localhost:8080/FROST-Server/v1.1 | JSON listing `Things`, `Locations`, `Datastreams`, `Sensors`, `ObservedProperties`, `Observations`, `FeaturesOfInterest`, `HistoricalLocations` |

**Passed** when the last URL shows that list. 🎉 **Your server is running.** Continue with §8 to create your first data.

#### 3c. Stuck?

| You see | Do this |
|---|---|
| Browser: *This site can't be reached* | FROST is still starting (wait a minute), or the container stopped: check `docker ps` and `docker logs web`. |
| Browser: *404 Not Found* | Wrong path. It is `/FROST-Server/` with that exact capitalisation. |
| Log: *Connection refused* / *UnknownHostException: database* | The database isn't running, or FROST is on another network. Check `docker ps`, then `docker network inspect sensorthings-net`: both containers must be listed. Fix it, then `docker restart web`. |
| Log: *password authentication failed for user "sensorthings"* | The FROST password doesn't match the database password. `docker rm -f web` and run 3a again with the right password. |
| *port is already allocated* on 8080 or 1883 | `docker rm -f web`, run 3a again with `-p 8081:8080` (and `serviceRootUrl=…:8081…`) or `-p 1885:1883`. |
| Page works, but the links inside point to `:8080` while you use `:8081` | `serviceRootUrl` still says 8080. `docker rm -f web` and run 3a again with the right port in both places. |
| `/v1.1/Things` returns an error about missing tables | Open `/DatabaseStatus` and click the update button. |
| *The container name "/web" is already in use* | `docker rm -f web`, then run 3a again. |

> **Changing an option means re-creating the container.** You can't edit a running container's settings: run `docker rm -f web`, then the full `docker run` command again. Your data is safe: it lives in the database, not in FROST.

---

### Stop, restart and clean up

```bash
docker stop web database          # stop (data is kept)
docker start database web         # start again: database first
docker rm -f web database         # remove the containers (data stays in the volume)
docker volume rm postgis_volume   # delete the data for good
docker network rm sensorthings-net
```

> After restarting your computer, start Docker Desktop, then `docker start database web`. You don't need to run the long commands again.

---

## 6. The same stack with Docker Compose

Once Steps 1–3 work, you can describe the same stack in **one file** and start it with **one command**. This is how you would run it in a real project.

> ⚠ Use **either** the manual containers (§5) **or** Docker Compose, not both: they use the same names and ports. Remove the manual ones first: `docker rm -f web database`.

**1. Create `.env`** in your project folder (adapt the ports from §4):

```dotenv
DB_HOST_PORT=5432
FROST_HTTP_PORT=8080
FROST_MQTT_PORT=1883
MOSQUITTO_HOST_PORT=1884
NODERED_HOST_PORT=1880
POSTGRES_PASSWORD=ChangeMe
```

**2. Create `docker-compose.yml`** in the same folder:

```yaml
services:

  # ── FROST-Server: the SensorThings API (HTTP + MQTT, all-in-one image) ──
  web:
    image: fraunhoferiosb/frost-server:latest
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
      - "${DB_HOST_PORT:-5432}:5432"
    volumes:
      - postgis_volume:/var/lib/postgresql/data

  # ── Optional: MQTT broker for devices (§10) ──
  mosquitto:
    image: eclipse-mosquitto:2
    profiles: ["iot"]
    restart: unless-stopped
    ports:
      - "${MOSQUITTO_HOST_PORT:-1884}:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config

  # ── Optional: low-code flows (§10) ──
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

> Compose creates its own network and gives the volume a project prefix, so it starts with an **empty database**, separate from §5.

**3. Start and check**

```bash
docker compose up -d      # start FROST + PostGIS
docker compose ps         # both services "running"
```

Open http://localhost:8080/FROST-Server/v1.1 as in Step 3.

| Compare with §5 | `docker run` (manual) | Docker Compose |
|---|---|---|
| Network | `docker network create` | created automatically |
| Settings | long command lines | `docker-compose.yml` + `.env` |
| Start / stop | `docker start/stop web database` | `docker compose up -d` / `down` |
| Logs | `docker logs web` | `docker compose logs -f web` |

```bash
docker compose down        # stop, KEEP the data
docker compose down -v     # stop and DELETE the database volume
```

> **Security:** this setup is for local teaching. It has no authentication and uses a default password. Do not expose it to the internet as it is.

---

## 7. The SensorThings data model

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

**Location** and **FeatureOfInterest** hold geometry, which is what makes spatial queries possible (§9).

---

## 8. Create your first data

### 8.1 One request for Thing + Location + Datastream (deep insert)

Create `data/station.json`:

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

Send it:

```bash
# macOS / Linux
curl -X POST -H "Content-Type: application/json" \
     -d @data/station.json \
     http://localhost:8080/FROST-Server/v1.1/Things
```

```powershell
# Windows (PowerShell)
Invoke-RestMethod -Method Post -ContentType "application/json; charset=utf-8" -InFile data/station.json -Uri http://localhost:8080/FROST-Server/v1.1/Things
```

No error means the Thing was created. Check it in the browser:
http://localhost:8080/FROST-Server/v1.1/Things?$expand=Locations,Datastreams

### 8.2 Post an Observation

Create `data/observation.json`:

```json
{
  "phenomenonTime": "2026-11-09T10:00:00Z",
  "result": 21.4,
  "Datastream": { "@iot.id": 1 }
}
```

```bash
# macOS / Linux
curl -X POST -H "Content-Type: application/json" \
     -d @data/observation.json \
     http://localhost:8080/FROST-Server/v1.1/Observations
```

```powershell
# Windows (PowerShell)
Invoke-RestMethod -Method Post -ContentType "application/json; charset=utf-8" -InFile data/observation.json -Uri http://localhost:8080/FROST-Server/v1.1/Observations
```

Change the time and the result and send it a few more times to build a short time series. If you send no `FeatureOfInterest`, FROST creates one from the Thing's Location.

> Check the Datastream id first: open `/v1.1/Datastreams` and use its `@iot.id` instead of `1` if it differs.

---

## 9. Query the data

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

FROST publishes every new entity on MQTT (port 1883). Connect **MQTT Explorer** to `localhost:1883` and subscribe to:

```
v1.1/Datastreams(1)/Observations
```

Post a new Observation (§8.2) and watch it arrive. You can also **create** an Observation by publishing JSON to that topic.

---

## 10. Optional: Mosquitto + Node-RED

Use this when real devices send data over MQTT in their own format.

**1. Create `mosquitto/config/mosquitto.conf`** (Mosquitto 2 rejects external clients without it):

```conf
listener 1883
allow_anonymous true
```

**2. Start both services** with the `iot` profile from §6:

```bash
docker compose --profile iot up -d
```

> Mosquitto is published on host port **1884** so it doesn't clash with FROST's MQTT on 1883. Devices outside Docker connect to `localhost:1884`.

**3. Build a flow in Node-RED** (http://localhost:1880):

1. **mqtt in** node: server `mosquitto`, port `1883`, topic `devices/station01/temp`
2. **function** node, to turn the device message into an Observation:
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

**4. Test it** by publishing a fake reading with MQTT Explorer (`localhost:1884`, topic `devices/station01/temp`, message `22.7`), then check `/v1.1/Datastreams(1)/Observations`.

---

## 11. View the data in QGIS

1. QGIS 3.36 or later: **Layer → Data Source Manager → SensorThings**
2. **New** connection, URL: `http://localhost:8080/FROST-Server/v1.1`
3. Entity type **Locations** (or Things), geometry type **Point** → **Add**
4. Style the layer, or join the latest Observation values to it for analysis

---

## 12. Build a simple web viewer

A small Leaflet map that shows every Thing with its latest value. It works because CORS is enabled in Step 3.

Create `Client_viewer/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>SensorThings Viewer</title>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
  <style>html, body, #map { height: 100%; margin: 0; }</style>
</head>
<body>
  <div id="map"></div>
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script src="script.js"></script>
</body>
</html>
```

Create `Client_viewer/script.js`:

```js
// Change the port if you changed it in §4
const BASE = "http://localhost:8080/FROST-Server/v1.1";

const map = L.map("map").setView([33.97, -6.86], 13);
L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
  attribution: "© OpenStreetMap contributors"
}).addTo(map);

// Things + their location + the latest observation of each datastream
const url = BASE + "/Things?$expand=Locations,Datastreams($expand=Observations($orderby=phenomenonTime desc;$top=1))";

fetch(url)
  .then(r => r.json())
  .then(data => {
    const points = [];
    data.value.forEach(thing => {
      const loc = thing.Locations[0];
      if (!loc) return;
      const [lon, lat] = loc.location.coordinates;   // GeoJSON: [lon, lat]
      const lines = thing.Datastreams.map(ds => {
        const obs = ds.Observations[0];
        return ds.name + ": " + (obs ? obs.result + " " + ds.unitOfMeasurement.symbol : "no data");
      });
      L.marker([lat, lon]).addTo(map)
        .bindPopup("<b>" + thing.name + "</b><br>" + lines.join("<br>"));
      points.push([lat, lon]);
    });
    if (points.length) map.fitBounds(points, { maxZoom: 15 });
  })
  .catch(err => alert("Cannot reach FROST: " + err));
```

Open `index.html` in your browser (double-click it). Click a marker to see the latest value.

---

## 13. Troubleshooting

Each step in §5 has its own *Stuck?* table. These are the remaining problems:

| Symptom | Cause / fix |
|---|---|
| `port is already allocated` / `address already in use` | Port clash. Change the host port (§4). |
| Links in responses point to `:8080` but you use `:8081` | `serviceRootUrl` doesn't match the published port. |
| `password authentication failed` with Docker Compose | The password is kept in the volume from the first start. `docker compose down -v` (**deletes data**) and start again. |
| *The container name "/web" is already in use* with Docker Compose | Containers from §5 still exist. `docker rm -f web database`. |
| `curl` errors on Windows | Use `Invoke-RestMethod` (§8) or Postman. |
| Web viewer: *Cannot reach FROST* | FROST isn't running, the port in `script.js` is wrong, or CORS is not enabled (Step 3). |
| Mosquitto: "only local connections" | `mosquitto.conf` is missing or not mounted (§10). |



---

## . ⚠ Check your ports first

Docker publishes each container port on your computer. If a port is already in use, the container won't start, or you end up talking to a different program. **Check before you start any container.**

| Service | Default | Typical clash | Change to |
|---|---|---|---|
| PostgreSQL | 5432 | PostgreSQL already installed on your laptop | 5433 |
| FROST HTTP | 8080 | GeoServer, Tomcat, another web app | 8081 |
| FROST MQTT | 1883 | a Mosquitto service already installed | 1885 |
| Node-RED | 1880 | another Node-RED | 1881 |

Find out what is using a port:

```bash
# Windows (PowerShell)
netstat -ano | findstr :5432
# macOS / Linux
lsof -i :5432
```

No output means the port is free. Write down any port you have to change: you need it in Steps 2 and 3.

**Three rules:**

1. A port mapping is `HOST:CONTAINER`, e.g. `-p 5433:5432`. **Change only the left number.**
2. If you change the FROST HTTP port, change `serviceRootUrl` to the same port.
3. **Never change** `database:5432` in `persistence_db_url`. Inside Docker's network, FROST always reaches the database on port 5432.

---

Useful commands:

```bash
docker ps -a                  # all containers, also stopped ones
docker logs -f web            # FROST logs (Ctrl+C to leave)
docker logs database          # database logs
docker restart web            # restart FROST only
docker compose logs -f web    # FROST logs when using Docker Compose
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
- Leaflet: https://leafletjs.com/
