# ⚙️ The Apparat Suite

A set of small, decoupled, single-purpose demoparty management tools. 

Originally built for the [Evoke demoparty](https://www.evoke.eu/) to handle the operational "side quests" that competition-focused systems like [Granola](https://gitlab.com/granola-compo/granola/) intentionally leave out. All tools are open-source, run as Docker containers, and are built with Go and React.

<p align="center">
  <a href="https://github.com/potibm/kasseapparat"><img src="https://github.com/potibm/kasseapparat/raw/main/docs/kasseapparat.svg" width="120" alt="kasseapparat logo" /></a>
  &nbsp;&nbsp;&nbsp;&nbsp;
  <a href="https://github.com/potibm/tidsapparat"><img src="https://github.com/potibm/tidsapparat/raw/main/doc/tidsapparat.svg" width="120" alt="tidsapparat logo" /></a>
  &nbsp;&nbsp;&nbsp;&nbsp;
  <a href="https://github.com/potibm/funkapparat"><img src="https://github.com/potibm/funkapparat/raw/main/doc/funkapparat.svg" width="120" alt="funkapparat logo" /></a>
  &nbsp;&nbsp;&nbsp;&nbsp;
  <a href="https://github.com/potibm/billedapparat"><img src="https://github.com/potibm/billedapparat/raw/main/doc/billedapparat.svg" width="120" alt="billedapparat logo" /></a>
</p>

## 📦 The Tools

* **[kasseapparat](https://github.com/potibm/kasseapparat):** Lightweight POS and guestlist manager for the entrance desk.
* **[tidsapparat](https://github.com/potibm/tidsapparat):** Timetable manager to adjust running orders and serve structured schedule data.
* **[funkapparat](https://github.com/potibm/funkapparat):** Minimal news hub for quick text announcements.
* **[billedapparat](https://github.com/potibm/billedapparat):** Big-screen display system for the main hall. A central hub rotates slides, while separate collectors feed it with schedules from *tidsapparat*, news from *funkapparat* and posts from social media.
* **[protokolapparat](https://github.com/potibm/protokolapparat):** The shared glue. It defines the protocol and data structures used to exchange news and events between *tidsapparat*, *funkapparat*, and *billedapparat* via Redis Streams.

## 🏗️ Architecture & Interaction

The Apparat suite relies on decoupling. There is no central monolithic database — each tool keeps its own SQLite database. Where tools need to talk to each other, they do so asynchronously via Redis Streams: `tidsapparat` and `funkapparat` append their updates to a stream, and `billedapparat` reads them through a consumer group.

`billedapparat` itself is split into two parts:

* The **hub** (`billedapparat serve`) stores slides, news and timetable entries, serves the admin UI and the beamer configuration, and pushes live updates to the beamer.
* The **collectors** (`billedapparat collect <source>`) run as separate processes. Each one gathers content from a single source and pushes it to the hub over HTTP, authenticated with an API key. There are collectors for news and the timetable (both reading from Redis) as well as for Bluesky, Mastodon, Discord, Twitch and Pouet. For posts with images, the hub downloads the media itself.

For party visitors there are two read paths: the beamer pulls its playlists and configuration from `billedapparat` over HTTP and receives live slide updates over an SSE stream, while `tidsapparat` and `funkapparat` can additionally export their data (iCal, RSS/Atom) to an S3 bucket. The beamer that drives the hall screens is just a browser on a machine in the organizer area — the big screens only mirror its output. Visitors fetch the exported files directly with a calendar client, an RSS reader or a simple web page — optionally served through a CDN in front of the bucket (e.g. CloudFront, as used at Evoke).

`kasseapparat` works on its own and does not use Redis. It talks to the [SumUp](https://www.sumup.com/) API for card payments (including a webhook back from SumUp) and sends guest arrival notifications to organizers by e-mail via SMTP (e.g. Amazon SES).

To keep the services lean, there is **no built-in user management**. Login is delegated to an external OIDC provider (e.g. Dex or Keycloak); `kasseapparat` can alternatively trust a user header set by a ForwardAuth middleware in the proxy. Routing and TLS are handled by a reverse proxy like Traefik.

```mermaid
flowchart TB
    Orga([Party Organizer])
    Visitor([Party Visitor])

    subgraph Edge["Edge"]
        Traefik["Reverse Proxy<br>Traefik"]
        IdP["OIDC Provider<br>e.g. Dex"]
    end

    subgraph Suite["The Apparat Suite"]
        K["kasseapparat<br>POS + guest list"]
        T["tidsapparat<br>timetable"]
        F["funkapparat<br>news"]

        subgraph BA["billedapparat"]
            Hub["Hub<br>Admin API · Config · SSE"]
            subgraph Collectors["Collectors · billedapparat collect &lt;source&gt;"]
                CNews["news"]
                CTime["timetable"]
                CSocial["bluesky · mastodon · discord<br>twitch · pouet"]
            end
        end
    end

    subgraph Data["Messaging & Storage"]
        Redis[("Redis Streams<br>protokolapparat")]
        S3[("S3 Bucket<br>iCal · RSS/Atom")]
    end

    subgraph External["External Services"]
        SES["Amazon SES<br>SMTP"]
        SumUp["SumUp API<br>card terminals"]
        Social["Bluesky · Mastodon · Discord<br>Twitch · Pouet"]
    end

    subgraph OrgaArea["Organizer Area"]
        Beamer["Beamer<br>Browser on /beamer"]
    end

    subgraph Hall["Hall"]
        Screens["Big Screens"]
    end

    %% Access
    Orga -->|"Admin UIs"| Traefik
    Orga -.->|"Login"| IdP
    Traefik --> K & T & F & Hub
    Beamer <-->|"Config + Slides<br>HTTP / SSE"| Traefik
    Beamer -->|"Video out"| Screens
    Visitor -->|"watches"| Screens
    Visitor -->|"Calendar, RSS, Web"| S3

    %% kasseapparat
    K -->|"arrival notifications"| SES
    SES -->|"E-Mail"| Orga
    K <-->|"payments + webhook"| SumUp

    %% timetable & news
    T -->|"XADD schedule"| Redis
    F -->|"XADD news"| Redis
    T -->|"export iCal"| S3
    F -->|"export RSS/Atom"| S3

    %% billedapparat ingest
    Redis -->|"XREADGROUP"| CNews & CTime
    Social -->|"API / Gateway / Polling"| CSocial
    CNews & CTime & CSocial -->|"HTTP push + API key"| Hub
    Hub -.->|"media download"| Social

    classDef app fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef ext fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef store fill:#dcfce7,stroke:#16a34a,color:#14532d
    class K,T,F,Hub,CNews,CTime,CSocial app
    class SES,SumUp,Social ext
    class Redis,S3 store
```

_Not shown for readability: each tool's SQLite database and the optional observability stack (OpenTelemetry, Sentry)._

## 🚀 Quickstart (Docker Compose)

The easiest way to run the full suite locally is using `docker compose`. This structural example includes a basic Traefik proxy for routing and a Redis instance for internal communication.

_Note: In a production environment, you should secure the admin interfaces by configuring each tool's `auth` section against your OIDC provider (or, for `kasseapparat`, a ForwardAuth middleware in Traefik). The `billedapparat` collectors are started as additional processes (`billedapparat collect <source>`) and are not part of this minimal example._

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

  redis:
    image: redis:7-alpine
    expose:
      - "6379"

  kasseapparat:
    image: ghcr.io/potibm/kasseapparat:latest
    labels:
      - "traefik.http.routers.kasse.rule=Host(`kasse.apparat.localhost`)"

  tidsapparat:
    image: ghcr.io/potibm/tidsapparat:latest
    environment:
      - REDIS_URL=redis:6379
    labels:
      - "traefik.http.routers.tids.rule=Host(`tids.apparat.localhost`)"

  funkapparat:
    image: ghcr.io/potibm/funkapparat:latest
    environment:
      - REDIS_URL=redis:6379
    labels:
      - "traefik.http.routers.funk.rule=Host(`funk.apparat.localhost`)"

  billedapparat:
    image: ghcr.io/potibm/billedapparat:latest
    environment:
      - REDIS_URL=redis:6379
    labels:
      - "traefik.http.routers.billed.rule=Host(`billed.apparat.localhost`)"
```

Run `docker compose up -d` and access the tools via `http://*.apparat.localhost`.

## ⚙️ Configuration & Data

The `docker-compose.yml` above outlines the general architecture and routing. However, to make the tools fully functional, you need to provide their specific configuration files (e.g., to set the Redis URL, CORS headers, or database paths).

Each tool is configured via its own `config.yaml` or environment variables. Please refer to the documentation in the individual repositories for detailed setup instructions and full configuration examples.