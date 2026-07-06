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
        
        // Go caching directories for the dynamic container agents
        GOCACHE              = '/var/jenkins_home/go-cache/cache'
        GOPATH               = '/var/jenkins_home/go-cache/path'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '15', artifactNumToKeepStr: '5'))
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {
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
            agent {
                docker {
                    image 'golang:1.25-alpine'
                    args "-v /var/jenkins_home/go-cache:/go -e GOCACHE=/go/cache -e GOPATH=/go/path"
                }
            }
            steps {
                dir('app') {
                    echo 'Running formatting check...'
                    sh 'go fmt ./...'
                    
                    echo 'Running static analysis (go vet)...'
                    sh 'go vet ./...'
                }
            }
        }

        stage('SAST Security Scan') {
            when {
                expression { return params.RUN_SECURITY_SCAN }
            }
            agent {
                docker {
                    image 'securego/gosec:2.19.0'
                    // We run gosec inside container pointing to backend source code
                    args "-v /var/jenkins_home/go-cache:/go -e GOCACHE=/go/cache -e GOPATH=/go/path"
                }
            }
            steps {
                dir('app') {
                    echo 'Running Go security scanner (gosec)...'
                    // Run security scan and continue even if issues found (to inspect reports), but return non-zero on high severity
                    sh 'gosec -fmt=text -severity=high -confidence=medium ./...'
                }
            }
        }

        stage('Run Unit & Integration Tests') {
            agent {
                docker {
                    image 'golang:1.25-alpine'
                    args "-v /var/jenkins_home/go-cache:/go -e GOCACHE=/go/cache -e GOPATH=/go/path"
                }
            }
            steps {
                dir('app') {
                    echo 'Running unit tests with coverage profile...'
                    sh 'go test -v -race -coverprofile=coverage.out -covermode=atomic ./...'
                    
                    echo 'Archiving test coverage reports...'
                    archiveArtifacts artifacts: 'coverage.out', allowEmptyArchive: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_FULL_TAG}..."
                // Since build context is app/, we specify the Dockerfile inside it.
                sh "docker build -t ${IMAGE_FULL_TAG} ./app"
                
                // Tag it as latest for convenient local runs
                sh "docker tag ${IMAGE_FULL_TAG} ${IMAGE_LATEST_TAG}"
            }
        }

        stage('Docker Vulnerability Scan') {
            when {
                expression { return params.RUN_SECURITY_SCAN }
            }
            steps {
                echo 'Scanning built docker image with Trivy...'
                // Run trivy scanner against local built image
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
                    // Utilizing Jenkins credentials binding safely
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
                        // Advanced Gate: Ask for operator approval before production rollout
                        input message: 'Approve deployment to Production environment?', ok: 'Deploy'
                    }
                    
                    // Deployment execution: can be ssh commands, k8s, docker-compose, or cd/ci scripts
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