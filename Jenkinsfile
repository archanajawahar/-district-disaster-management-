
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 25, unit: 'MINUTES')
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
            description: 'Deploy to Kubernetes after a successful image push.'
        )
    }

    environment {
        DOCKERHUB_USERNAME = 'archanajawahar14'
        DOCKER_IMAGE = 'archanajawahar14/disaster-response-portal'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS_ID = 'kubernetes-creds'
        K8S_DEPLOYMENT = 'disaster-response-portal'
        K8S_CONTAINER = 'disaster-response-portal'
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
                stage('Validate HTML') {
                    steps {
                        bat 'findstr /C:"INC-001" index.html'
                        bat 'findstr /C:"Central Community Hall" index.html'
                        bat 'findstr /C:"CRITICAL OCCUPANCY ALERT" index.html'
                        bat 'findstr /C:"severityFilter" index.html'
                        bat 'findstr /C:"openCount" index.html'
                    }
                }

                stage('Validate Kubernetes Manifest') {
                    steps {
                        bat 'findstr /C:"kind: Deployment" deployment.yaml'
                        bat 'findstr /C:"kind: Service" deployment.yaml'
                        bat 'findstr /C:"readinessProbe" deployment.yaml'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "v${env.BUILD_NUMBER}"

                    bat "docker build -t ${env.DOCKER_IMAGE}:${env.IMAGE_TAG} ."

                    bat "docker image inspect ${env.DOCKER_IMAGE}:${env.IMAGE_TAG} > image-inspect.json"
                }
            }
        }

        stage('Local Container Smoke Test') {
            steps {
                script {
                    bat 'docker rm -f disaster-smoke >NUL 2>&1 || exit /b 0'

                    bat "docker run -d --name disaster-smoke -p 18087:80 ${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"

                    bat '''
                    @echo off
                    powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0;$i -lt 15;$i++){try{$r=Invoke-WebRequest -UseBasicParsing http://localhost:18087; if($r.StatusCode -eq 200 -and $r.Content.Contains('Incident Records')){$ok=$true;break}}catch{}; Start-Sleep -Seconds 2}; if(-not $ok){docker logs disaster-smoke; exit 1}"
                    if errorlevel 1 exit /b 1
                    '''

                    echo 'Local Docker smoke test passed.'
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
                            powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; if ([string]::IsNullOrWhiteSpace($env:DH_USER) -or [string]::IsNullOrEmpty($env:DH_TOKEN)) { Write-Error 'Docker credentials are missing'; exit 1 }; $env:DH_TOKEN | docker login --username $env:DH_USER --password-stdin; if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }"
                            '''
                        )

                        if (loginStatus != 0) {
                            error('Docker Hub login failed. Check the Jenkins credential and Docker Hub access token.')
                        }

                        try {
                            bat "docker push ${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"

                            echo "Docker image pushed successfully: ${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"
                        } finally {
                            bat(
                                returnStatus: true,
                                script: 'docker logout'
                            )
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

                        bat "kubectl set image deployment/${env.K8S_DEPLOYMENT} ${env.K8S_CONTAINER}=${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"

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
            "name": "disaster-response-portal",
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

                            bat 'kubectl patch deployment disaster-response-portal --type=strategic --patch-file=faulty-patch.json'

                            int faultyRolloutStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/disaster-response-portal --timeout=90s'
                            )

                            if (faultyRolloutStatus == 0) {
                                echo 'The faulty deployment unexpectedly became ready.'

                                int undoStatus = bat(
                                    returnStatus: true,
                                    script: 'kubectl rollout undo deployment/disaster-response-portal'
                                )

                                if (undoStatus != 0) {
                                    error('The rollback command failed.')
                                }

                                bat 'kubectl rollout status deployment/disaster-response-portal --timeout=120s'

                                error('The faulty rollout unexpectedly passed. The previous revision was restored.')
                            }

                            echo 'Faulty rollout failed as expected. Attempting rollback.'

                            bat 'kubectl get pods -l app=disaster-response-portal -o wide'
                            bat 'kubectl describe deployment disaster-response-portal'
                            bat 'kubectl rollout history deployment/disaster-response-portal'

                            int undoStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout undo deployment/disaster-response-portal'
                            )

                            if (undoStatus != 0) {
                                error('Rollback command failed. Check Kubernetes access and deployment history.')
                            }

                            int recoveryStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/disaster-response-portal --timeout=120s'
                            )

                            if (recoveryStatus != 0) {
                                error('Rollback was requested, but the previous revision did not become healthy.')
                            }

                            bat 'kubectl get deployment disaster-response-portal'
                            bat 'kubectl get pods -l app=disaster-response-portal'
                            bat 'kubectl get service disaster-response-portal'

                            echo 'Rollback test passed: the failed rollout was detected and the previous revision recovered.'
                        } else {
                            int rolloutStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/disaster-response-portal --timeout=120s'
                            )

                            if (rolloutStatus != 0) {
                                echo 'Stable deployment failed. Collecting diagnostics.'

                                bat 'kubectl get pods -l app=disaster-response-portal -o wide'
                                bat 'kubectl describe deployment disaster-response-portal'
                                bat 'kubectl rollout history deployment/disaster-response-portal'

                                int undoStatus = bat(
                                    returnStatus: true,
                                    script: 'kubectl rollout undo deployment/disaster-response-portal'
                                )

                                if (undoStatus != 0) {
                                    error('Deployment failed and rollback could not be initiated.')
                                }

                                int recoveryStatus = bat(
                                    returnStatus: true,
                                    script: 'kubectl rollout status deployment/disaster-response-portal --timeout=120s'
                                )

                                if (recoveryStatus != 0) {
                                    error('The previous deployment revision did not become healthy.')
                                }

                                error('Stable deployment failed. The previous revision was restored.')
                            }

                            bat 'kubectl get deployment disaster-response-portal'
                            bat 'kubectl get pods -l app=disaster-response-portal'
                            bat 'kubectl get service disaster-response-portal'

                            echo 'Stable deployment completed successfully.'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f disaster-smoke >NUL 2>&1 || exit /b 0'
            echo 'Pipeline finished. Review the stage logs and deployment status.'
        }
    }
}
