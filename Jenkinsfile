pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    environment {
        SONAR_SCANNER_HOME = tool 'SonarQube Scanner'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh """
                    ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=todo-ui-devsecops \
                    -Dsonar.projectName=Todo-UI-DevSecOps \
                    -Dsonar.sources=src \
                    -Dsonar.language=js
                    """
                }
            }
        }
    }
}
