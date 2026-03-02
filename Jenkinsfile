pipeline {
    agent any
    environment {
        AWS_REGION = "us-east-1"
        STACK_NAME = "todo-api-staging"
    }
    stages {
        stage('==========>CHECKOUT DEVELOP<===========') {
            steps {
                echo "📥 Descargando código de develop..."
                git branch: 'develop',
                url: 'https://github.com/Magd13/todo-list-aws.git'
                echo "✅ Código descargado correctamente"
            }
        }
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
        stage('==========>STATIC TEST<===========') {
           parallel {
                stage('Flake8') {
                    steps {
                        echo "EJECUTANDO FLAKE8"
                        sh '''
                            mkdir -p reports
                            flake8 src --output-file=reports/flake8-report.txt || true
                        '''
                    }
                }
                stage('Bandit') {
                    steps {
                        echo "EJECUTANDO BANDIT"
                        sh '''
                            mkdir -p reports
                            bandit -r src -f txt -o reports/bandit-reports.txt || true
                        '''
                    }
                }
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