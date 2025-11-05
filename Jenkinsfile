pipeline {
    agent any

    environment {
        PROJECT_NAME = "pipeline-sec"
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_TOKEN = "sqa_8fcc68789aa69ec6176ac80bd11b81d2f43d169e"
        TARGET_URL = "http://172.23.41.49:5000"
    }

    stages {
        stage('Build') {
            steps {
                echo "🏗️ Compilando y preparando el entorno..."
                sh 'echo "Simulando build (no se requiere compilación en Python)"'
            }
        }

        stage('Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    sh '''
                        mkdir -p reports
                        chmod -R 777 reports
                        docker volume create dependency-check-data || true
                        echo "🔍 Ejecutando OWASP Dependency-Check..."
                        docker run --rm \
                            -v "$PWD":/src \
                            -v dependency-check-data:/usr/share/dependency-check/data \
                            owasp/dependency-check:10.0.2 \
                            --project pipeline-sec \
                            --scan /src \
                            --format HTML \
                            --out /src/reports \
                            --noupdate \
                            --enableExperimental || true
                    '''
                }
            }
            post {
                success {
                    echo "✅ Reporte generado en reports/dependency-check-report.html"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "🧠 Analizando código con SonarQube..."
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('sonarqube') {
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
                echo "🕵️ Ejecutando escaneo dinámico con OWASP ZAP..."
                sh '''
                    chmod -R 777 $(pwd)
                    docker run --rm --user root \
                        --network jenkins-net \
                        -v $(pwd):/zap/wrk/:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://172.23.41.49:5000 \
                        -r zap-report.html || true
                '''
            }
            post {
                success {
                    echo "✅ Reporte de ZAP generado: zap-report.html"
                }
            }
        }

        stage('Deploy (simulado)') {
            steps {
                echo "🚀 Desplegando aplicación en entorno de prueba..."
                sh 'echo "Despliegue simulado completado."'
            }
        }
    }

    post {
        always {
            echo "🧾 Guardando reportes de análisis..."
    
            // Verificar si los reportes existen antes de archivarlos
            sh '''
                echo "📁 Archivos en reports/:"
                ls -la reports || true
                echo "📁 Archivos en workspace:"
                ls -la || true
            '''
    
            // Archivar lo que exista
            archiveArtifacts artifacts: '**/dependency-check-report.*', fingerprint: true, allowEmptyArchive: true
            archiveArtifacts artifacts: 'zap-report.html', fingerprint: true, allowEmptyArchive: true
        }
    }
}
