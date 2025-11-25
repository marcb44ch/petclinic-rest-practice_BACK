pipeline {
    agent any

    tools {
        maven 'my-maven'
        jdk 'my-jdk'

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
                    // Publicar resultados de tests JUnit
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
                    // Archivar reporte de cobertura Jacoco
                    archiveArtifacts artifacts: 'target/site/jacoco/**/*', allowEmptyArchive: false
                    
                    // Publicar reporte de cobertura (opcional, para visualización en Jenkins)
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java',
                        exclusionPattern: 'src/test*'
                    )
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
                        # Opción 1: Usar sonar-scanner directamente (recomendado)
                        "${SONAR_SCANNER}/bin/sonar-scanner" -X
                        
                        # Opción 2: Usar Maven (alternativa)
                        # mvn sonar:sonar -Dsonar.token=${SONAR_AUTH_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    echo "⏳ Esperando que SonarQube sincronice el estado del análisis..."
                    
                    // Espera larga para sincronización
                    sleep 300 // 5 minutos
                    
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
            
            // Limpieza opcional
            sh '''
                echo "=== Espacio utilizado ==="
                du -h --max-depth=1 . || echo "No se pudo verificar espacio"
            '''
        }
        success {
            echo "✅ Pipeline Backend completado exitosamente!"
            emailext (
                subject: "✅ Pipeline Backend EXITOSO - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "El pipeline del backend se completó correctamente.\n\nVer build: ${env.BUILD_URL}",
                to: "tu-email@dominio.com"  // Ajusta con tu email
            )
        }
        failure {
            echo "❌ Pipeline Backend falló"
            emailext (
                subject: "❌ Pipeline Backend FALLIDO - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "El pipeline del backend ha fallado.\n\nVer build: ${env.BUILD_URL}",
                to: "tu-email@dominio.com"  // Ajusta con tu email
            )
        }
        unstable {
            echo "⚠️ Pipeline Backend inestable (Quality Gate no pasado)"
        }
    }
}