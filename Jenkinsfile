
pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        IMAGE_NAME = 'bbhavanasree/food-ordering'
    }

    stages {
        stage('Checkout and Test') {
            steps {
                checkout scm

                bat '''
                    if not exist index.html exit /b 1
                    if not exist Dockerfile exit /b 1
                    if not exist deployment.yaml exit /b 1
                    findstr /I /C:"<html" index.html
                    if errorlevel 1 exit /b 1
                    findstr /I /C:"FROM nginx" Dockerfile
                    if errorlevel 1 exit /b 1
                '''
                echo 'Application files and basic checks passed.'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    docker build -t %IMAGE_NAME%:stable .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    bat '''
                        @echo off
                        echo %DOCKER_PASS% | docker login -u "%DOCKER_USER%" --password-stdin
                        if errorlevel 1 exit /b 1

                        docker push %IMAGE_NAME%:stable
                        if errorlevel 1 exit /b 1

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-creds',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        @echo off
                        set "KUBECONFIG=%KUBECONFIG_FILE%"

                        kubectl config current-context
                        if errorlevel 1 exit /b 1

                        kubectl apply -f deployment.yaml
                        if errorlevel 1 exit /b 1

                        kubectl set image deployment/food-ordering food-ordering=%IMAGE_NAME%:stable
                        if errorlevel 1 exit /b 1

                        kubectl rollout status deployment/food-ordering --timeout=180s
                        if errorlevel 1 exit /b 1

                        kubectl get deployments
                        kubectl get pods -l app=food-ordering
                        kubectl get service food-ordering-active
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Build, Docker Hub push, and Kubernetes deployment succeeded.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output for the failing stage.'
        }
    }
}
