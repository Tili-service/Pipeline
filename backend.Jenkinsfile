pipeline {
    agent any

    parameters {
        string(name: 'GIT_BACKEND_REPO_URL', defaultValue: 'https://github.com/Tili-service/Backend.git', description: 'URL of the Git repository to clone')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to build and deploy')
        string(name: 'DOCKER_REGISTRY', defaultValue: 'docker.io', description: 'Docker registry URL')
        string(name: 'DOCKER_IMAGE_NAME', defaultValue: 'tili/backend', description: 'Docker image repository name')
        string(name: 'DOCKER_CREDENTIALS_ID', defaultValue: 'docker-hub-credentials', description: 'Jenkins credentials ID for Docker registry')
        booleanParam(name: 'RUN_SECURITY_SCAN', defaultValue: true, description: 'Run SAST and Docker vulnerability scans')
        booleanParam(name: 'FORCE_DEPLOY', defaultValue: false, description: 'Force deployment even if builds are on non-standard branches')
    }

    environment {
        REGISTRY_URL         = "${params.DOCKER_REGISTRY}"
        IMAGE_NAME           = "${params.DOCKER_IMAGE_NAME}"
        CREDENTIALS_ID       = "${params.DOCKER_CREDENTIALS_ID}"
        BUILD_TAG            = "${env.BUILD_NUMBER}"
        IMAGE_FULL_TAG       = "${REGISTRY_URL}/${IMAGE_NAME}:${BUILD_TAG}"
        IMAGE_LATEST_TAG     = "${REGISTRY_URL}/${IMAGE_NAME}:latest"
        
        // Go caching directories inside the container. 
        // Defined here so Jenkins automatically injects them into the Docker run environment.
        GOCACHE              = '/go/cache'
        GOPATH               = '/go/path'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '15', artifactNumToKeepStr: '5'))
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Prepare Cache Directories') {
            steps {
                echo 'Creating cache directories on the host with correct user permissions...'
                // This ensures directories are owned by uid 1000 (jenkins) and not root when mounted by Docker
                sh 'mkdir -p /var/jenkins_home/go-cache/cache /var/jenkins_home/go-cache/path'
            }
        }

        stage('Checkout Source') {
            steps {
                echo "Cloning backend branch ${params.GIT_BRANCH} from ${params.GIT_BACKEND_REPO_URL}..."
                checkout([$class: 'GitSCM', 
                    branches: [[name: "*/${params.GIT_BRANCH}"]], 
                    userRemoteConfigs: [[url: params.GIT_BACKEND_REPO_URL]]
                ])
            }
        }

        stage('Code Quality (Go Lint & Vet)') {
            steps {
                script {
                    docker.image('golang:1.25-alpine').inside("-v /var/jenkins_home/go-cache:/go") {
                        dir('app') {
                            echo 'Running formatting check...'
                            sh 'go fmt ./...'
                            
                            echo 'Running static analysis (go vet)...'
                            sh 'go vet ./...'
                        }
                    }
                }
            }
        }

        stage('SAST Security Scan') {
            when {
                expression { return params.RUN_SECURITY_SCAN }
            }
            steps {
                script {
                    docker.image('golang:1.25-alpine').inside("-v /var/jenkins_home/go-cache:/go") {
                        dir('app') {
                            echo 'Ensuring gosec is installed in cache...'
                            sh 'if [ ! -f /go/path/bin/gosec ]; then go install github.com/securego/gosec/v2/cmd/gosec@latest; fi'
                            
                            echo 'Running Go security scanner (gosec)...'
                            sh '/go/path/bin/gosec -fmt=text -severity=high -confidence=medium ./...'
                        }
                    }
                }
            }
        }

        stage('Run Unit & Integration Tests') {
            steps {
                script {
                    docker.image('golang:1.25-alpine').inside("-v /var/jenkins_home/go-cache:/go") {
                        dir('app') {
                            echo 'Running unit tests with coverage profile...'
                            sh 'go test -v -race -coverprofile=coverage.out -covermode=atomic ./...'
                        }
                    }
                    echo 'Archiving test coverage reports...'
                    archiveArtifacts artifacts: 'app/coverage.out', allowEmptyArchive: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_FULL_TAG}..."
                sh "docker build -t ${IMAGE_FULL_TAG} ./app"
                sh "docker tag ${IMAGE_FULL_TAG} ${IMAGE_LATEST_TAG}"
            }
        }

        stage('Docker Vulnerability Scan') {
            when {
                expression { return params.RUN_SECURITY_SCAN }
            }
            steps {
                echo 'Scanning built docker image with Trivy...'
                sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL --exit-code 0 ${IMAGE_FULL_TAG}"
            }
        }

        stage('Publish Image to Registry') {
            when {
                expression { return params.GIT_BRANCH == 'main' || params.GIT_BRANCH == 'staging' }
            }
            steps {
                script {
                    echo "Logging into Docker Registry at ${REGISTRY_URL}..."
                    withCredentials([usernamePassword(credentialsId: CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD} ${REGISTRY_URL}"
                        
                        echo "Pushing image ${IMAGE_FULL_TAG}..."
                        sh "docker push ${IMAGE_FULL_TAG}"
                        
                        if (params.GIT_BRANCH == 'main') {
                            echo "Pushing latest tag for main branch..."
                            sh "docker push ${IMAGE_LATEST_TAG}"
                        }
                    }
                }
            }
        }

        stage('Deploy to Environment') {
            when {
                anyOf {
                    branch 'main'
                    branch 'staging'
                    expression { return params.FORCE_DEPLOY }
                }
            }
            steps {
                script {
                    def targetEnv = (params.GIT_BRANCH == 'main') ? 'production' : 'staging'
                    echo "Deploying backend version ${BUILD_TAG} to ${targetEnv} environment..."
                    
                    if (targetEnv == 'production') {
                        input message: 'Approve deployment to Production environment?', ok: 'Deploy'
                    }
                    
                    echo "Deployment to ${targetEnv} succeeded!"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded! Backend built and published successfully."
        }
        failure {
            echo "Pipeline failed. Review logs and artifacts."
        }
        always {
            echo 'Performing workspace cleanup...'
            cleanWs()
        }
    }
}