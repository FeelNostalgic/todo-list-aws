pipeline {
    agent any

    environment {
        GIT_CREDENTIALS_ID = 'github-token-user'
        REPO_URL = 'https://github.com/FeelNostalgic/todo-list-aws.git'
    }

    stages {
        stage('Get Code') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Static Test') {
            steps {
                sh '''
                flake8 src --exit-zero --format=pylint > flake8.out
                '''

                sh '''
                bandit -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}" || true
                '''

                recordIssues tools: [
                    flake8(pattern: 'flake8.out'), 
                    pyLint(name: 'Bandit', pattern: 'bandit.out')
                ]
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh '''
                        sam build
                        set +e
                        sam deploy --config-env staging --no-confirm-changeset > deploy.log 2>&1
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
                    def apiUrl = sh(script: "aws cloudformation describe-stacks --stack-name todo-list-aws-staging --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text", returnStdout: true).trim()
                    echo "API desplegada en: ${apiUrl}"

                    withEnv(["BASE_URL=${apiUrl}"]) {
                        sh 'pytest --junitxml=result-rest.xml test/integration/todoApiTest.py'
                    }
                }
            }
            post {
                always {
                    junit 'result-rest.xml'
                }
            }
        }

        stage('Promote') {
            steps {
                script {
                    def repoUrl = sh(returnStdout: true, script: 'git config remote.origin.url').trim()
                    withCredentials([usernamePassword(credentialsId: env.GIT_CREDENTIALS_ID, passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                        git fetch origin
                        git checkout master
                        git pull origin master
                        
                        git checkout develop
                        git pull origin develop
                        
                        git checkout master
                        git merge develop
                        git push https://\${GIT_USERNAME}:\${GIT_PASSWORD}@${repoUrl.replace("https://", "")} master
                        git checkout develop
                        """
                    }
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