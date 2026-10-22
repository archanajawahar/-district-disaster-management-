
pipeline {
    agent any

    parameters {
        choice(
            name: 'RELEASE_MODE',
            choices: ['STABLE', 'FAULTY_ROLLBACK_TEST'],
            description: 'Deploy stable release or demonstrate rollback'
        )
        booleanParam(
            name: 'DEPLOY_TO_KUBERNETES',
            defaultValue: true,
            description: 'Deploy to Docker Desktop Kubernetes'
        )
    }

    environment {
        DOCKER_IMAGE = 'archanajawahar14/disaster-response-portal'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS_ID = 'kubernetes-creds'
        K8S_DEPLOYMENT = 'disaster-response-portal'
        K8S_CONTAINER = 'disaster-response-portal'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 25, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Parallel Validation') {
            parallel {
                stage('Validate HTML') {
                    steps {
                        powershell '''
                            $html = Get-Content index.html -Raw

                            $required = @(
                                'INC-001',
                                'Central Community Hall',
                                'CRITICAL OCCUPANCY ALERT',
                                'severityFilter',
                                'openCount'
                            )

                            foreach ($item in $required) {
                                if (-not $html.Contains($item)) {
                                    throw "Missing required HTML content: $item"
                                }
                            }

                            Write-Host "HTML validation passed."
                        '''
                    }
                }

                stage('Validate Kubernetes Manifest') {
                    steps {
                        powershell '''
                            $yaml = Get-Content deployment.yaml -Raw

                            if ($yaml -notmatch 'kind: Deployment') {
                                throw "Deployment definition missing."
                            }
                            if ($yaml -notmatch 'kind: Service') {
                                throw "Service definition missing."
                            }
                            if ($yaml -notmatch 'readinessProbe') {
                                throw "Readiness probe missing."
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
                    env.IMAGE_TAG = "v${BUILD_NUMBER}"
                }

                bat 'docker build -t %DOCKER_IMAGE%:%IMAGE_TAG% .'
                bat 'docker image inspect %DOCKER_IMAGE%:%IMAGE_TAG%'
            }
        }

        stage('Local Container Smoke Test') {
            steps {
                powershell '''
                    $name = "disaster-smoke-$env:BUILD_NUMBER"

                    docker rm -f $name 2>$null
                    docker run -d --name $name -p 18087:80 "$env:DOCKER_IMAGE`:$env:IMAGE_TAG"

                    try {
                        $passed = $false

                        for ($i = 0; $i -lt 12; $i++) {
                            try {
                                $response = Invoke-WebRequest `
                                    -Uri "http://localhost:18087" `
                                    -UseBasicParsing `
                                    -TimeoutSec 3

                                if ($response.StatusCode -eq 200 -and
                                    $response.Content.Contains("Incident Records")) {
                                    $passed = $true
                                    break
                                }
                            } catch {
                                Start-Sleep -Seconds 2
                            }
                        }

                        if (-not $passed) {
                            throw "Local Docker smoke test failed."
                        }

                        Write-Host "Local Docker smoke test passed."
                    }
                    finally {
                        docker rm -f $name 2>$null
                    }
                '''
            }
        }

        
        
        
stage('Push Image to Docker Hub') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'dockerhub-creds',
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            )
        ]) {
            powershell '''
                $ErrorActionPreference = 'Stop'

                $dockerUser = $env:DOCKER_USER
                $dockerPass = $env:DOCKER_PASS

                if ([string]::IsNullOrWhiteSpace($dockerUser) -or
                    [string]::IsNullOrEmpty($dockerPass)) {
                    throw "Docker Hub credentials are missing from Jenkins."
                }

                $dockerPass | docker login `
                    --username $dockerUser `
                    --password-stdin

                if ($LASTEXITCODE -ne 0) {
                    throw "Docker Hub login failed."
                }

                Write-Host "Docker Hub login successful."
            '''

            bat 'docker push %DOCKER_IMAGE%:%IMAGE_TAG%'

            powershell '''
                docker logout
            '''
        }
    }
}




        stage('Deploy and Verify') {
            when {
                expression { return params.DEPLOY_TO_KUBERNETES }
            }

            steps {
                withCredentials([
                    file(
                        credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    powershell '''
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes apply failed."
                        }

                        kubectl set image `
                            deployment/$env:K8S_DEPLOYMENT `
                            $env:K8S_CONTAINER="$env:DOCKER_IMAGE`:$env:IMAGE_TAG"

                        if ($LASTEXITCODE -ne 0) {
                            throw "Unable to update the deployment image."
                        }

                        if ($env:RELEASE_MODE -eq 'FAULTY_ROLLBACK_TEST') {
                            Write-Host "Introducing a deliberately faulty readiness probe."

                            kubectl patch deployment $env:K8S_DEPLOYMENT `
                                --type=json `
                                -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/invalid-health-check"}]'

                            if ($LASTEXITCODE -ne 0) {
                                throw "Could not introduce faulty probe."
                            }

                            kubectl rollout status `
                                deployment/$env:K8S_DEPLOYMENT `
                                --timeout=45s

                            if ($LASTEXITCODE -eq 0) {
                                throw "Faulty rollout unexpectedly succeeded."
                            }

                            Write-Host "Failed rollout detected. Starting rollback."

                            kubectl rollout undo deployment/$env:K8S_DEPLOYMENT

                            if ($LASTEXITCODE -ne 0) {
                                throw "Automatic rollback failed."
                            }
                        }

                        kubectl rollout status `
                            deployment/$env:K8S_DEPLOYMENT `
                            --timeout=180s

                        if ($LASTEXITCODE -ne 0) {
                            throw "Deployment did not recover successfully."
                        }

                        kubectl get deployment $env:K8S_DEPLOYMENT
                        kubectl get pods -l app=disaster-response-portal
                        kubectl get service disaster-response-portal-service

                        $service = kubectl get service disaster-response-portal-service -o json
                        $serviceObject = $service | ConvertFrom-Json
                        $nodePort = $serviceObject.spec.ports[0].nodePort

                        $response = Invoke-WebRequest `
                            -Uri "http://localhost:$nodePort" `
                            -UseBasicParsing `
                            -TimeoutSec 10

                        if ($response.StatusCode -ne 200) {
                            throw "Restored application HTTP verification failed."
                        }

                        Write-Host "Application HTTP verification passed."
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'index.html,Dockerfile,Jenkinsfile,deployment.yaml',
                allowEmptyArchive: true
            )
        }

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}

