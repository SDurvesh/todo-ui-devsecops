pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    environment {
        SONAR_SCANNER_HOME = tool 'SonarQube Scanner'
        APP_URL = 'http://4.240.60.209:4173'
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

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Start App for DAST') {
            steps {
                sh '''
                  nohup npm run preview -- --host 0.0.0.0 --port 4173 > app.log 2>&1 &
                  sleep 20
                '''
            }
        }

        stage('OWASP ZAP DAST (Report Only)') {
            steps {
                sh '''
                  /snap/zaproxy/current/zap-baseline.py \
                    -t ${APP_URL} \
                    -r zap-report.html || true
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
            sh 'pkill -f "npm run preview" || true'
        }

        success {
            echo '✅ Pipeline completed successfully (SAST + DAST)'
        }

        failure {
            echo '❌ Pipeline failed (Check SonarQube Quality Gate)'
        }
    }
}
