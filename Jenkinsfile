
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    parameters {
        choice(
            name: 'RELEASE_MODE',
            choices: ['STABLE', 'FAULTY_ROLLBACK_TEST'],
            description: 'Select stable deployment or automatic rollback test.'
        )

        booleanParam(
            name: 'DEPLOY_TO_KUBERNETES',
            defaultValue: true,
            description: 'Deploy the application to Kubernetes.'
        )
    }

    environment {
        DOCKERHUB_USERNAME = 'archanajawahar14'
        IMAGE_NAME = 'e-waste-portal'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS_ID = 'kubernetes-creds'
        K8S_DEPLOYMENT = 'e-waste-portal'
        K8S_CONTAINER = 'e-waste-portal'
        K8S_SERVICE = 'e-waste-portal'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                bat 'git log -1 --oneline'
                bat 'dir'
            }
        }

        stage('Parallel Validation') {
            parallel {

                stage('HTML Content Test') {
                    steps {
                        bat 'findstr /C:"Central Civic Collection Centre" index.html'
                        bat 'findstr /C:"Successfully recycled" index.html'
                        bat 'findstr /C:"scheduleForm" index.html'
                    }
                }

                stage('Kubernetes Manifest Test') {
                    steps {
                        bat 'findstr /C:"kind: Deployment" deployment.yaml'
                        bat 'findstr /C:"readinessProbe" deployment.yaml'
                        bat 'findstr /C:"kind: Service" deployment.yaml'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "v${env.BUILD_NUMBER}"

                    bat "docker build -t ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG} ."

                    bat "docker image inspect ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG} > image-inspect.json"
                }
            }
        }

        stage('Local Container Smoke Test') {
            steps {
                script {
                    bat 'docker rm -f e-waste-smoke >NUL 2>&1'
                    // The preceding cleanup may fail when no old container exists.
                }
            }
        }

        stage('Run Application Smoke Test') {
            steps {
                script {
                    bat '''
                        @echo off
                        docker rm -f e-waste-smoke >NUL 2>&1
                        exit /b 0
                    '''

                    bat "docker run -d --name e-waste-smoke -p 18087:80 ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    bat '''
                        @echo off
                        powershell -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0; $i -lt 15; $i++){ try { $r=Invoke-WebRequest -UseBasicParsing http://localhost:18087; if($r.StatusCode -eq 200 -and $r.Content.Contains('Central Civic Collection Centre')) { $ok=$true; break } } catch {}; Start-Sleep -Seconds 2 }; if(-not $ok){ docker logs e-waste-smoke; exit 1 }"
                        if errorlevel 1 exit /b 1
                    '''

                    bat 'docker rm -f e-waste-smoke'
                }
            }
        }

        stage('Archive Build Artifacts') {
            steps {
                archiveArtifacts(
                    artifacts: 'index.html,Dockerfile,Jenkinsfile,deployment.yaml,image-inspect.json',
                    fingerprint: true
                )
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    script {
                        int loginStatus = bat(
                            returnStatus: true,
                            script: '''
                                @echo off
                                powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; if ([string]::IsNullOrWhiteSpace($env:DH_USER)) { Write-Error 'Docker Hub username is missing'; exit 1 }; if ([string]::IsNullOrWhiteSpace($env:DH_TOKEN)) { Write-Error 'Docker Hub token is missing'; exit 1 }; $env:DH_TOKEN | docker login --username $env:DH_USER --password-stdin; if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }"
                            '''
                        )

                        if (loginStatus != 0) {
                            error('Docker Hub login failed. Verify Jenkins credential ID and Docker Hub access token.')
                        }

                        try {
                            bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                            if (params.RELEASE_MODE == 'STABLE') {
                                bat "docker tag ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG} ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:latest"
                                bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:latest"
                            }
                        } finally {
                            bat 'docker logout'
                        }
                    }
                }
            }
        }

        stage('Deploy and Verify on Kubernetes') {
            when {
                expression {
                    return params.DEPLOY_TO_KUBERNETES
                }
            }

            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: 'kubernetes-creds',
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        bat 'kubectl version --client'
                        bat 'kubectl config current-context'

                        bat 'kubectl apply -f deployment.yaml'

                        bat "kubectl set image deployment/${env.K8S_DEPLOYMENT} ${env.K8S_CONTAINER}=${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                        if (params.RELEASE_MODE == 'FAULTY_ROLLBACK_TEST') {
                            echo 'Applying an intentionally faulty readiness probe.'

                            writeFile(
                                file: 'faulty-patch.json',
                                text: '''{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "e-waste-portal",
            "readinessProbe": {
              "httpGet": {
                "path": "/faulty-health-check",
                "port": "http"
              },
              "initialDelaySeconds": 1,
              "periodSeconds": 3,
              "timeoutSeconds": 1,
              "failureThreshold": 2
            }
          }
        ]
      }
    }
  }
}'''
                            )

                            bat 'kubectl patch deployment e-waste-portal --type=strategic --patch-file=faulty-patch.json'

                            int faultyStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/e-waste-portal --timeout=90s'
                            )

                            if (faultyStatus == 0) {
                                echo 'Faulty rollout unexpectedly succeeded; restoring the previous revision.'

                                int undoStatus = bat(
                                    returnStatus: true,
                                    script: 'kubectl rollout undo deployment/e-waste-portal'
                                )

                                if (undoStatus != 0) {
                                    error('Rollback command failed.')
                                }

                                bat 'kubectl rollout status deployment/e-waste-portal --timeout=120s'

                                error('The faulty rollout unexpectedly passed. Check the readiness probe.')
                            }

                            echo 'Faulty rollout failure detected. Rolling back.'

                            bat 'kubectl get pods -l app=e-waste-portal -o wide'
                            bat 'kubectl describe deployment e-waste-portal'
                            bat 'kubectl rollout history deployment/e-waste-portal'

                            int undoStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout undo deployment/e-waste-portal'
                            )

                            if (undoStatus != 0) {
                                error('Automatic rollback command failed.')
                            }

                            int recoveryStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/e-waste-portal --timeout=120s'
                            )

                            if (recoveryStatus != 0) {
                                error('Rollback did not restore a healthy deployment.')
                            }

                            bat 'kubectl get deployment e-waste-portal'
                            bat 'kubectl get pods -l app=e-waste-portal'
                            bat 'kubectl get service e-waste-portal'

                            echo 'Rollback test completed: the failed rollout was detected and the previous revision recovered.'

                        } else {
                            int rolloutStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/e-waste-portal --timeout=120s'
                            )

                            if (rolloutStatus != 0) {
                                echo 'Stable deployment failed. Collecting diagnostics.'

                                bat 'kubectl get pods -l app=e-waste-portal -o wide'
                                bat 'kubectl describe deployment e-waste-portal'
                                bat 'kubectl rollout history deployment/e-waste-portal'

                                int undoStatus = bat(
                                    returnStatus: true,
                                    script: 'kubectl rollout undo deployment/e-waste-portal'
                                )

                                if (undoStatus != 0) {
                                    error('Deployment failed and rollback could not be initiated.')
                                }

                                bat 'kubectl rollout status deployment/e-waste-portal --timeout=120s'

                                error('Stable deployment failed; the previous revision was restored.')
                            }

                            bat 'kubectl get deployment e-waste-portal'
                            bat 'kubectl get pods -l app=e-waste-portal'
                            bat 'kubectl get service e-waste-portal'

                            echo 'Stable deployment completed successfully.'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            bat '''
                @echo off
                docker rm -f e-waste-smoke >NUL 2>&1
                exit /b 0
            '''

            echo 'Pipeline finished. Review stage logs and deployment status.'
        }

        success {
            echo 'Jenkins pipeline completed successfully.'
        }

        failure {
            echo 'Jenkins pipeline failed. Check the first failed stage in Console Output.'
        }
    }
}
