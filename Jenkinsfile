
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
            description: 'Deploy to Kubernetes after successful image push.'
        )
    }

    environment {
        DOCKERHUB_USERNAME = 'archanajawahar14'
        IMAGE_NAME = 'disaster-response-portal'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS_ID = 'kubernetes-creds'
        K8S_DEPLOYMENT = 'disaster-response-portal'
        K8S_CONTAINER = 'disaster-response-portal'
        K8S_SERVICE = 'disaster-response-portal'
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
                        powershell '''
                            $ErrorActionPreference = 'Stop'

                            if (!(Test-Path "index.html")) {
                                throw "index.html is missing."
                            }

                            $html = Get-Content "index.html" -Raw

                            if ($html.Length -eq 0) {
                                throw "index.html is empty."
                            }

                            Write-Host "HTML validation passed."
                        '''
                    }
                }

                stage('Validate Kubernetes Manifest') {
                    steps {
                        powershell '''
                            $ErrorActionPreference = 'Stop'

                            if (!(Test-Path "deployment.yaml")) {
                                throw "deployment.yaml is missing."
                            }

                            $yaml = Get-Content "deployment.yaml" -Raw

                            foreach ($required in @(
                                "kind: Deployment",
                                "kind: Service",
                                "readinessProbe"
                            )) {
                                if (!$yaml.Contains($required)) {
                                    throw "Missing Kubernetes manifest field: $required"
                                }
                            }

                            Write-Host "Manifest validation passed."
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
                powershell '''
                    $ErrorActionPreference = 'Stop'

                    docker rm -f disaster-smoke 2>$null
                    $LASTEXITCODE = 0

                    docker run -d --name disaster-smoke -p 18087:80 "$env:DOCKERHUB_USERNAME/$env:IMAGE_NAME:$env:IMAGE_TAG"

                    if ($LASTEXITCODE -ne 0) {
                        throw "Unable to start the smoke-test container."
                    }

                    try {
                        $ok = $false

                        for ($i = 0; $i -lt 15; $i++) {
                            try {
                                $response = Invoke-WebRequest `
                                    -UseBasicParsing `
                                    -Uri "http://localhost:18087"

                                if ($response.StatusCode -eq 200) {
                                    $ok = $true
                                    break
                                }
                            } catch {
                                Start-Sleep -Seconds 2
                            }

                            Start-Sleep -Seconds 2
                        }

                        if (!$ok) {
                            docker logs disaster-smoke
                            throw "Local smoke test failed."
                        }

                        Write-Host "Local Docker smoke test passed."
                    }
                    finally {
                        docker rm -f disaster-smoke 2>$null
                    }
                '''
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
                        int loginStatus = powershell(
                            returnStatus: true,
                            script: '''
                                $ErrorActionPreference = 'Stop'

                                if ([string]::IsNullOrWhiteSpace($env:DH_USER)) {
                                    Write-Error "Docker Hub username is empty."
                                    exit 1
                                }

                                if ([string]::IsNullOrWhiteSpace($env:DH_TOKEN)) {
                                    Write-Error "Docker Hub token is empty."
                                    exit 1
                                }

                                $env:DH_TOKEN |
                                    docker login --username $env:DH_USER --password-stdin

                                if ($LASTEXITCODE -ne 0) {
                                    exit $LASTEXITCODE
                                }

                                Write-Host "Docker Hub authentication succeeded."
                            '''
                        )

                        if (loginStatus != 0) {
                            error('Docker Hub login failed. Verify the Jenkins credential binding and valid token.')
                        }

                        try {
                            bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                            bat "docker tag ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG} ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:latest"

                            bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:latest"
                        } finally {
                            powershell 'docker logout'
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

                        bat "kubectl set image deployment/${env.K8S_DEPLOYMENT} ${env.K8S_CONTAINER}=${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                        if (params.RELEASE_MODE == 'FAULTY_ROLLBACK_TEST') {
                            echo 'Applying intentionally faulty readiness probe.'

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

                            bat "kubectl patch deployment ${env.K8S_DEPLOYMENT} --type=strategic --patch-file=faulty-patch.json"

                            int faultyStatus = bat(
                                returnStatus: true,
                                script: "kubectl rollout status deployment/${env.K8S_DEPLOYMENT} --timeout=90s"
                            )

                            if (faultyStatus == 0) {
                                echo 'Faulty rollout unexpectedly became ready.'
                            } else {
                                echo 'Faulty rollout failure detected.'
                            }

                            bat "kubectl rollout undo deployment/${env.K8S_DEPLOYMENT}"

                            int recoveryStatus = bat(
                                returnStatus: true,
                                script: "kubectl rollout status deployment/${env.K8S_DEPLOYMENT} --timeout=120s"
                            )

                            if (recoveryStatus != 0) {
                                error('Rollback recovery failed. Inspect Kubernetes rollout history and pod events.')
                            }

                            bat "kubectl get deployment ${env.K8S_DEPLOYMENT}"
                            bat "kubectl get pods -l app=disaster-response-portal"
                            bat "kubectl get service ${env.K8S_SERVICE}"

                            if (faultyStatus == 0) {
                                error('Recovery completed, but the intentionally faulty rollout did not fail. Check the readiness probe.')
                            }

                            echo 'Rollback test passed: failed rollout detected and previous revision restored.'
                        } else {
                            int rolloutStatus = bat(
                                returnStatus: true,
                                script: "kubectl rollout status deployment/${env.K8S_DEPLOYMENT} --timeout=120s"
                            )

                            if (rolloutStatus != 0) {
                                echo 'Stable rollout failed. Collecting diagnostics.'

                                bat "kubectl get pods -l app=disaster-response-portal -o wide"
                                bat "kubectl describe deployment ${env.K8S_DEPLOYMENT}"
                                bat "kubectl rollout history deployment/${env.K8S_DEPLOYMENT}"

                                int undoStatus = bat(
                                    returnStatus: true,
                                    script: "kubectl rollout undo deployment/${env.K8S_DEPLOYMENT}"
                                )

                                if (undoStatus != 0) {
                                    error('Deployment failed and rollback could not be initiated.')
                                }

                                bat "kubectl rollout status deployment/${env.K8S_DEPLOYMENT} --timeout=120s"

                                error('Stable deployment failed. Previous revision was restored.')
                            }

                            bat "kubectl get deployment ${env.K8S_DEPLOYMENT}"
                            bat "kubectl get pods -l app=disaster-response-portal"
                            bat "kubectl get service ${env.K8S_SERVICE}"

                            echo 'Stable deployment completed successfully.'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            powershell '''
                docker rm -f disaster-smoke 2>$null
                exit 0
            '''

            echo 'Pipeline finished. Review the stage logs and deployment status.'
        }
    }
}
