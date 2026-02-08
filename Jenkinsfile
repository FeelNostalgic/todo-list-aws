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
                bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
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
                        sam deploy --config-env staging --no-confirm-changeset
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
                        git checkout master
                        git pull origin master
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