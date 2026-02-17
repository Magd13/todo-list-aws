pipeline {
    agent any
    environment {
        AWS_REGION = "us-east-1"
        STACK_NAME = "todo-api-staging"
        STAGE = "staging"
    }
    stages {
        stage('====>Download configuration<====') {
            steps {
                echo "📥 Descargando configuración de entorno..."
                 sh '''
                    curl -o samconfig.toml \
                    https://raw.githubusercontent.com/Magd13/todo-list-aws-config/staging/samconfig.toml
                '''
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
                    // Ejecutar SAM deploy
                    sh '''
                        . venv/bin/activate
        
                        echo "📦 SAM BUILD..."
                        sam build
        
                        echo "✅ SAM VALIDATE..."
                        sam validate --region ${AWS_REGION}
        
                        rm -f samconfig.toml
                        
                        echo "${STACK_NAME}"
                        
                        echo "🚀 SAM DEPLOY (NO INTERACTIVE)..."
                        sam deploy \
                            --stack-name ${STACK_NAME} \
                            --region ${AWS_REGION} \
                            --capabilities CAPABILITY_IAM \
                            --no-confirm-changeset \
                            --no-fail-on-empty-changeset \
                            --s3-bucket todo-sam-artifacts-196164087862 \
                            --parameter-overrides Stage=staging
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
        stage('==========>PROMOTE (MERGE MASTER)<===========') {
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
}