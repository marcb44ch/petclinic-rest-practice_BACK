pipeline {
    agent any

    tools {
        maven 'my-maven'
        jdk 'my-jdk'
    }

    environment {
        SONAR_SERVER_NAME = 'sonar-server'
        SONAR_SCANNER = tool 'my-sonar-scanner'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Tests') {
            steps {
                sh '''
                    echo "=== Compilando y ejecutando tests unitarios ==="
                    mvn clean compile test -q
                '''
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                }
            }
        }

        stage('Integration Tests & Coverage') {
            steps {
                sh '''
                    echo "=== Ejecutando tests de integración y generando cobertura ==="
                    mvn verify -q
                    mvn jacoco:report -q
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/site/jacoco/**/*', allowEmptyArchive: false
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    sh '''
                        echo "=== Verificando herramientas ==="
                        echo "Java:"
                        java -version
                        echo "Maven:"
                        mvn -v
                        echo "SonarScanner:"
                        ls -la "${SONAR_SCANNER}" || echo "No se pudo acceder a SONAR_SCANNER"
                        
                        echo "=== Verificando reportes generados ==="
                        ls -la target/site/jacoco/ || echo "No existe directorio jacoco"
                        ls -la target/surefire-reports/ || echo "No existe directorio surefire-reports"
                    '''
                }
                withSonarQubeEnv(SONAR_SERVER_NAME) {
                    sh '''
                        echo "=== Ejecutando análisis SonarQube ==="
                        "${SONAR_SCANNER}/bin/sonar-scanner" -X
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    echo "⏳ Esperando que SonarQube sincronice el estado del análisis..."
                    sleep 300
                    echo "🎯 Iniciando verificación del Quality Gate..."
                    timeout(time: 15, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Package Application') {
            steps {
                sh '''
                    echo "=== Empaquetando aplicación ==="
                    mvn package -q -DskipTests
                '''
                archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: false
            }
        }
    }

    post {
        always {
            echo "Pipeline Backend - Resultado: ${currentBuild.result}"
        }
        success {
            echo "✅ Pipeline Backend completado exitosamente!"
        }
        failure {
            echo "❌ Pipeline Backend falló"
        }
        unstable {
            echo "⚠️ Pipeline Backend inestable (Quality Gate no pasado)"
        }
    }
}