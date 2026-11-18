# DevOps Assessment Application

A simple "Hello World" full-stack application built with **Django** (Backend) and **React with Vite** (Frontend).

## Project Overview

- **Backend**: Django 6.0 (REST API)
- **Frontend**: React (Vite, TypeScript, Lucide Icons)
- **Styling**: Premium custom CSS with dark/light mode support.
- **Communication**: REST API using Axios with CORS enabled.

## Getting Started

### Prerequisites

- Python 3.10+
- Node.js 18+
- npm 9+

### Backend Setup (Django)

1. Navigate to the backend directory:
   ```bash
   cd backend
   ```
2. Create and activate a virtual environment:
   ```bash
   python3 -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Create the backend environment file:
   ```powershell
   Copy-Item .env.example .env
   ```
   Replace `DJANGO_SECRET_KEY` in `backend/.env` with a long random value.
5. Run the development server:
   ```bash
   python manage.py runserver
   ```
   The backend will be available at `http://localhost:8000/api/hello/`.

### Frontend Setup (React/Vite)

1. Navigate to the frontend directory:
   ```bash
   cd frontend
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Create the frontend environment file:
   ```powershell
   Copy-Item .env.example .env
   ```
4. Run the development server:
   ```bash
   npm run dev
   ```
   The frontend will be available at `http://localhost:5173/`.

## Architecture Decisions

- **Vite**: Used for its superior development experience and fast build times.
- **Django**: Chosen for its robustness and ease of setting up a structured API.
- **CORS**: Configured in Django to allow the React frontend to fetch data during local development.
- **Responsive Design**: Custom CSS ensures the application looks premium on all screen sizes and supports dark mode.

## Configuration Scope

Runtime configuration is supplied through separate ignored `.env` files. Copy each folder's example file before starting the application:

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

On Windows PowerShell, use:

```powershell
Copy-Item backend\.env.example backend\.env
Copy-Item frontend\.env.example frontend\.env
```

The `.env` files are never committed. `DJANGO_SECRET_KEY` is injected into the backend container at runtime. `VITE_API_URL` is passed from `frontend/.env` to the frontend Docker build as a build argument. Do not put private secrets in `VITE_*` variables because frontend values are visible in the browser.

## Docker

Build and run the backend and frontend as separate images from their respective directories:

```bash
docker build -t devops-backend ./backend
docker run --rm -p 8000:8000 devops-backend

docker build --build-arg VITE_API_URL=http://localhost:8000 -t devops-frontend ./frontend
docker run --rm -p 5173:8080 devops-frontend
```

Both images use multi-stage builds. The backend runs as the `app` user, while the frontend uses the non-root `nginx-unprivileged` image on port `8080`.

To build and run both services together with Docker Compose:

```powershell
docker compose --env-file frontend/.env up --build
```

The frontend is available at `http://localhost:5173/`, and the backend is available at `http://localhost:8000/api/hello/`. Compose runs the Django migrations before starting Gunicorn and waits for the backend health check before starting the frontend.

## Jenkins CI/CD

The root `Jenkinsfile` defines the CI/CD workflow for pushes to the `main` branch. It validates the Compose file, builds both Docker images, pushes commit-tagged images to Docker Hub under the `codewithharshal` namespace, and deploys those exact images on a Jenkins agent labelled `local-docker-deploy`.

Before creating the Jenkins job, configure:

1. A Jenkins agent with the label `local-docker-deploy`, Docker, and Docker Compose installed.
2. A Jenkins username/password credential with ID `docker`. Use the Docker Hub access token as the password.
3. A Pipeline job that loads `Jenkinsfile` from the repository and tracks the `main` branch.

The deployment uses the commit SHA as the image tag, which makes each deployment traceable and allows rollback to an earlier image tag.
