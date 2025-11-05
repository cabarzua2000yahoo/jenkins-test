pipeline {
    agent any

    environment {
        // Nombre del proyecto para SonarQube y Dependency-Check
        PROJECT_NAME = "pipeline-sec"
        // URL del SonarQube (ajusta según tu instalación)
        SONARQUBE_URL = "http://localhost:9000"
        // Token de SonarQube (puedes guardarlo como credencial Jenkins si prefieres)
        SONARQUBE_TOKEN = "sqa_ee95b0074ba94ff9dda84a4a9df3bdb609f28ac0"
        // URL de la app Flask que analizará ZAP
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
              mkdir -p reports
        
              # Crear volumen persistente para cachear la base de datos NVD
              docker volume create dependency-check-data || true
        
              echo "🔍 Ejecutando OWASP Dependency-Check (con API Key y cache persistente)..."
        
              docker run --rm \
                  -v "$PWD":/src \
                  -v dependency-check-data:/usr/share/dependency-check/data \
                  owasp/dependency-check:12.1.2 \
                  --project pipeline-sec \
                  --scan /src \
                  --format HTML \
                  --out /src/reports \
                  --noupdate \
                  --enableExperimental

                )
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
                withSonarQubeEnv('sonarqube') {
                    sh """
                    sonar-scanner \
                        -Dsonar.projectKey=$PROJECT_NAME \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONARQUBE_URL \
                        -Dsonar.login=$SONARQUBE_TOKEN
                    """

                }
            }
        }

        stage('Security Test - OWASP ZAP') {
            steps {
                echo "🕵️ Ejecutando escaneo dinámico con OWASP ZAP..."
                // Si ZAP corre en Docker en la misma red que Jenkins
                sh """
                docker run --rm --network jenkins-net \
                    -v \$(pwd):/zap/wrk/:rw \
                    ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py \
                    -t $TARGET_URL \
                    -r zap-report.html
                """

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
            archiveArtifacts artifacts: 'reports/**/*.html, zap-report.html', fingerprint: true
        }
    }
}
