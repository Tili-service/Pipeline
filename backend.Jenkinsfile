pipeline {
    agent any
    
    parameters {
        string(name: 'GIT_BACKEND_REPO_URL', defaultValue: 'https://github.com/Tili-service/Backend.git', description: 'URL of the Git (http) repository to clone')
    }
    
    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: params.GIT_BACKEND_REPO_URL
            }
        }
        
        stage('Build Backend Image') {
            steps {
                sh 'docker compose build backend'              
            }
        }
    }
    
    post {
        always {
            echo 'Nettoyage du workspace...'
            cleanWs()
        }
    }
}