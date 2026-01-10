# Traefik Ingress for Multi-App VPS

This project sets up a single **Traefik container** that handles all incoming traffic on a VPS and routes it to different Docker apps. Traefik also handles **TLS (HTTPS)** automatically.



## How It Works

- Traefik listens on **ports 80 (HTTP) and 443 (HTTPS)**.  
- Apps connect to a shared Docker network called **`edge`**.  
- Traefik routes requests to apps using **Docker labels**.  
- Private services like **Postgres, Redis, Celery, Flower** stay on their **internal networks** and are not exposed directly.  

**Traffic Flow Example:**<br/>
```
Internet -> VPS:80/443 -> Traefik -> edge network -> App container
```

## Getting Started

1. Install on your VPS:  
   - Docker  
   - Git  

2. Pull this repo from github

3. Create the global `edge` network:

```bash
docker network create edge
```

4. Ensure that following folders exist
- platform/infra/traefik/acme/acme.json
- platform/infra/traefik/dynamic/

5. Change acme.json file permissions:
```bash
chmod 600 platform/infra/traefik/acme/acme.json
```
6. Define an environment variable for emails
```bash
export TRAEFIK_ACME_EMAIL=<youremailaddress>
```
7. Navigate to the Traefik app folder
```bash
cd traefik
```

8. Run docker compose
```bash
docker compose up -d
```

At this point your Traefik container should be running. You can now add apps.

## Adding Apps

1. Each app has its own folder and Docker Compose file.
2. Public services attach to the `edge` network.
3. Private services (DB, Redis, workers) use a per-app internal network.
4. Connect the app with Traefik by including Traefik specific labels in the Docker Compose file for the app. Traefik will recognize the labels.

Example Traefik labels for a public service:

```bash
labels:
  - traefik.enable=true
  - traefik.http.routers.myapp.rule=Host(`myapp.example.com`)
  - traefik.http.routers.myapp.entrypoints=websecure
  - traefik.http.routers.myapp.tls.certresolver=letsencrypt
networks:
  edge: # required if the app wants to connect to the internet
    external: true
```

## Best Practices

1. Only Traefik binds host ports (80/443).
2. Do not expose app ports directly.
3. Keep acme.json safe (chmod 600).
4. Use labels for routing; dynamic folder for shared middlewares.
5. Maintain consistent folder structure between dev and prod.

## Sample NGINX Docker Compose File
```yaml
services:
  web:
    image: nginx:stable-alpine
    container_name: myapp_web
    restart: unless-stopped
    networks:
      - edge
    labels:
      - traefik.enable=true
      - traefik.http.routers.myapp.rule=Host(`www.example.com`)
      - traefik.http.routers.myapp.entrypoints=websecure
      - traefik.http.routers.myapp.tls.certresolver=letsencrypt

networks:
  edge:
    external: true
```