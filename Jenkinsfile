pipeline {
    agent any

    environment {
        PROJECT_NAME = "pipeline-sec"
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_TOKEN = "squ_24a1b6f8e42dfa511669fa9f68c7035e861e3546"
        TARGET_URL = "http://172.23.131.149:5000"
    }

    stages {
        stage('Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    sh '''
                        mkdir -p reports
                        chmod -R 777 reports
                        docker volume create dependency-check-data || true
                        echo "Ejecutando OWASP Dependency-Check..."
                        docker run --rm --user root \
                            -v "$PWD":/src \
                            -v dependency-check-data:/usr/share/dependency-check/data \
                            owasp/dependency-check:8.4.0 \
                            --project pipeline-sec \
                            --scan /src \
                            --format "ALL" \
                            --out /src/reports \
                            --enableExperimental || true
                    '''
                }
            }
            post {
                always {
                    echo "Dependency-Check finalizado"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "Analizando codigo con SonarQube..."
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQubeScanner') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=$PROJECT_NAME \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=$SONARQUBE_URL \
                                -Dsonar.login=$SONARQUBE_TOKEN
                        """
                    }
                }
            }
        }

        stage('Security Test - OWASP ZAP') {
            steps {
                echo "Ejecutando escaneo con OWASP ZAP..."
                script {
                    // Inicia ZAP dentro de un contenedor (igual que antes)
                    sh '''
                        docker run -d --rm \
                            --name zap-temp \
                            --network jenkins-net \
                            -v $(pwd):/zap/wrk/:rw \
                            -p 8090:8090 \
                            ghcr.io/zaproxy/zaproxy:stable \
                            zap.sh -daemon -port 8090 -host 0.0.0.0 -config api.disablekey=true
                    '''

                    // Espera unos segundos a que ZAP inicie
                    sleep 15

                    // Usa los pasos del plugin
                    zapStart zapHome: '/zap', port: 8090
                    zapScan target: "${TARGET_URL}"
                    zapReport fileName: 'zap-report.html', reportDir: '.', reportType: 'HTML'
                    zapReport fileName: 'zap-report.xml', reportDir: '.', reportType: 'XML'
                    zapStop()

                    // Detiene el contenedor
                    sh 'docker stop zap-temp || true'
                }
            }
            post {
                always {
                    echo "Reporte de ZAP generado (html/json)"
                }
            }
        }
    }

    post {
        always {
            sh '''
                echo "Archivos generados:"
                find $(pwd) -maxdepth 2 -type f -name "*.html" -o -name "*.json" || true
            '''
            archiveArtifacts artifacts: 'reports/**/*.html, reports/**/*.json', fingerprint: true, allowEmptyArchive: true
            archiveArtifacts artifacts: 'zap-report.html, zap-warnings.html, zap-report.json', fingerprint: true, allowEmptyArchive: true
        }
    }
}
