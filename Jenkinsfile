pipeline {
    agent any

    environment {
        PROJECT_NAME = "pipeline-sec"
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_TOKEN = "sqa_3ad0adb13c4c79046b2c67a27b63995025307220"
        TARGET_URL = "http://localhost:5000"
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
                        mkdir -p $(pwd)/reports
                        chmod -R 777 $(pwd)/reports
            
                        # Copiar el código fuente del workspace al contenedor
                        docker cp . dependency-check:/src
            
                        # Ejecutar el análisis dentro del contenedor permanente
                        docker run --rm --user root \
                          -v dependency-check-data:/usr/share/dependency-check/data \
                          -v $(pwd):/src \
                          -v /home/seb/dependency-reports:/reports \
                          owasp/dependency-check:10.0.2 \
                          --project pipeline-sec \
                          --scan /src \
                          --format HTML \
                          --out /reports \
                          --enableExperimental || true

            
                        # Copiar el reporte generado desde el contenedor al workspace de Jenkins
                        docker cp dependency-check:/reports/dependency-check-report.html $(pwd)/reports/ || true
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
                        --network host \
                        -v $(pwd):/zap/wrk/:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                        -t http://127.0.0.1:5000 \
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
            sh '''
                echo "📦 Verificando ubicación de reportes..."
                ls -R $(pwd)
                mkdir -p reports
                # Copiar el reporte de ZAP si existe
                if [ -f zap-report.html ]; then
                    echo "📄 Copiando zap-report.html a reports/"
                    cp zap-report.html reports/
                fi
                # Confirmar que dependency-check-report.html exista
                if [ -f reports/dependency-check-report.html ]; then
                    echo "✅ dependency-check-report.html encontrado"
                else
                    echo "⚠️ dependency-check-report.html no encontrado"
                fi
            '''
            archiveArtifacts artifacts: '**/*.html', fingerprint: true
        }
    }
}
