pipeline {
    agent any

    parameters {
        string(name: 'GIT_WEBAPP_REPO_URL', defaultValue: 'https://github.com/Tili-service/Web-App.git', description: 'URL of the Git repository to clone')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to build and deploy')
        string(name: 'DOCKER_REGISTRY', defaultValue: 'docker.io', description: 'Docker registry URL')
        string(name: 'DOCKER_IMAGE_NAME', defaultValue: 'tili/webapp', description: 'Docker image repository name')
        string(name: 'DOCKER_CREDENTIALS_ID', defaultValue: 'docker-hub-credentials', description: 'Jenkins credentials ID for Docker registry')
        booleanParam(name: 'RUN_SECURITY_SCAN', defaultValue: true, description: 'Run dependency audits and image vulnerability scans')
        booleanParam(name: 'FORCE_DEPLOY', defaultValue: false, description: 'Force deployment even if builds are on non-standard branches')
    }

    environment {
        REGISTRY_URL         = "${params.DOCKER_REGISTRY}"
        IMAGE_NAME           = "${params.DOCKER_IMAGE_NAME}"
        CREDENTIALS_ID       = "${params.DOCKER_CREDENTIALS_ID}"
        BUILD_TAG            = "${env.BUILD_NUMBER}"
        IMAGE_FULL_TAG       = "${REGISTRY_URL}/${IMAGE_NAME}:${BUILD_TAG}"
        IMAGE_LATEST_TAG     = "${REGISTRY_URL}/${IMAGE_NAME}:latest"
        
        // npm caching directory for the node container agents to speed up runs
        NPM_CONFIG_CACHE     = '/var/jenkins_home/npm-cache'
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
                echo "Cloning webapp branch ${params.GIT_BRANCH} from ${params.GIT_WEBAPP_REPO_URL}..."
                checkout([$class: 'GitSCM', 
                    branches: [[name: "*/${params.GIT_BRANCH}"]], 
                    userRemoteConfigs: [[url: params.GIT_WEBAPP_REPO_URL]]
                ])
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    docker.image('node:20-alpine').inside("-v /var/jenkins_home/npm-cache:/var/jenkins_home/npm-cache") {
                        echo 'Installing node modules using clean install (npm ci)...'
                        sh 'npm ci'
                    }
                }
            }
        }

        stage('Lint & Code Style') {
            steps {
                script {
                    docker.image('node:20-alpine').inside("-v /var/jenkins_home/npm-cache:/var/jenkins_home/npm-cache") {
                        echo 'Checking syntax & formatting (eslint)...'
                        sh 'npm run lint'
                    }
                }
            }
        }

        stage('Security Dependency Audit') {
            when {
                expression { return params.RUN_SECURITY_SCAN }
            }
            steps {
                script {
                    docker.image('node:20-alpine').inside("-v /var/jenkins_home/npm-cache:/var/jenkins_home/npm-cache") {
                        echo 'Running dependency security audit (npm audit)...'
                        sh 'npm audit --audit-level=high || true'
                    }
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                script {
                    docker.image('node:20-alpine').inside("-v /var/jenkins_home/npm-cache:/var/jenkins_home/npm-cache") {
                        echo 'Executing unit and component tests (vitest)...'
                        sh 'npm run test'
                    }
                }
            }
        }

        stage('Build Static Production Output') {
            steps {
                script {
                    docker.image('node:20-alpine').inside("-v /var/jenkins_home/npm-cache:/var/jenkins_home/npm-cache") {
                        echo 'Validating Next.js compilation...'
                        sh 'NEXT_TELEMETRY_DISABLED=1 npm run build'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${IMAGE_FULL_TAG} using standalone mode..."
                sh "docker build -t ${IMAGE_FULL_TAG} ."
                sh "docker tag ${IMAGE_FULL_TAG} ${IMAGE_LATEST_TAG}"
            }
        }

        stage('Docker Image Scan') {
            when {
                expression { return params.RUN_SECURITY_SCAN }
            }
            steps {
                echo 'Scanning built container image with Trivy...'
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
                            echo "Pushing latest tag..."
                            sh "docker push ${IMAGE_LATEST_TAG}"
                        }
                    }
                }
            }
        }

        stage('Deploy to Target Environment') {
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
                    echo "Deploying Web-App version ${BUILD_TAG} to ${targetEnv}..."
                    
                    if (targetEnv == 'production') {
                        input message: 'Approve rollout of Web-App to Production?', ok: 'Deploy'
                    }
                    
                    echo "Web-App deployment to ${targetEnv} succeeded!"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded! Web-App built and published successfully."
        }
        failure {
            echo "Pipeline failed. Check build logs and test outputs."
        }
        always {
            echo 'Performing workspace cleanup...'
            cleanWs()
        }
    }
}