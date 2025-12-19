pipeline {
    agent any

    tools {
        nodejs 'node18'     
        sonarScanner 'SonarQube Scanner'
    }

    environment {
        SONAR_PROJECT_KEY = 'todo-ui-devsecops'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                  node -v
                  npm -v
                  npm install
                '''
            }
        }

        stage('Build UI') {
            steps {
                sh '''
                  npm run build
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                      sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=node_modules/**,dist/**
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

    }

    post {
        success {
            echo '✅ Pipeline completed successfully. Quality Gate PASSED.'
        }
        failure {
            echo '❌ Pipeline failed. Check SonarQube Quality Gate.'
        }
    }
}
