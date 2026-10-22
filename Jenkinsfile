
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
            description: 'Choose normal deployment or test automatic rollback.'
        )

        booleanParam(
            name: 'DEPLOY_TO_KUBERNETES',
            defaultValue: true,
            description: 'Deploy to Docker Desktop Kubernetes.'
        )
    }

    environment {
        DOCKERHUB_USERNAME = 'archanajawahar14'
        IMAGE_NAME = 'disaster-response-portal'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS_ID = 'kubernetes-creds'
        DEPLOYMENT_NAME = 'disaster-response-portal'
        CONTAINER_NAME = 'disaster-response-portal'
        SERVICE_NAME = 'disaster-response-portal'
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
                stage('HTML Validation') {
                    steps {
                        bat '''
                        powershell -NoProfile -Command "$ErrorActionPreference='Stop'; if (!(Test-Path 'index.html')) { throw 'index.html missing' }; $c=Get-Content 'index.html' -Raw; if ($c.Length -lt 100) { throw 'index.html is empty or incomplete' }; if ($c -notmatch '(?i)incident|emergency|disaster') { throw 'Expected disaster portal content not found' }; Write-Host 'HTML validation passed'"
                        '''
                    }
                }

                stage('Kubernetes Manifest Validation') {
                    steps {
                        bat '''
                        powershell -NoProfile -Command "$ErrorActionPreference='Stop'; if (!(Test-Path 'deployment.yaml')) { throw 'deployment.yaml missing' }; $c=Get-Content 'deployment.yaml' -Raw; if ($c -notmatch 'kind:\\s*Deployment') { throw 'Deployment resource missing' }; if ($c -notmatch 'kind:\\s*Service') { throw 'Service resource missing' }; if ($c -notmatch 'readinessProbe') { throw 'readinessProbe missing' }; Write-Host 'Manifest validation passed'"
                        '''
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
                    bat 'docker rm -f disaster-smoke >NUL 2>&1 || exit /b 0'

                    bat "docker run -d --name disaster-smoke -p 18087:80 ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    bat '''
                    powershell -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0;$i -lt 15;$i++){try{$r=Invoke-WebRequest -UseBasicParsing http://localhost:18087; if($r.StatusCode -eq 200 -and $r.Content -match '(?i)incident|emergency|disaster'){$ok=$true;break}}catch{}; Start-Sleep -Seconds 2}; if(-not $ok){docker logs disaster-smoke; exit 1}; Write-Host 'Container smoke test passed'"
                    '''

                    bat 'docker rm -f disaster-smoke'
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
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    script {
                        int loginStatus = bat(
                            returnStatus: true,
                            script: '''
                            @echo off
                            powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $env:DH_TOKEN | docker login --username $env:DH_USER --password-stdin; if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }"
                            '''
                        )

                        if (loginStatus != 0) {
                            error('Docker Hub login failed. Check the username and access token stored in Jenkins credentials.')
                        }

                        try {
                            bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                            bat "docker tag ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG} ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:latest"
                            bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:latest"
                        } finally {
                            bat 'docker logout'
                        }
                    }
                }
            }
        }

        stage('Deploy and Verify on Kubernetes') {
            when {
                expression { return params.DEPLOY_TO_KUBERNETES }
            }

            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        bat 'kubectl version --client'
                        bat 'kubectl apply -f deployment.yaml'

                        bat "kubectl set image deployment/${env.DEPLOYMENT_NAME} ${env.CONTAINER_NAME}=${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                        if (params.RELEASE_MODE == 'FAULTY_ROLLBACK_TEST') {
                            echo 'Testing a deliberately unhealthy readiness probe.'

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
                "port": 80
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

                            bat "kubectl patch deployment/${env.DEPLOYMENT_NAME} --type=strategic --patch-file=faulty-patch.json"

                            int faultyStatus = bat(
                                returnStatus: true,
                                script: "kubectl rollout status deployment/${env.DEPLOYMENT_NAME} --timeout=90s"
                            )

                            if (faultyStatus == 0) {
                                echo 'Faulty rollout unexpectedly became ready; restoring the previous revision.'
                            } else {
                                echo 'Faulty rollout failure detected. Starting rollback.'
                            }

                            int undoStatus = bat(
                                returnStatus: true,
                                script: "kubectl rollout undo deployment/${env.DEPLOYMENT_NAME}"
                            )

                            if (undoStatus != 0) {
                                error('Automatic rollback command failed.')
                            }

                            int recoveryStatus = bat(
                                returnStatus: true,
                                script: "kubectl rollout status deployment/${env.DEPLOYMENT_NAME} --timeout=120s"
                            )

                            if (recoveryStatus != 0) {
                                error('Rollback was requested but recovery did not become healthy.')
                            }

                            bat "kubectl get deployment ${env.DEPLOYMENT_NAME}"
                            bat "kubectl get pods -l app=${env.DEPLOYMENT_NAME}"
                            bat "kubectl get service ${env.SERVICE_NAME}"

                            if (faultyStatus == 0) {
                                error('Rollback recovered the deployment, but the faulty rollout unexpectedly passed. Check the readiness probe and manifest.')
                            }

                            echo 'Rollback test passed: failed rollout detected and previous revision recovered.'
                        } else {
                            int rolloutStatus = bat(
                                returnStatus: true,
                                script: "kubectl rollout status deployment/${env.DEPLOYMENT_NAME} --timeout=120s"
                            )

                            if (rolloutStatus != 0) {
                                bat "kubectl get pods -l app=${env.DEPLOYMENT_NAME} -o wide"
                                bat "kubectl describe deployment ${env.DEPLOYMENT_NAME}"

                                int undoStatus = bat(
                                    returnStatus: true,
                                    script: "kubectl rollout undo deployment/${env.DEPLOYMENT_NAME}"
                                )

                                if (undoStatus != 0) {
                                    error('Stable rollout failed and rollback could not be initiated.')
                                }

                                bat "kubectl rollout status deployment/${env.DEPLOYMENT_NAME} --timeout=120s"
                                error('Stable rollout failed. Previous revision was restored.')
                            }

                            bat "kubectl get deployment ${env.DEPLOYMENT_NAME}"
                            bat "kubectl get pods -l app=${env.DEPLOYMENT_NAME}"
                            bat "kubectl get service ${env.SERVICE_NAME}"

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
            echo 'Pipeline completed. Check Console Output for stage results.'
        }
    }
}
