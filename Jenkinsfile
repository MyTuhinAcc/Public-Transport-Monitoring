
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

                    if not exist project\\index.html exit /b 1
                    if not exist project\\Dockerfile exit /b 1
                    if not exist project\\deployment.yaml exit /b 1
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

            REM Allow Nginx time to initialize
            powershell -NoProfile -Command "Start-Sleep -Seconds 10"

            REM Retry HTTP requests until application responds
            curl.exe -fsS --retry 10 --retry-all-errors --retry-delay 3 http://127.0.0.1:%TEST_PORT%/ -o test-page.html

            if errorlevel 1 (
                echo APPLICATION TEST FAILED
                docker logs %TEST_CONTAINER%
                exit /b 1
            )

            REM Verify the expected webpage content
            findstr /I /C:"%EXPECTED_TEXT%" test-page.html

            if errorlevel 1 (
                echo HTML CONTENT VALIDATION FAILED
                type test-page.html
                exit /b 1
            )

            echo APPLICATION TEST SUCCESSFUL
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

                        REM Verify the current stable deployment
                        kubectl get deployment %APP_NAME%
                        if errorlevel 1 exit /b 1

                        kubectl rollout status deployment/%APP_NAME% --timeout=180s
                        if errorlevel 1 exit /b 1

                        echo CURRENT STABLE IMAGE:
                        kubectl get deployment/%APP_NAME% -o jsonpath="{.spec.template.spec.containers[0].image}"
                        if errorlevel 1 exit /b 1

                        REM Introduce a deliberately faulty release
                        kubectl set image deployment/%APP_NAME% %CONTAINER_NAME%=%DOCKER_IMAGE%:invalid-rollback-demo
                        if errorlevel 1 exit /b 1

                        echo Testing deliberately faulty release...

                        REM Wait for Kubernetes to report rollout failure
                        kubectl rollout status deployment/%APP_NAME% --timeout=60s

                        if errorlevel 1 (
                            echo FAULTY RELEASE DETECTED
                            echo Starting automatic rollback...

                            kubectl rollout undo deployment/%APP_NAME%
                            if errorlevel 1 exit /b 1

                            kubectl rollout status deployment/%APP_NAME% --timeout=180s
                            if errorlevel 1 exit /b 1

                            echo AUTOMATIC ROLLBACK SUCCESSFUL

                            kubectl get deployment %APP_NAME%
                            kubectl get pods -l app=%APP_NAME%
                            kubectl get service %SERVICE_NAME%

                            REM Mark the faulty release as rejected
                            exit /b 1
                        )

                        echo ERROR: Faulty image unexpectedly deployed
                        exit /b 1
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
            echo 'Pipeline failed or faulty release rejected.'
            echo 'Check the rollback messages in Console Output.'
        }

        always {
            bat 'docker rm -f %TEST_CONTAINER% 2>NUL || exit /b 0'
        }
    }
}
