pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonarqube-token')
        DOJO_TOKEN = credentials('defectdojo-token')
        DOJO_URL = 'http://13.219.239.73:8080'
        ENGAGEMENT_ID = '1'
        TARGET_URL = 'http://100.31.192.76'
    }

    stages {

        stage('Clone Code') {
            steps {
                git branch: 'master',
                    credentialsId: 'github-credentials',
                    url: 'https://github.com/Amar9340/DVWA'
            }
        }

        // A04 + A01: Insecure Design & Access Control - SonarQube SAST
        stage('A04+A01: SonarQube SAST') {
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

        // A08: Software & Data Integrity - Docker Build
        stage('A08: Docker Build') {
            steps {
                sh 'docker build -t dvwa:latest .'
            }
        }

        // A06: Vulnerable & Outdated Components - Trivy
        stage('A06: Trivy Container Scan') {
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

        // A06: Vulnerable Components - OWASP Dependency Check
        stage('A06: Dependency Check') {
            steps {
                sh '''
                    dependency-check.sh \
                    --project DVWA \
                    --scan $(pwd) \
                    --format JSON \
                    --out $(pwd)/dc-report \
                    --disableYarnAudit \
                    --disableNodeAudit \
                    || true
                '''
            }
        }

        // A02: Cryptographic Failures - testssl.sh
        stage('A02: TLS/SSL Scan') {
            steps {
                sh '''
                    testssl.sh \
                    --jsonfile $(pwd)/testssl-report.json \
                    --severity HIGH \
                    --quiet \
                    $TARGET_URL || true
                '''
            }
        }

        // A03: Injection - SQLMap
        stage('A03: SQL Injection') {
            steps {
                sh '''
                    sqlmap -u "$TARGET_URL/vulnerabilities/sqli/?id=1&Submit=Submit" \
                    --cookie="PHPSESSID=abc123; security=low" \
                    --batch \
                    --level=3 \
                    --risk=2 \
                    --output-dir=$(pwd)/sqlmap-output \
                    --format=json || true
                '''
            }
        }

        // A01+A05+A07: Access Control, Misconfig, Auth - OWASP ZAP
        stage('A01+A05+A07: OWASP ZAP DAST') {
            steps {
                sh '''
                    docker run --rm \
                    --user root \
                    -v $(pwd):/zap/wrk/:rw \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-full-scan.py \
                    -t $TARGET_URL \
                    -r zap-report.html \
                    -J zap-report.json \
                    -x zap-report.xml \
                    -I || true
                '''
            }
        }

        // A05: Security Misconfiguration - Nikto
        stage('A05: Nikto Server Scan') {
            steps {
                sh '''
                    docker run --rm \
                    --user root \
                    -v $(pwd):/report \
                    secfigo/nikto:latest \
                    -h $TARGET_URL \
                    -o /report/nikto-report.xml \
                    -Format xml || true
                '''
            }
        }

        // A09: Logging Failures - Check security headers
        stage('A09: Security Headers Check') {
            steps {
                sh '''
                    curl -s -I $TARGET_URL > $(pwd)/headers-report.txt
                    echo "=== Checking Security Headers ===" >> $(pwd)/headers-report.txt
                    grep -i "x-frame-options" $(pwd)/headers-report.txt || echo "MISSING: X-Frame-Options" >> $(pwd)/headers-report.txt
                    grep -i "x-content-type" $(pwd)/headers-report.txt || echo "MISSING: X-Content-Type-Options" >> $(pwd)/headers-report.txt
                    grep -i "content-security-policy" $(pwd)/headers-report.txt || echo "MISSING: Content-Security-Policy" >> $(pwd)/headers-report.txt
                    grep -i "strict-transport-security" $(pwd)/headers-report.txt || echo "MISSING: Strict-Transport-Security" >> $(pwd)/headers-report.txt
                    grep -i "referrer-policy" $(pwd)/headers-report.txt || echo "MISSING: Referrer-Policy" >> $(pwd)/headers-report.txt
                    cat $(pwd)/headers-report.txt
                '''
            }
        }

        // A10: SSRF - ZAP already covers this via active rules
        // A08: Integrity check via Trivy already done above

        // Upload ALL findings to DefectDojo
        stage('Upload to DefectDojo') {
            steps {
                sh '''
                    # Trivy - A06
                    curl -X POST $DOJO_URL/api/v2/import-scan/ \
                    -H "Authorization: Token $DOJO_TOKEN" \
                    -F "scan_type=Trivy Scan" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "file=@trivy-report.json" \
                    -F "active=true" \
                    -F "verified=false"

                    # ZAP - A01+A05+A07
                    curl -X POST $DOJO_URL/api/v2/import-scan/ \
                    -H "Authorization: Token $DOJO_TOKEN" \
                    -F "scan_type=ZAP Scan" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "file=@zap-report.xml" \
                    -F "active=true" \
                    -F "verified=false"

                    # Nikto - A05
                    curl -X POST $DOJO_URL/api/v2/import-scan/ \
                    -H "Authorization: Token $DOJO_TOKEN" \
                    -F "scan_type=Nikto Scan" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "file=@nikto-report.xml" \
                    -F "active=true" \
                    -F "verified=false"

                    # Dependency Check - A06
                    curl -X POST $DOJO_URL/api/v2/import-scan/ \
                    -H "Authorization: Token $DOJO_TOKEN" \
                    -F "scan_type=Dependency Check Scan" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "file=@dc-report/dependency-check-report.json" \
                    -F "active=true" \
                    -F "verified=false" || true
                '''
            }
        }
    }

    post {
        always {
            echo 'OWASP Top 10 Pipeline finished!'
        }
    }
}
