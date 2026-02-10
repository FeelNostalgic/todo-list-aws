pipeline {
    agent any

    environment {
        GIT_CREDENTIALS_ID = 'github-token-user'
    }

    stages {
        stage('Get Code') {
            steps {
                cleanWs()
                checkout scm
                script{
                    sh 'curl -o samconfig.toml https://raw.githubusercontent.com/FeelNostalgic/todo-list-aws-config/production/samconfig.toml'
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh '''
                        sam build
                        set +e
                        sam deploy --no-confirm-changeset > deploy.log 2>&1
                        EXIT_CODE=$?
                        cat deploy.log
                        if [ $EXIT_CODE -ne 0 ]; then
                            if grep -q "No changes to deploy" deploy.log; then
                                echo "No hay cambios en la infraestructura, continuando..."
                                exit 0
                            else
                                echo "Error en el despliegue"
                                exit $EXIT_CODE
                            fi
                        fi
                        set -e
                    '''
                }
            }
        }

        stage('Rest Test') {
            steps {
                script {
                    def apiUrl = sh(script: "aws cloudformation describe-stacks --stack-name todo-list-aws-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text", returnStdout: true).trim()
                    echo "API desplegada en: ${apiUrl}"

                    withEnv(["BASE_URL=${apiUrl}"]) {
                        sh 'pytest --junitxml=result-rest.xml -m "readonly" test/integration/todoApiTest.py'
                    }
                }
            }
            post {
                always {
                    junit 'result-rest.xml'
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}