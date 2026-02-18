pipeline {
    agent any
    environment {
        AWS_REGION = "us-east-1"
    }
    stages {
      stage('====>Download configuration<====') {
        steps {
          echo "📥 Descargando configuración de entorno..."
          if (env.BRANCH_NAME == "develop") {
            sh '''
              curl -o samconfig.toml \
              https://raw.githubusercontent.com/Magd13/todo-list-aws-config/staging/samconfig.toml
            '''
          }
          if (env.BRANCH_NAME == "master") {
            sh '''
              curl -o samconfig.toml \
              https://raw.githubusercontent.com/Magd13/todo-list-aws-config/production/samconfig.toml
            '''
          }
          echo "✅ Config descargada:"
          sh "cat samconfig.toml"
        }
      }
      stage('=====> CREATE VENV & INSTALL TOOLS <=====') {
        steps {
          echo "🐍 Creando entorno virtual e instalando dependencias..."
          sh '''
            python3 -m venv venv
            . venv/bin/activate
            pip install --upgrade pip
            pip install flake8 bandit aws-sam-cli
          '''
        }
      }
      stage('==========>STATIC TEST<===========') {
        when {
          branch 'develop'
        }
        steps {
          echo "EJECUTANDO ANALISIS ESTATICO EN SRC/"
          sh '''
            mkdir -p reports
            . venv/bin/activate
            
            echo "RUNNUNG FLAKE8"
            flake8 src --output-file=reports/flake8-report.txt || true
            
            echo "RUNNING BANDIT" > reports/bandit.out
            bandit -r src -f txt -o reports/bandit-reports.txt || true
          '''
        }
        post {
          always {
            echo "📌 Publicando reportes estáticos..."
            archiveArtifacts artifacts: "reports/*.txt", fingerprint: true
          }
        }
      }
      stage('==========>DEPLOY (SAM)<===========') {
        steps {
          echo "🚀 Construyendo y desplegando con AWS SAM en STAGING..."
          script {
            if (env.BRANCH_NAME == "develop") {
              env.STACK_NAME = "todo-api-staging"
              echo "🚀 Deploy STAGING..."
            }
            if (env.BRANCH_NAME == "master") {
              env.STACK_NAME = "todo-api-production"
              echo "🚀 Deploy PRODUCTION..."
            }
            sh '''
              . venv/bin/activate

              echo "📦 SAM BUILD..."
              sam build

              echo "✅ SAM VALIDATE..."
              sam validate --region ${AWS_REGION}
              
              echo "🚀 SAM DEPLOY (NO INTERACTIVE)..."
              sam deploy
            '''
            // Capturar API Key en variable de entorno
            def apiKeyId = sh(
              script: '''
                aws cloudformation describe-stack-resources \
                  --stack-name ${STACK_NAME} \
                  --query "StackResources[?ResourceType=='AWS::ApiGateway::ApiKey'].PhysicalResourceId" \
                  --output text
              ''',
              returnStdout: true
            ).trim()
            
            echo "📌 API KEY ID detectada: ${apiKeyId}"
            
            def apiKeyValue = sh(
              script: """
                aws apigateway get-api-key \
                  --api-key ${apiKeyId} \
                  --include-value \
                  --query 'value' \
                  --output text \
                  --region ${AWS_REGION}
              """,
              returnStdout: true
            ).trim()

            // Capturar BASE_URL en variable de entorno
            def baseUrlOutput = sh(
              script: '''
                aws cloudformation describe-stacks \
                  --stack-name ${STACK_NAME} \
                  --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                  --output text
              ''',
              returnStdout: true
            ).trim()

            env.BASE_URL = baseUrlOutput
            echo "📌 BASE_URL configurada: ${env.BASE_URL}"
            env.API_KEY = apiKeyValue
            echo "📌 API_KEY configurada: ${env.API_KEY}"
          }
        }
      }

      stage('==========>REST TEST (PYTEST)<===========') {
        when {
          branch 'develop'
        }
        environment {
            BASE_URL = "${env.BASE_URL}"
            API_KEY = "${env.API_KEY}"
        }
        steps {
          echo "🌐 Ejecutando pruebas de integración..."
          sh '''
            . venv/bin/activate

            pip install pytest requests

            pytest -v test/integration/todoApiTest.py
          '''
        }
      }
      stage('==========> REST TEST (READ ONLY) <===========') {
        when {
          branch 'master'
        }
        environment {
          BASE_URL = "${env.BASE_URL}"
          API_KEY  = "${env.API_KEY}"
        }
        steps {
          echo "🌐 Ejecutando pruebas REST SOLO LECTURA (GET)..."

          sh '''
            . venv/bin/activate
            echo "✅ TEST 1: GET /todos"
            curl -s -H "x-api-key: ${API_KEY}" ${BASE_URL}/todos | grep "\\[" || exit 1
            echo "✅ TEST 2: GET /todos/{id}"
            curl -s -H "x-api-key: ${API_KEY}" ${BASE_URL}/todos/1 || exit 1
            echo "🎉 REST TEST READ-ONLY PASSED!"
          '''
        }
      }
      stage('==========>PROMOTE (MERGE MASTER)<===========') {
        when {
          branch 'develop'
        }
        steps {
          echo "🚀 Promoviendo versión a Release..."
          withCredentials([usernamePassword(credentialsId: 'case1.4', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) {
            sh '''
              git fetch origin
              git checkout master
              git pull origin master
              git merge origin/develop
              git push https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/Magd13/todo-list-aws.git master
            '''
          }
        }
      }
    }
  post {
    success {
      echo "✅ Pipeline ejecutado con éxito en rama: ${BRANCH_NAME}"
    }
    failure {
      echo "❌ Pipeline falló en rama: ${BRANCH_NAME}"
    }
  }
}