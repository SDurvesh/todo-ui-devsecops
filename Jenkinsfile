pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    environment {
        SONAR_SCANNER_HOME = tool 'SonarQube Scanner'
        APP_URL = 'http://4.240.60.209:4173'
        SAST_STATUS = 'NOT_RUN'
        DAST_STATUS = 'NOT_RUN'
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
                script {
                    try {
                        timeout(time: 2, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                        }
                    env.SAST_STATUS = 'PASSED'
                    } catch (err) {
                        env.SAST_STATUS = 'FAILED'
                        throw err
                    }
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
                script {
                    sh '''
                    zaproxy -cmd \
                    -port 8091 \
                    -quickurl http://4.240.60.209:4173 \
                    -quickout /var/lib/jenkins/workspace/todo-ui-devsecops/zap-report.html || true
                    '''
                env.DAST_STATUS = 'COMPLETED'
                }
            }
        }
        stage('Fetch SonarQube Report') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                    echo "Fetching SonarQube Quality Gate..."
                    curl -s -u $SONAR_TOKEN: \
                    "http://4.240.60.209:9000/api/qualitygates/project_status?projectKey=todo-ui-devsecops" \
                    > sonar-quality-gate.json

                    echo "Fetching SonarQube Metrics..."
                    curl -s -u $SONAR_TOKEN: \
                    "http://4.240.60.209:9000/api/measures/component?component=todo-ui-devsecops&metricKeys=bugs,vulnerabilities,code_smells,coverage,duplicated_lines_density,ncloc" \
                    > sonar-metrics.json
                    '''
                }
            }
        }
        stage('Parse SonarQube Metrics') {
            steps {
                sh '''
                echo "Parsing SonarQube reports..."

                QUALITY_GATE=$(jq -r '.projectStatus.status' sonar-quality-gate.json)

                BUGS=$(jq -r '.component.measures[] | select(.metric=="bugs") | .value' sonar-metrics.json)
                VULNERABILITIES=$(jq -r '.component.measures[] | select(.metric=="vulnerabilities") | .value' sonar-metrics.json)
                CODE_SMELLS=$(jq -r '.component.measures[] | select(.metric=="code_smells") | .value' sonar-metrics.json)
                COVERAGE=$(jq -r '.component.measures[] | select(.metric=="coverage") | .value' sonar-metrics.json)
                DUPLICATION=$(jq -r '.component.measures[] | select(.metric=="duplicated_lines_density") | .value' sonar-metrics.json)
                LOC=$(jq -r '.component.measures[] | select(.metric=="ncloc") | .value' sonar-metrics.json)

                echo "QUALITY_GATE=$QUALITY_GATE" >> sonar-env.txt
                echo "BUGS=$BUGS" >> sonar-env.txt
                echo "VULNERABILITIES=$VULNERABILITIES" >> sonar-env.txt
                echo "CODE_SMELLS=$CODE_SMELLS" >> sonar-env.txt
                echo "COVERAGE=$COVERAGE" >> sonar-env.txt
                echo "DUPLICATION=$DUPLICATION" >> sonar-env.txt
                echo "LOC=$LOC" >> sonar-env.txt
                '''
            }
        }
        stage('Send Security Report Email') {
            steps {
                emailext(
    to: 'durveshsshendokar@gmail.com',
    from: 'durveshsshendokar@gmail.com',
    subject: "DevSecOps Pipeline Report - ${currentBuild.currentResult}",
    mimeType: 'text/html',
    body: """
        <h2>DevSecOps Pipeline Report</h2>

        <p><b>Job:</b> ${env.JOB_NAME}</p>
        <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
        <p><b>Status:</b> ${currentBuild.currentResult}</p>
        <p><b>URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>

        <h3>Security Summary</h3>
            <ul>
            <li>SAST (SonarQube): <b>${env.SAST_STATUS}</b></li>
            <li>DAST (OWASP ZAP): <b>${env.DAST_STATUS}</b></li>
            </ul>


        <p>ZAP report attached.</p>
    """,
    attachmentsPattern: '''
                zap-report.html,
                sonar-quality-gate.json,
                sonar-metrics.json
            '''
)
                }
            }
        }

    post {
        always {
        archiveArtifacts artifacts: '''
            zap-report.html,
            sonar-quality-gate.json,
            sonar-metrics.json,
            sonar-env.txt,
            ''', allowEmptyArchive: true

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
