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
   pip install django django-cors-headers psycopg2-binary
   ```
4. Run the development server:
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
3. Run the development server:
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

This is a small assessment application with fixed local development settings, so it intentionally does not use a `.env` file. Adding environment-variable loading would introduce extra configuration and deployment complexity without providing meaningful value for this simple example.

The current application does not provide:

- Environment-driven Django settings such as `DEBUG`, `SECRET_KEY`, or `ALLOWED_HOSTS`.
- A configurable frontend `API_URL`; the React client currently calls `http://localhost:8000/api/hello/` directly.
- Separate development, staging, and production configuration profiles.
- Secret management or production database configuration.

For a larger or production application, these values should be moved to environment variables or a secrets manager. In that case, `.env` files may be useful for local development, while production secrets should be supplied by the deployment platform rather than committed to the repository.

## Docker

Build and run the backend and frontend as separate images from their respective directories:

```bash
docker build -t devops-backend ./backend
docker run --rm -p 8000:8000 devops-backend

docker build -t devops-frontend ./frontend
docker run --rm -p 5173:8080 devops-frontend
```

Both images use multi-stage builds. The backend runs as the `app` user, while the frontend uses the non-root `nginx-unprivileged` image on port `8080`.

To build and run both services together with Docker Compose:

```bash
docker compose up --build
```

The frontend is available at `http://localhost:5173/`, and the backend is available at `http://localhost:8000/api/hello/`. Compose runs the Django migrations before starting Gunicorn and waits for the backend health check before starting the frontend.

## Jenkins CI/CD

The root `Jenkinsfile` defines the CI/CD workflow for pushes to the `main` branch. It validates the Compose file, builds both Docker images, pushes commit-tagged images to Docker Hub under the `codewithharshal` namespace, and deploys those exact images on a Jenkins agent labelled `local-docker-deploy`.

Before creating the Jenkins job, configure:

1. A Jenkins agent with the label `local-docker-deploy`, Docker, and Docker Compose installed.
2. A Jenkins username/password credential with ID `dockerhub-credentials`. Use the Docker Hub access token as the password.
3. A Pipeline job that loads `Jenkinsfile` from the repository and tracks the `main` branch.
4. A GitHub webhook pointing to Jenkins so pushes trigger the Pipeline.

The deployment uses the commit SHA as the image tag, which makes each deployment traceable and allows rollback to an earlier image tag.
