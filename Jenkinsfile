pipeline {
    agent any

    environment {
        // Configuración de registro
        REGISTRY = 'docker.io'
        // (En el PDF, para GHCR sería: REGISTRY = 'ghcr.io')

        // Nombre de la imagen (formato: usuario/repo)
        IMAGE_NAME = 'tu-usuario-dockerhub/mi-app'

        // Variables de versión
        COMMIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        BUILD_TIMESTAMP = sh(script: 'date +%Y%m%d-%H%M%S', returnStdout: true).trim()

        // Tags de la imagen
        IMAGE_TAG_LATEST = "${IMAGE_NAME}:latest"
        IMAGE_TAG_COMMIT = "${IMAGE_NAME}:${COMMIT_SHA}"
        IMAGE_TAG_BUILD  = "${IMAGE_NAME}:build-${BUILD_TIMESTAMP}"
    }

    stages {
        // ============================================
        // STAGE 1: Preparación
        // ============================================
        stage('Prepare') {
            steps {
                echo 'Preparando entorno...'
                sh 'docker --version'
                sh 'node --version'
                sh 'npm --version'
            }
        }

        // ============================================
        // STAGE 2: Instalación de Dependencias
        // ============================================
        stage('Install Dependencies') {
            steps {
                echo 'Instalando dependencias...'
                sh 'npm ci'
            }
        }

        // ============================================
        // STAGE 3: Ejecutar Tests
        // ============================================
        stage('Test') {
            steps {
                echo 'Ejecutando tests...'
                sh 'npm test'
            }
            post {
                always {
                    // Publicar reportes de tests si existen
                    junit 'test-results/**/*.xml'
                }
            }
        }

        // ============================================
        // STAGE 4: Construcción de Imagen Docker
        // ============================================
        stage('Build Docker Image') {
            steps {
                echo 'Construyendo imagen Docker...'
                script {
                    docker.build("${IMAGE_TAG_COMMIT}")
                }
            }
        }

        // ============================================
        // STAGE 5: Publicación en Registro (Docker Hub)
        // Variante del PASO 4.2 del PDF
        // ============================================
        stage('Push to Docker Hub') {
            when {
                // Solo publicar en main o si es un tag
                branch 'main'
            }
            steps {
                echo 'Publicando imagen en Docker Hub...'
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-hub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        // Login a Docker Hub
                        sh '''
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        '''

                        // Taggear la imagen
                        sh """
                            docker tag ${IMAGE_TAG_COMMIT} ${IMAGE_TAG_LATEST}
                            docker tag ${IMAGE_TAG_COMMIT} ${IMAGE_TAG_BUILD}
                        """

                        // Publicar todas las tags
                        sh """
                            docker push ${IMAGE_TAG_COMMIT}
                            docker push ${IMAGE_TAG_LATEST}
                            docker push ${IMAGE_TAG_BUILD}
                        """
                    }
                }
            }
        }

        // ============================================
        // STAGE 6: Verificación
        // ============================================
        stage('Verify Published Image') {
            when {
                branch 'main'
            }
            steps {
                echo 'Verificando imagen publicada...'
                script {
                    sh """
                        echo "Imagen publicada: ${IMAGE_TAG_COMMIT}"
                        echo "Tags disponibles:"
                        echo " - ${IMAGE_TAG_COMMIT}"
                        echo " - ${IMAGE_TAG_LATEST}"
                        echo " - ${IMAGE_TAG_BUILD}"
                    """
                }
            }
        }
    }

    // ============================================
    // POST: Acciones finales
    // ============================================
    post {
        success {
            echo 'Pipeline completado exitosamente!'
            emailext (
                subject: "Pipeline Success: ${JOB_NAME} - ${BUILD_NUMBER}",
                body: "El pipeline se ha completado exitosamente.\n\nImagen publicada: ${IMAGE_TAG_COMMIT}",
                to: 'equipo@ejemplo.com'
            )
        }
        failure {
            echo 'Pipeline falló!'
            emailext (
                subject: "Pipeline Failed: ${JOB_NAME} - ${BUILD_NUMBER}",
                body: "El pipeline ha fallado. Revisa los logs para más detalles.",
                to: 'equipo@ejemplo.com'
            )
        }
        cleanup {
            echo 'Limpiando recursos...'
            script {
                // Limpiar imágenes locales para ahorrar espacio
                sh """
                    docker image prune -f
                """
            }
        }
    }
}
