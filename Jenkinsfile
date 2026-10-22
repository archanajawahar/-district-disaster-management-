
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
            description: 'Choose stable deployment or test automatic rollback.'
        )

        booleanParam(
            name: 'DEPLOY_TO_KUBERNETES',
            defaultValue: true,
            description: 'Deploy to Kubernetes after successful image push.'
        )
    }

    environment {
        DOCKERHUB_USERNAME = 'archanajawahar14'
        IMAGE_NAME = 'e-waste-portal'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS_ID = 'kubernetes-creds'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                bat 'git log -1 --oneline'
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

                stage('Deployment Manifest Test') {
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
                    bat '''
                        @echo off
                        docker rm -f e-waste-smoke >NUL 2>&1
                        exit /b 0
                    '''

                    bat "docker run -d --name e-waste-smoke -p 18087:80 ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    bat '''
                        @echo off
                        powershell -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0;$i -lt 15;$i++){try{$r=Invoke-WebRequest -UseBasicParsing http://localhost:18087; if($r.StatusCode -eq 200 -and $r.Content.Contains('Central Civic Collection Centre')){$ok=$true;break}}catch{}; Start-Sleep -Seconds 2}; if(-not $ok){docker logs e-waste-smoke; exit 1}"
                        if errorlevel 1 exit /b 1
                    '''

                    bat '''
                        @echo off
                        docker rm -f e-waste-smoke
                        exit /b 0
                    '''
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
powershell.exe -NoProfile -NonInteractive -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $u=$env:DH_USER.Trim(); $p=$env:DH_TOKEN; if ([string]::IsNullOrWhiteSpace($u) -or [string]::IsNullOrWhiteSpace($p)) { Write-Error 'Credential binding is empty'; exit 1 }; $p | docker login --username $u --password-stdin; exit $LASTEXITCODE"
'''
                )

                if (loginStatus != 0) {
                    error("Docker Hub login failed with exit code ${loginStatus}.")
                }

                try {
                    bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
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
                        bat 'kubectl apply -f deployment.yaml'

                        bat "kubectl set image deployment/e-waste-portal e-waste-portal=${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

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

                            int faultyRolloutStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/e-waste-portal --timeout=90s'
                            )

                            if (faultyRolloutStatus == 0) {
                                echo 'The faulty deployment unexpectedly became ready.'

                                int undoStatus = bat(
                                    returnStatus: true,
                                    script: 'kubectl rollout undo deployment/e-waste-portal'
                                )

                                if (undoStatus != 0) {
                                    error('Rollback command failed.')
                                }

                                bat 'kubectl rollout status deployment/e-waste-portal --timeout=120s'

                                error('Rollback recovery succeeded, but the faulty rollout unexpectedly passed. Review the probe configuration.')
                            }

                            echo 'Faulty rollout failed as expected. Attempting automatic rollback.'

                            bat 'kubectl get pods -l app=e-waste-portal -o wide'
                            bat 'kubectl describe deployment e-waste-portal'
                            bat 'kubectl rollout history deployment/e-waste-portal'

                            int undoStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout undo deployment/e-waste-portal'
                            )

                            if (undoStatus != 0) {
                                error('Rollback command failed. Check Kubernetes access and deployment history.')
                            }

                            int recoveryStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/e-waste-portal --timeout=120s'
                            )

                            if (recoveryStatus != 0) {
                                error('Rollback was requested, but the previous revision did not become healthy.')
                            }

                            bat 'kubectl get deployment e-waste-portal'
                            bat 'kubectl get pods -l app=e-waste-portal'
                            bat 'kubectl get service e-waste-portal'

                            echo 'Rollback test passed: the failed rollout was detected and the previous revision recovered.'

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

                                error('Stable deployment failed. Previous revision was restored.')
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

            echo 'Pipeline finished. Review the stage logs and deployment status.'
        }

        success {
            echo 'Jenkins pipeline completed successfully.'
        }

        failure {
            echo 'Jenkins pipeline failed. Check the first failed stage in Console Output.'
        }
    }
}
