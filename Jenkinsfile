pipeline {
    // ─────────────────────────────────────────────────────────────
    // El "agent any" le dice a Jenkins que puede ejecutar este
    // pipeline en cualquier nodo disponible (máquina/contenedor).
    // ─────────────────────────────────────────────────────────────
    agent any

    // ─────────────────────────────────────────────────────────────
    // OPCIONES DEL JOB (Parte obligatoria punto 1 + Opcional punto 2)
    //
    // - disableConcurrentBuilds: No deja que dos builds del mismo
    //   job se ejecuten a la vez (evita conflictos).
    // - timestamps: Añade la hora a cada línea del log para
    //   facilitar la depuración.
    // - timeout: Si el pipeline tarda más de 5 minutos, se cancela
    //   automáticamente (protección contra builds colgados).
    // - buildDiscarder: Solo guarda las últimas 10 ejecuciones
    //   en el historial (Opcional punto 2).
    // ─────────────────────────────────────────────────────────────
    options {
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 5, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    // ─────────────────────────────────────────────────────────────
    // VARIABLES DE ENTORNO (Parte obligatoria punto 2)
    //
    // Estas variables están disponibles en TODAS las etapas.
    // - FORCE_COLOR = '0': Desactiva colores ANSI (valor numérico)
    // - NO_COLOR = 'true': Desactiva colores (valor booleano)
    //
    // ¿Por qué? Jenkins no renderiza bien los caracteres de color
    // en los logs, así que los desactivamos para que sean legibles.
    // ─────────────────────────────────────────────────────────────
    environment {
        FORCE_COLOR = '0'
        NO_COLOR = 'true'
    }

    // ─────────────────────────────────────────────────────────────
    // ETAPAS DEL PIPELINE
    // Cada "stage" es un paso visible en la interfaz de Jenkins.
    // ─────────────────────────────────────────────────────────────
    stages {

        // ─────────────────────────────────────────────────────────
        // ETAPA 1: Auditoría de herramientas (Obligatoria punto 3)
        //
        // Comprobamos que Node.js está instalado y disponible.
        // Esto es útil para diagnosticar problemas si algo falla
        // más adelante.
        // ─────────────────────────────────────────────────────────
        stage('Audit tools') {
            steps {
                sh 'node --version'
            }
        }

        // ─────────────────────────────────────────────────────────
        // ETAPA 2: Instalación de dependencias (Obligatoria punto 4)
        //
        // "npm install" descarga todos los paquetes que el proyecto
        // necesita (definidos en package.json).
        // ─────────────────────────────────────────────────────────
        stage('Install dependencies') {
            steps {
                sh 'npm install'
            }
        }

        // ─────────────────────────────────────────────────────────
        // ETAPA 3: Generar ficheros (Obligatoria punto 5)
        //
        // Prisma es un ORM (herramienta para manejar la base de
        // datos). Este comando genera los ficheros que Prisma
        // necesita para funcionar (como los tipos TypeScript).
        // ─────────────────────────────────────────────────────────
        stage('Generate files') {
            steps {
                sh 'npm run prisma:generate'
            }
        }

        // ─────────────────────────────────────────────────────────
        // ETAPA 4: Linting en paralelo (Opcional punto 3)
        //
        // En vez de ejecutar "Format check" y "Code quality" una
        // detrás de otra, las ejecutamos AL MISMO TIEMPO con
        // "parallel". Esto ahorra tiempo.
        //
        // Si NO quieres la parte opcional, simplemente separa
        // estas dos etapas fuera del bloque "parallel".
        // ─────────────────────────────────────────────────────────
        stage('Linting') {
            parallel {

                // ─────────────────────────────────────────────────
                // Format check (Obligatoria punto 6)
                //
                // Verifica que el código sigue las reglas de
                // formato (indentación, espacios, etc.).
                // Si alguien no formateó bien su código, falla.
                // ─────────────────────────────────────────────────
                stage('Format check') {
                    steps {
                        sh 'npm run format:check'
                    }
                }

                // ─────────────────────────────────────────────────
                // Code quality permisivo (Obligatoria punto 7 +
                //                         Opcional punto 4)
                //
                // "npm run lint" analiza el código buscando errores
                // y malas prácticas.
                //
                // warnError(): Si el comando falla, en vez de
                //   parar todo el pipeline, solo muestra un aviso.
                //
                // currentBuild.result = 'UNSTABLE': Marca la build
                //   en amarillo (ni verde/éxito ni rojo/fallo).
                //
                // currentBuild.description: Texto que aparece
                //   debajo del número de build en el historial.
                //
                // Para probar que funciona: cambia "const app" por
                // "let app" en src/server.ts, haz commit+push y
                // lanza un nuevo build.
                // ─────────────────────────────────────────────────
                stage('Code quality') {
                    steps {
                        warnError('No se superaron los chequeos de calidad de código.') {
                            sh 'npm run lint'
                        }
                        script {
                            currentBuild.result = 'UNSTABLE'
                            currentBuild.description = 'UNSTABLE: Code quality'
                        }
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────
        // ETAPA 5: Comprobación de tipos (Obligatoria punto 8)
        //
        // TypeScript verifica que los tipos de datos son correctos.
        // Por ejemplo, si una función espera un número y le pasas
        // un texto, este paso lo detectará.
        // ─────────────────────────────────────────────────────────
        stage('Type check') {
            steps {
                sh 'npm run type-check'
            }
        }

        // ─────────────────────────────────────────────────────────
        // ETAPA 6: Tests con cobertura (Obligatoria punto 9 +
        //                               Opcional punto 1)
        //
        // Ejecuta los tests y genera un reporte de cobertura
        // (qué porcentaje del código está cubierto por tests).
        //
        // publishHTML: Publica el reporte HTML para que sea
        //   accesible desde la interfaz de Jenkins.
        //
        // Opciones de publishHTML:
        // - reportDir: Carpeta donde están los ficheros del reporte
        // - reportFiles: Fichero principal del reporte
        // - reportName: Nombre visible en Jenkins
        // - keepAll: Guarda reportes de TODAS las ejecuciones
        // - alwaysLinkToLastBuild: Enlaza al último build
        //   desde la página de la rama
        // - allowMissing: No falla si no encuentra los ficheros
        //   (útil si los tests no generan cobertura)
        // ─────────────────────────────────────────────────────────
        stage('Tests') {
            steps {
                sh 'npm run test:coverage'
                publishHTML([
                    reportDir: 'coverage',
                    reportFiles: 'index.html',
                    reportName: 'Coverage Report',
                    keepAll: true,
                    alwaysLinkToLastBuild: true,
                    allowMissing: true
                ])
            }
        }

        // ─────────────────────────────────────────────────────────
        // ETAPA 7: Build + Archivado (Obligatoria punto 10)
        //
        // "npm run build" compila el código TypeScript a JavaScript
        // y genera el fichero dist/server.mjs.
        //
        // archiveArtifacts: Guarda los ficheros generados en
        //   Jenkins para poder descargarlos después.
        // - artifacts: Patrón de ficheros a guardar (todo en dist/)
        // - fingerprint: Crea una "huella digital" para rastrear
        //   exactamente qué versión del fichero se generó
        // ─────────────────────────────────────────────────────────
        stage('Build') {
            steps {
                sh 'npm run build'
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }

        // ─────────────────────────────────────────────────────────
        // ETAPA 8: Tests E2E (Opcional punto 5)
        //
        // Los tests End-to-End (E2E) prueban la aplicación
        // completa, incluyendo la base de datos real (MongoDB).
        //
        // Docker Compose levanta:
        // - Un contenedor con MongoDB
        // - Un contenedor que ejecuta los tests
        //
        // La variable TEST_MODE=e2e le indica a vitest que ejecute
        // solo los tests de la carpeta src/e2e/.
        //
        // post > always: Pase lo que pase (éxito o error), limpia
        // todos los contenedores levantados para no dejar basura.
        // El "|| true" evita que falle si no hay nada que limpiar.
        // ─────────────────────────────────────────────────────────
        stage('E2E Tests') {
            environment {
                TEST_MODE = 'e2e'
            }
            steps {
                sh 'docker compose -f compose.e2e.yml run tests'
            }
            post {
                always {
                    sh 'docker compose -f compose.e2e.yml down -v --remove-orphans || true'
                }
            }
        }

        // ─────────────────────────────────────────────────────────
        // ETAPA 9: Publicar imagen Docker (Opcional punto 6)
        //
        // Esta etapa SOLO se ejecuta cuando:
        // 1. La rama es "main" (no queremos publicar desde ramas
        //    de desarrollo)
        // 2. La build es exitosa (result es null o SUCCESS)
        //
        // Variables de entorno de esta etapa:
        // - APP_VERSION: Versión del package.json (ej: "1.0.0")
        // - APP_BUILD_VERSION: Versión + número de build (ej: "1.0.0-13")
        // - DOCKER_HUB_REPO: Tu usuario/repositorio en DockerHub
        //
        // withDockerRegistry: Se autentica en DockerHub usando las
        //   credenciales que configuraste en Jenkins.
        //
        // docker.build: Construye la imagen Docker
        // image.push: Sube la imagen a DockerHub con un tag
        //
        // ⚠️ IMPORTANTE: Debes cambiar los valores de:
        //   - DOCKER_HUB_REPO: Pon tu usuario/repositorio real
        //   - credentialsId: Pon el ID de tus credenciales de Jenkins
        // ─────────────────────────────────────────────────────────
        stage('Publish') {
            when {
                branch 'main'
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            environment {
                APP_VERSION = sh(script: "npm pkg get version | tr -d '\"'", returnStdout: true).trim()
                APP_BUILD_VERSION = "${APP_VERSION}-${BUILD_NUMBER}"
                // ⚠️ CAMBIA ESTO: pon tu usuario de DockerHub
                DOCKER_HUB_REPO = '<tu-usuario>/cep-devops-backend'
            }
            steps {
                script {
                    // ⚠️ CAMBIA 'dockerhub-credentials': pon el ID
                    // de las credenciales que creaste en Jenkins
                    withDockerRegistry(url: '', credentialsId: 'dockerhub-credentials') {
                        def image = docker.build("${DOCKER_HUB_REPO}")
                        image.push('latest')
                        image.push("${APP_BUILD_VERSION}")
                    }
                }
            }
        }
    }

    // ─────────────────────────────────────────────────────────────
    // ETAPAS FINALES (Obligatoria punto 11)
    //
    // El bloque "post" se ejecuta DESPUÉS de todas las etapas,
    // sin importar si el pipeline tuvo éxito o falló.
    //
    // - success: Solo si todo fue bien → mensaje de éxito
    // - failure: Solo si algo falló → mensaje de error
    // - always: SIEMPRE se ejecuta → limpia el workspace
    //   (borra todos los ficheros descargados/generados para
    //   que el próximo build empiece limpio)
    // ─────────────────────────────────────────────────────────────
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Review logs.'
        }
        always {
            cleanWs()
        }
    }
}
