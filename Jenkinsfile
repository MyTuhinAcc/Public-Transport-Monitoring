
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        GIT_REPO = 'https://github.com/MyTuhinAcc/Public-Transport-Monitoring.git'
        DOCKER_IMAGE = 'tuhinnew/public-transport'
        APP_NAME = 'public-transport'
        CONTAINER_NAME = 'public-transport'
        SERVICE_NAME = 'public-transport-service'
        TEST_PORT = '18085'
        TEST_CONTAINER = 'transport-jenkins-test'
        EXPECTED_TEXT = 'Public Transport Monitoring'
    }

    stages {

        stage('Clone Code') {
            steps {
                bat '''
                    if exist project rmdir /s /q project
                    git clone --branch main "%GIT_REPO%" project
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Parallel Validation') {
            parallel {

                stage('HTML Check') {
                    steps {
                        bat '''
                            findstr /I /C:"%EXPECTED_TEXT%" project\\index.html
                            if errorlevel 1 exit /b 1
                        '''
                    }
                }

                stage('Dockerfile Check') {
                    steps {
                        bat '''
                            if not exist project\\Dockerfile exit /b 1
                            if not exist project\\deployment.yaml exit /b 1
                            findstr /I /C:"FROM nginx" project\\Dockerfile
                            if errorlevel 1 exit /b 1
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('project') {
                    bat '''
                        docker build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Test Application') {
            steps {
                bat '''
                    docker rm -f %TEST_CONTAINER% 2>NUL

                    docker run -d --name %TEST_CONTAINER% -p %TEST_PORT%:80 %DOCKER_IMAGE%:%BUILD_NUMBER%
                    if errorlevel 1 exit /b 1

                    curl.exe -fsS --retry 10 --retry-delay 2 --retry-connrefused http://127.0.0.1:%TEST_PORT%/ -o test-page.html
                    if errorlevel 1 exit /b 1

                    findstr /I /C:"%EXPECTED_TEXT%" test-page.html
                    if errorlevel 1 exit /b 1
                '''

                archiveArtifacts artifacts: 'test-page.html',
                                 fingerprint: true
            }

            post {
                always {
                    bat 'docker rm -f %TEST_CONTAINER% 2>NUL || exit /b 0'
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'ccc0439c-efa7-42a7-86c4-31f11069fea5',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    bat '''
                        powershell -NoProfile -Command "$env:DOCKER_PASS | docker login -u $env:DOCKER_USER --password-stdin"
                        if errorlevel 1 exit /b 1

                        docker push %DOCKER_IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'config',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        set "KUBECONFIG=%KUBECONFIG_FILE%"

                        kubectl apply -f project/deployment.yaml
                        if errorlevel 1 exit /b 1

                        kubectl set image deployment/%APP_NAME% %CONTAINER_NAME%=%DOCKER_IMAGE%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        kubectl rollout status deployment/%APP_NAME% --timeout=180s

                        if errorlevel 1 (
                            echo FAILED ROLLOUT DETECTED
                            echo Attempting automatic rollback...

                            kubectl rollout undo deployment/%APP_NAME%
                            if errorlevel 1 exit /b 1

                            kubectl rollout status deployment/%APP_NAME% --timeout=180s
                            if errorlevel 1 exit /b 1

                            echo AUTOMATIC ROLLBACK SUCCESSFUL
                            exit /b 1
                        )

                        kubectl get deployment %APP_NAME%
                        kubectl get pods -l app=%APP_NAME%
                        kubectl get service %SERVICE_NAME%
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'BUILD AND DEPLOYMENT SUCCESSFUL'
        }

        failure {
            echo 'Pipeline failed. Inspect the failed stage.'
        }

        always {
            bat 'docker rm -f %TEST_CONTAINER% 2>NUL || exit /b 0'
        }
    }
}
