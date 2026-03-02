pipeline {
  agent any
  environment {
    AWS_REGION = "us-east-1"
    STACK_NAME = "todo-api-production"
  }

  stages {
    stage('====>Download configuration<====') {
      steps {
        echo "📥 Descargando samconfig.toml..."
        sh '''
          curl -o samconfig.toml \
          https://raw.githubusercontent.com/Magd13/todo-list-aws-config/production/samconfig.toml
        '''
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

            echo "🚀 SAM DEPLOY (NO INTERACTIVE)..."
            sam deploy
          '''
          // Capturar API KEY ID desde
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
            curl -s -H "x-api-key: ${API_KEY}" ${BASE_URL}/todosssss | grep "\\[" || exit 1
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