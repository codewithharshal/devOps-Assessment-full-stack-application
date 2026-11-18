# DevOps Runbook

This document explains how to run, validate, and deploy the application. It is written for a Windows development machine using Docker Desktop, Jenkins, Docker Hub, and a local Jenkins deployment agent.

## 1. Setup Guide

### Run locally

Install Docker Desktop, Git, and Docker Compose. From the repository root, verify the tools:

```powershell
docker version
docker compose version
git --version
```

Start both application services:

```powershell
docker compose --env-file frontend/.env up --build
```

The application is then available at:

```text
Frontend: http://localhost:5173/
Backend:  http://localhost:8000/api/hello/
```

Stop the local deployment with:

```powershell
docker compose down
```

### Run on the deployment server

The deployment server is the machine running the Jenkins agent. It must have Docker Desktop or Docker Engine, Docker Compose, Git, and network access to Docker Hub.

1. Configure a Jenkins agent with the label `local-docker-deploy`.
2. Give the agent access to Docker and Docker Compose.
3. Create protected `.env` files on the deployment agent from `backend/.env.example` and `frontend/.env.example`.
4. Configure the Docker Hub credential described in the Jenkins section below.
5. Create a Multibranch Pipeline that loads the repository `Jenkinsfile`.
6. Configure the pipeline to discover the `main` branch.
7. Run the pipeline manually or trigger it with a GitHub webhook.

The Jenkins pipeline builds images tagged with the commit SHA, pushes them to Docker Hub, and starts the same Compose services on the deployment server:

```text
codewithharshal/devops-backend:<git-sha>
codewithharshal/devops-frontend:<git-sha>
```

After a successful deployment, verify the server:

```powershell
docker ps
docker compose ps
```

Then check:

```text
http://localhost:5173/
http://localhost:8000/api/hello/
```

## Application layout

- `backend/`: Django API, built on port `8000`.
- `frontend/`: React/Vite application served by unprivileged Nginx on container port `8080`.
- `docker-compose.yml`: Runs both services together.
- `Jenkinsfile`: Builds, publishes, and deploys images when the `main` branch pipeline runs.

## Prerequisites

Install and verify the following on the machine that runs Docker and the Jenkins agent:

```powershell
docker version
docker compose version
git --version
```

Also verify that Docker can run a container:

```powershell
docker run --rm hello-world
```

The Jenkins agent must run as a Windows agent because the `Jenkinsfile` uses Jenkins `bat` steps.

## Run locally with Docker Compose

From the repository root:

```powershell
docker compose --env-file frontend/.env up --build
```

Compose will:

1. Build the backend and frontend images.
2. Run Django migrations before starting Gunicorn.
3. Wait for the backend health check.
4. Start the frontend after the backend is healthy.

Open the services at:

- Frontend: `http://localhost:5173/`
- Backend API: `http://localhost:8000/api/hello/`

Stop the services with:

```powershell
docker compose down
```

The current SQLite database is inside the backend container and is not persisted by a Compose volume. This is acceptable for the assessment application, but a production deployment should use a persistent database and storage strategy.

## Build and run images separately

Build the backend:

```powershell
docker build -t devops-backend ./backend
docker run --rm -p 8000:8000 devops-backend
```

Build the frontend:

```powershell
docker build --build-arg VITE_API_URL=http://localhost:8000 -t devops-frontend ./frontend
docker run --rm -p 5173:8080 devops-frontend
```

The `-p` options are required. `EXPOSE` documents a container port but does not publish it to the host.

## Configuration boundaries

The application uses separate ignored `.env` files for each service. Create them from the safe templates before running Compose:

```powershell
Copy-Item backend\.env.example backend\.env
Copy-Item frontend\.env.example frontend\.env
```

Replace `DJANGO_SECRET_KEY` in `backend/.env` with a long random value. Compose passes `backend/.env` into the backend container at runtime, while `frontend/.env` supplies `VITE_API_URL` as a frontend Docker build argument. Never commit `.env` or put private secrets in `VITE_*` variables because Vite embeds those values into browser JavaScript.

The service-level `.env.example` files document the required variables without containing real secrets. Production deployments should provide these values through a deployment secret store or protected server-side `.env` files.

## Jenkins CI/CD

The root `Jenkinsfile` performs these stages for the `main` branch:

1. Checkout the repository.
2. Validate Docker Compose configuration.
3. Build backend and frontend images.
4. Run `python manage.py check` inside the backend image.
5. Push both images to Docker Hub.
6. Deploy the pushed images locally with Docker Compose.

Images are published under the Docker Hub account `codewithharshal`:

```text
codewithharshal/devops-backend:<git-sha>
codewithharshal/devops-frontend:<git-sha>
```

The Git commit SHA is used as the image tag so every deployment can be traced and rolled back to a specific commit.

### Jenkins agent setup

Create or configure a Jenkins agent with:

```text
Node label:              local-docker-deploy
Remote root directory:  C:\Jenkins\agent
```

The agent account must be able to run Docker and Docker Compose. Verify this using the same Windows account that runs the Jenkins agent:

```powershell
docker version
docker compose version
git --version
docker run --rm hello-world
```

If Docker works in a personal PowerShell window but fails in Jenkins, the Jenkins service account usually lacks Docker access or has a different `PATH`.

### Docker Hub credential

Create a Docker Hub access token and add a Jenkins credential under:

```text
Manage Jenkins -> Credentials -> System -> Global credentials
```

Use:

```text
Kind:     Username with password
Username: codewithharshal
Password: Docker Hub access token
ID:       docker
```

The Jenkinsfile refers to this credential by ID. The token must not be committed to the repository.

### Jenkins Pipeline job

Create a **Multibranch Pipeline** job:

1. Add the Git repository as the branch source.
2. Configure Git credentials if the repository is private.
3. Enable discovery of the `main` branch.
4. Set the script path to `Jenkinsfile`.
5. Save and run **Scan Multibranch Pipeline Now**.
6. Open the discovered `main` branch job and select **Build Now**.

The job must use an agent with the `local-docker-deploy` label.

## 2. Troubleshooting Log

### GitHub personal access token could not update a resource

**Symptom:** Jenkins reported:

```text
Resource not accessible by personal access token
```

**Cause:** The GitHub personal access token used by Jenkins did not have permission to update commit statuses for the repository. This is a GitHub credential permission problem, separate from the Docker Hub credential.

**Resolution:** Update the GitHub token used by Jenkins so it has access to the repository. For a fine-grained token, grant repository access and enable the required permissions, including:

- Contents: Read
- Metadata: Read
- Commit statuses: Read and write

For a classic token, use the repository access scope required by the Jenkins GitHub integration. Replace the Jenkins credential with the updated token and rerun the pipeline.

The Docker Hub credential is separate and must continue to use the Jenkins credential ID `docker`.

### `Could not find credentials entry with ID 'docker'`

Confirm that a credential with the exact ID `docker` exists under the Jenkins **Global credentials** store. The credential must be a username/password credential, with the Docker Hub access token as the password.

### `Resource not accessible by personal access token`

This usually comes from Jenkins trying to update a GitHub commit status. Give the GitHub token repository permission to read contents and write commit statuses, or disable commit-status publishing if it is not needed. This is separate from the Docker Hub credential.

### Browser cannot reach `localhost:8000` or `localhost:5173`

Check that the containers are running and that the ports are published:

```powershell
docker ps
```

Expected mappings include:

```text
0.0.0.0:8000->8000/tcp
0.0.0.0:5173->8080/tcp
```

Use `localhost` or `127.0.0.1` in the browser. `0.0.0.0` is only a server bind address.

### Nginx reports it cannot modify `default.conf`

The unprivileged Nginx image may print a warning because its non-root user cannot modify the default configuration. If Nginx ends with `ready for start up` and starts worker processes, the warning is non-fatal.

### Jenkins cannot run Docker commands

Run the Docker checks as the Jenkins agent account. On Windows, restart the Jenkins agent after installing Docker Desktop or changing the system `PATH`. Confirm Docker Desktop is running and configured for Linux containers.

## Production limitations

This setup is suitable for a small assessment or local deployment. Before using it for production, add:

- A persistent database such as PostgreSQL.
- Persistent media and static-file storage where required.
- Environment-based secrets and settings.
- HTTPS and a domain name.
- Container restart and backup policies.
- Security hardening for Django, CORS, and allowed hosts.
- A protected deployment agent and restricted Docker access.
