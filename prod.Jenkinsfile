pipeline {
  agent any
  environment {
    AWS_REGION = "us-east-1"
    STACK_NAME = "todo-api-production"
    STAGE = "production"
  }

  stages {
    stage('=====> CREATE VENV & INSTALL TOOLS <=====') {
      steps {
        echo "🐍 Creando entorno virtual e instalando dependencias..."
        sh '''
          python3 -m venv venv
          . venv/bin/activate
          pip install --upgrade pip
          pip install aws-sam-cli pytest requests
        '''
      }
    }
    stage('==========> DEPLOY (SAM PRODUCTION) <===========') {
      steps {
        echo "🚀 Desplegando con AWS SAM en PRODUCCIÓN..."
        script {
          sh '''
            . venv/bin/activate

            echo "📦 SAM BUILD..."
            sam build

            echo "✅ SAM VALIDATE..."
            sam validate --region ${AWS_REGION}
            rm -f samconfig.toml

            echo "🚀 SAM DEPLOY (NO INTERACTIVE)..."
            sam deploy \
              --stack-name ${STACK_NAME} \
              --region ${AWS_REGION} \
              --capabilities CAPABILITY_IAM \
              --no-confirm-changeset \
              --no-fail-on-empty-changeset \
              --s3-bucket todo-sam-artifacts-196164087862 \
              --parameter-overrides Stage=production
          '''
          // Capturar API KEY ID desde
          def apiKeyOutput = sh(
            script: '''
              aws cloudformation describe-stacks \
                  --stack-name ${STACK_NAME} \
                  --query "Stacks[0].Outputs[?OutputKey=='TodoApiKey'].OutputValue" \
                  --output text
            ''',
            returnStdout: true
          ).trim()
          // Obtener el valor real de la API KEY
          def apiKeyValue = sh(
            script: """
              aws apigateway get-api-key \
                --api-key ${apiKeyOutput} \
                --include-value \
                --query 'value' \
                --output text \
                --region ${AWS_REGION}
            """,
            returnStdout: true
          ).trim()
          // Capturar BASE URL
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
          env.API_KEY = apiKeyValue
          echo "📌 BASE_URL producción: ${env.BASE_URL}"
          echo "📌 API_KEY producción configurada correctamente"
        }
      }
    }
    stage('==========> REST TEST (READ ONLY) <===========') {
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
  }
  post {
    success {
        echo "✅ CD Pipeline completado con éxito. Producción desplegada 🚀"
    }

    failure {
        echo "❌ CD Pipeline falló. Producción NO fue promovida."
    }
  }
}