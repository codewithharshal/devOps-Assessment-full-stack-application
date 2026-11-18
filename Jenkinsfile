pipeline {
    agent { label 'local-docker-deploy' }

    triggers {
        githubPush()
    }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        DOCKERHUB_NAMESPACE = 'codewithharshal'
        DOCKERHUB_CREDENTIALS_ID = 'docker'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate') {
            when {
                branch 'main'
            }
            steps {
                bat 'docker compose config'
            }
        }

        stage('Build Images') {
            when {
                branch 'main'
            }
            steps {
                script {
                    env.IMAGE_TAG = bat(
                        returnStdout: true,
                        script: '@git rev-parse --short HEAD'
                    ).trim()
                    env.BACKEND_IMAGE = "${env.DOCKERHUB_NAMESPACE}/devops-backend:${env.IMAGE_TAG}"
                    env.FRONTEND_IMAGE = "${env.DOCKERHUB_NAMESPACE}/devops-frontend:${env.IMAGE_TAG}"
                }
                bat 'docker build -t "%BACKEND_IMAGE%" ./backend'
                bat 'docker build -t "%FRONTEND_IMAGE%" ./frontend'
                bat 'docker run --rm "%BACKEND_IMAGE%" python manage.py check'
            }
        }

        stage('Push Images') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    bat 'echo %DOCKER_TOKEN%| docker login --username %DOCKER_USER% --password-stdin'
                    bat 'docker push "%BACKEND_IMAGE%"'
                    bat 'docker push "%FRONTEND_IMAGE%"'
                    bat 'docker logout'
                }
            }
        }

        stage('Deploy Locally') {
            when {
                branch 'main'
            }
            steps {
                withEnv([
                    "BACKEND_IMAGE=${env.BACKEND_IMAGE}",
                    "FRONTEND_IMAGE=${env.FRONTEND_IMAGE}"
                ]) {
                    bat 'docker compose pull'
                    bat 'docker compose up -d --no-build --remove-orphans'
                }
            }
        }
    }

}
