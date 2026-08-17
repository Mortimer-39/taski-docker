# AGENTS.md

## Project Purpose

This repository contains the containerized production deployment of **Taski**.

The main goal is to complete the project according to the Yandex Practicum assignment requirements:

- run the project in Docker containers;
- use Docker Compose for production;
- run automated tests in GitHub Actions;
- build and push Docker images to Docker Hub;
- automatically deploy the latest images to the remote Ubuntu server after a push to `main`;
- configure the server-side Nginx reverse proxy;
- keep the production deployment reproducible and minimal.

This file is guidance for AI coding agents such as OpenCode.

---

## Primary Rule

Before changing anything, inspect the actual repository.

Do not assume filenames, environment variable names, service names, ports, image names, or project structure.

The real repository is the source of truth.

When an assignment file supplied by the user is available, treat it as the primary source of requirements.

Do not redesign the project unless the existing implementation cannot satisfy the assignment.

Prefer minimal, targeted fixes.

---

## Current Known State

At the time this file was created:

- GitHub Actions starts successfully.
- GitHub Actions can connect to the remote Ubuntu server over SSH.
- Docker is installed on the remote server.
- Docker images can be pulled from Docker Hub.
- The server has a directory:

```text
~/taski
```

- The server currently contains:

```text
~/taski/docker-compose.production.yml
```

- Deployment currently expects:

```text
~/taski/.env
```

- The `.env` file is currently missing on the server.
- This causes Docker Compose deployment to fail.
- A typical failure observed was:

```text
env file /home/ubuntu/taski/.env not found
service "backend" is not running
```

- System Nginx on the Ubuntu host is not yet configured for the project.
- The repository may already contain a Docker `gateway` service using Nginx internally.

Do not confuse the Docker gateway Nginx with the Ubuntu host Nginx.

---

## Expected High-Level Architecture

The intended production flow should be approximately:

```text
Git push to main
        |
        v
GitHub Actions
        |
        +--> run tests
        |
        +--> build Docker images
        |
        +--> push images to Docker Hub
        |
        v
SSH to remote Ubuntu server
        |
        +--> update production compose configuration
        +--> ensure production environment variables exist
        +--> pull latest images
        +--> start/update containers
        +--> run migrations
        +--> collect backend static if required
        |
        v
Host Nginx
        |
        v
Docker gateway
        |
        +--> frontend
        +--> backend
        +--> static/media where applicable
        |
        v
PostgreSQL
```

Adapt this only after inspecting the real Compose configuration.

---

## Files That Must Be Inspected First

Before editing, inspect all relevant files that exist, especially:

```text
.github/workflows/
docker-compose.yml
docker-compose.production.yml
backend/
frontend/
gateway/
nginx/
Dockerfile
Dockerfile.*
.env.example
.env.template
.gitignore
```

Also inspect:

- Django `settings.py`;
- Django `urls.py`;
- backend entrypoint/start command;
- frontend API configuration;
- gateway Nginx configuration;
- production Docker Compose services, volumes, networks and ports.

---

## Environment Variables

Do not invent environment variable names.

Derive them from the real code.

Check:

- `docker-compose.production.yml`;
- Django settings;
- PostgreSQL configuration;
- Dockerfiles;
- entrypoint scripts;
- GitHub Actions workflow;
- existing `.env.example` or `.env.template`.

The production deployment must have a valid:

```text
~/taski/.env
```

If `.env.example` does not exist, create one containing every required variable but no real secrets.

Example structure only:

```env
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=
SECRET_KEY=
DEBUG=
ALLOWED_HOSTS=
```

The actual names must match the project.

Ensure `.env` itself is ignored by Git.

---

## GitHub Secrets

Secrets must not be committed.

Determine which secrets are required by the actual workflow.

Likely categories include:

```text
Docker Hub credentials
SSH host
SSH user
SSH private key
PostgreSQL credentials
Django secret key
Allowed hosts or domain configuration
```

Use the exact secret names already used by the workflow where possible.

If new secrets are required, document them clearly.

---

## GitHub Actions Requirements

For pushes to `main`, the workflow should:

1. run tests;
2. build required Docker images;
3. authenticate with Docker Hub;
4. push updated images;
5. connect to the remote server;
6. deploy the updated containers.

Inspect the current workflow before editing.

Do not rewrite the workflow from scratch unless necessary.

The deployment stage should explicitly use the production Compose file:

```bash
docker compose -f docker-compose.production.yml ...
```

Do not rely on an unrelated default Compose file.

---

## Production Deployment

The deployment should be safe and repeatable.

A typical flow may be:

```bash
cd ~/taski

docker compose -f docker-compose.production.yml pull

docker compose -f docker-compose.production.yml up -d

docker compose -f docker-compose.production.yml exec backend \
    python manage.py migrate

docker compose -f docker-compose.production.yml exec backend \
    python manage.py collectstatic --noinput

docker compose -f docker-compose.production.yml ps
```

This is only a model.

Use the real service names and real project requirements.

Do not add `down` automatically unless there is a clear reason. Prefer avoiding unnecessary downtime.

---

## Database Readiness

Check whether the backend can start before PostgreSQL is ready.

If this is possible, use the smallest reasonable fix.

Possible options include:

- PostgreSQL `healthcheck`;
- `depends_on` with a healthy condition where supported;
- an existing backend wait-for-database mechanism.

Do not introduce unnecessary third-party tooling if the project already has a mechanism.

---

## Django Migrations

Determine whether migrations are already executed by:

- an entrypoint;
- container startup command;
- GitHub Actions;
- manual server commands.

Avoid running migrations twice unnecessarily.

If not handled elsewhere, production deployment should run:

```bash
python manage.py migrate
```

inside the backend container.

---

## Static Files

Inspect Django static configuration.

The Practicum material uses a backend static path concept similar to:

```python
STATIC_URL = "/static_backend/"
STATIC_ROOT = BASE_DIR / "static_backend"
```

However, the containerized project may use Docker volumes and a gateway container instead of copying files manually into `/var/www/taski`.

Follow the current Docker architecture.

Do not blindly copy commands from the older non-containerized deployment instructions.

If `collectstatic` is needed, run it inside the backend container and ensure the resulting files are exposed through the correct volume or gateway configuration.

---

## Frontend

Inspect how the frontend calls the API.

Production frontend code must not depend on values such as:

```text
http://localhost:8000
```

or an obsolete hardcoded server IP/port.

If the architecture uses a reverse proxy, prefer relative routes such as:

```text
/api/
```

when consistent with the existing application.

Do not change frontend API behavior unless the current configuration is incorrect.

---

## Docker Gateway

If a `gateway` service exists, inspect it carefully.

Determine:

- whether it is an Nginx container;
- which container port it listens on;
- which host port is published;
- how `/api/` is proxied;
- how `/admin/` is proxied;
- how frontend files are served;
- how backend static files are served;
- whether media/static volumes are mounted.

The gateway is not the same thing as system Nginx on Ubuntu.

A likely architecture is:

```text
Internet
   |
   v
Ubuntu Nginx :80/:443
   |
   v
localhost:<gateway-host-port>
   |
   v
Docker gateway :80
   |
   +--> frontend
   +--> backend
```

Use the real port from `docker-compose.production.yml`.

---

## Host Nginx

The Ubuntu server-side Nginx must be installed and configured as part of one-time server setup.

Typical installation:

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Always validate configuration before reload:

```bash
sudo nginx -t
```

Only after a successful test:

```bash
sudo systemctl reload nginx
```

Do not place package installation inside every GitHub Actions deployment.

---

## Host Nginx Reverse Proxy

Do not blindly proxy to `127.0.0.1:8000`.

The older Practicum lesson used port `8000` because Gunicorn ran directly on the host.

In the containerized version, the host Nginx should normally proxy to the host port published by the Docker `gateway` service.

Example only:

```nginx
server {
    listen 80;
    server_name SERVER_IP_OR_DOMAIN;

    location / {
        proxy_pass http://127.0.0.1:GATEWAY_PORT;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Replace `GATEWAY_PORT` with the actual value from the production Compose file.

---

## Port Conflicts

Check whether Docker currently publishes its gateway directly on:

```text
0.0.0.0:80
```

If system Nginx also needs port 80, this causes a conflict.

Check with:

```bash
sudo ss -tulpn
```

or:

```bash
sudo lsof -i :80
```

If appropriate, expose the Docker gateway only on localhost, for example:

```yaml
ports:
  - "127.0.0.1:9000:80"
```

Then configure host Nginx to proxy to:

```text
127.0.0.1:9000
```

Use a different port if required by the actual project.

---

## Firewall

The assignment expects the server to allow:

```text
22  SSH
80  HTTP
443 HTTPS
```

Typical commands:

```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

Never enable UFW before ensuring SSH is allowed.

Avoid exposing internal PostgreSQL or backend container ports publicly.

---

## Old Host Gunicorn

Check whether an earlier non-Docker deployment left a host-level Gunicorn service.

Examples:

```bash
systemctl status gunicorn
systemctl list-units --type=service | grep -i gunicorn
```

If Docker now owns backend execution, an old host Gunicorn service may be obsolete or cause conflicts.

Do not disable anything blindly.

Confirm what is running first.

---

## Validation Commands

After changes, validate the whole deployment.

### Docker Compose

```bash
cd ~/taski
docker compose -f docker-compose.production.yml config
docker compose -f docker-compose.production.yml ps
```

### Containers

```bash
docker ps -a
```

### Backend logs

```bash
docker compose -f ~/taski/docker-compose.production.yml logs backend
```

### Gateway logs

```bash
docker compose -f ~/taski/docker-compose.production.yml logs gateway
```

### Database logs

Use the actual database service name:

```bash
docker compose -f ~/taski/docker-compose.production.yml logs db
```

### Host Nginx

```bash
sudo nginx -t
sudo systemctl status nginx
```

### Ports

```bash
sudo ss -tulpn
```

### HTTP

```bash
curl -I http://127.0.0.1
```

and, when appropriate:

```bash
curl -I http://PUBLIC_IP
```

---

## Application Routes

Inspect real Django URLs and gateway rules.

At minimum verify the routes required by the project, likely including:

```text
/
```

```text
/api/
```

```text
/admin/
```

Also verify static and media paths if applicable.

Do not assume an endpoint exists without checking `urls.py`.

---

## Safe Editing Rules

When making changes:

1. inspect first;
2. explain the root cause;
3. make the smallest necessary change;
4. preserve working parts of the project;
5. do not rename services or secrets unnecessarily;
6. do not commit real credentials;
7. do not replace the existing architecture without a clear requirement;
8. validate Docker Compose syntax;
9. validate Nginx syntax;
10. keep deployment repeatable.

---

## Do Not Do These

Do not:

- commit `.env`;
- commit private SSH keys;
- hardcode production passwords;
- hardcode Docker Hub passwords;
- hardcode Django `SECRET_KEY`;
- expose PostgreSQL publicly;
- expose backend ports publicly unless required;
- install Nginx on every deployment;
- run old host Gunicorn and Docker backend simultaneously without a reason;
- copy the old non-containerized Nginx configuration blindly;
- rewrite the whole CI/CD pipeline when a small fix is enough;
- assume service names such as `db`, `backend`, `frontend`, or `gateway` without checking the actual Compose file.

---

## Expected Agent Output

When asked to fix deployment, provide:

1. root cause;
2. files inspected;
3. files changed;
4. exact diff;
5. required GitHub Secrets;
6. required `.env.example`;
7. one-time server setup commands;
8. final Nginx configuration;
9. final deployment flow;
10. validation commands;
11. remaining manual steps, if any.

If something cannot be determined from the repository or assignment, state that explicitly instead of guessing.

---

## Final Goal

A successful solution should make this sequence work:

```bash
git push origin main
```

and result in:

- successful GitHub Actions;
- successful Docker image publication;
- successful SSH deployment;
- healthy PostgreSQL;
- healthy backend;
- healthy frontend;
- healthy gateway;
- working migrations;
- working static files;
- working host Nginx;
- working Taski through the public server address.

Keep the implementation aligned with the Yandex Practicum assignment and the existing project architecture.
