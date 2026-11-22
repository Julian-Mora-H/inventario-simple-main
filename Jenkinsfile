pipeline {
  agent any

  options {
    timestamps()
    timeout(time: 15, unit: 'MINUTES')
    disableConcurrentBuilds()
  }

  environment {
    // Nombre del proyecto Compose: aísla los contenedores del pipeline
    COMPOSE_PROJECT_NAME = 'inventario-ci'
    // Puerto publicado del frontend (compose.yaml lo lee con ${FRONTEND_PORT:-8080})
    FRONTEND_PORT        = '8082'
    // La prueba automatizada entra por el frontend; debe coincidir con FRONTEND_PORT
    INVENTARIO_URL       = 'http://127.0.0.1:8082'

    PATH                 = "C:\\Program Files\\Docker\\Docker\\resources\\bin;C:\\Program Files\\nodejs;${env.PATH}"
  }
  
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Diagnóstico') {
      steps {
        bat(returnStatus: true, script: 'where docker')
        bat(returnStatus: true, script: 'docker version')
        bat(returnStatus: true, script: 'where node')
      }
    }
    stage('Preparar .env') {
      steps {
        // Credencial tipo "Secret text" con ID: inventario-db-password
        withCredentials([string(credentialsId: 'inventario-db-password', variable: 'DB_PASS')]) {
          bat 'copy /Y .env.example .env'
          powershell '''
            $lineas = Get-Content .env | ForEach-Object {
              if ($_ -match '^INVENTARIO_DB_PASSWORD=') { "INVENTARIO_DB_PASSWORD=$env:DB_PASS" } else { $_ }
            }
            Set-Content -Path .env -Value $lineas -Encoding ascii
          '''
        }
      }
    }

    stage('Construir y levantar servicios') {
      steps {
        bat 'docker compose up -d --build --wait'
        bat 'docker compose ps'
      }
    }

    stage('Prueba automatizada') {
      steps {
        bat 'node --test tests/inventario.test.mjs'
      }
    }
  }

  post {
    always {
      // Diagnóstico y limpieza: no deben hacer fallar el build
      bat(returnStatus: true, script: 'docker compose logs --no-color')
      bat(returnStatus: true, script: 'docker compose down -v')
      bat(returnStatus: true, script: 'if exist .env del .env')
    }
    success {
      echo 'Pipeline completado: la prueba de integración pasó.'
    }
    failure {
      echo 'Pipeline fallido: revisa los logs de docker compose arriba.'
    }
  }
}
