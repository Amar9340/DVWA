pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonarqube-token')
        DOJO_TOKEN = credentials('defectdojo-token')
        DOJO_URL = 'http://13.219.239.73:8080'
        ENGAGEMENT_ID = '1'
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'master',
                    credentialsId: 'github-credentials',
                    url: 'https://github.com/Amar9340/DVWA'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        docker run --rm \
                        -e SONAR_HOST_URL=http://98.92.114.128:9000 \
                        -e SONAR_TOKEN=$SONAR_TOKEN \
                        -v $(pwd):/usr/src \
                        sonarsource/sonar-scanner-cli \
                        -Dsonar.projectKey=DVWA \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t dvwa:latest .'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    docker run --rm \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    -v $(pwd):/output \
                    aquasec/trivy:latest image \
                    --exit-code 0 \
                    --severity HIGH,CRITICAL \
                    --format json \
                    --output /output/trivy-report.json \
                    dvwa:latest
                '''
            }
        }

        stage('OWASP ZAP DAST') {
            steps {
                sh '''
                    docker run --rm \
                    --user root \
                    -v $(pwd):/zap/wrk/:rw \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-full-scan.py \
                    -t http://100.31.192.76 \
                    -r zap-report.html \
                    -J zap-report.json \
                    -x zap-report.xml \
                    -I || true
                '''
            }
        }

        stage('Upload to DefectDojo') {
            steps {
                sh '''
                    curl -X POST $DOJO_URL/api/v2/import-scan/ \
                    -H "Authorization: Token $DOJO_TOKEN" \
                    -F "scan_type=Trivy Scan" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "file=@trivy-report.json" \
                    -F "active=true" \
                    -F "verified=false"

                    curl -X POST $DOJO_URL/api/v2/import-scan/ \
                    -H "Authorization: Token $DOJO_TOKEN" \
                    -F "scan_type=ZAP Scan" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "file=@zap-report.xml" \
                    -F "active=true" \
                    -F "verified=false"
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished!'
        }
    }
}
