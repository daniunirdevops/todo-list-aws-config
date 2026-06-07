pipeline {
    agent any
    environment {
        REPO_URL    = 'https://github.com/daniunirdevops/todo-list-aws.git'
        REPO_URL_CONFIG = 'https://github.com/daniunirdevops/todo-list-aws-config.git'
        AWS_REGION  = "us-east-1"
        BRANCH      = "master"
        BRANCH_CONFIG = "production"        
        ENVIRONMENT = "production"
        // to create a temporal bucket for deploy
        TEMP_BUCKET = "sam-temp-${env.BUILD_ID}-${ENVIRONMENT}"
    }
    stages {
        stage('Get Code') {
            agent {label 'agent_1'}
            steps {
                sh 'whoami ; hostname'
                git branch: env.BRANCH,
                    url: env.REPO_URL                
                dir('config_tmp') {
                    git branch: env.BRANCH_CONFIG, url: env.REPO_URL_CONFIG
                }
                // Copy samconfig.toml from config 
                sh 'cp config_tmp/samconfig.toml ./samconfig.toml'
                stash name: 'code', includes: '**'
            }
        }
        stage('Deploy') {
            agent {label 'principal'}
            steps {
                sh 'whoami ; hostname'
                unstash 'code'     
                sh '''
                    aws s3 mb s3://${TEMP_BUCKET} --region ${AWS_REGION}
                    sam build
                    sam validate --region ${AWS_REGION}

                    sam deploy template.yaml \
                        --config-env ${ENVIRONMENT} \
                        --region ${AWS_REGION} \
                        --s3-bucket ${TEMP_BUCKET} \
                        --no-confirm-changeset \
                        --force-upload \
                        --no-fail-on-empty-changeset \
                        --on-failure DELETE \
                        --no-progressbar
                '''
                // obtain the URL
                script {
                    env.BASE_URL = sh(
                        script: """
                            aws cloudformation describe-stacks \
                                --stack-name todo-list-aws-${ENVIRONMENT} \
                                --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                                --region ${AWS_REGION} \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()   
                    echo "BASE_URL = ${env.BASE_URL}"
                }
            }
            post {
                always {
                    sh '''
                        aws s3 rb s3://${TEMP_BUCKET} --force || true
                    '''
                }
            }    
        }
        stage('Rest Test') {
            agent {label 'integration'}
            steps {
                sh 'whoami ; hostname'
                unstash 'code'     
                // obtain environment variable and assign to shell variable 
                // we run test readonly method
                sh """
                    export BASE_URL="${env.BASE_URL}"
                    pytest -s \
                        test/integration/todoApiTest.py::TestApi::test_api_listtodos \
                        test/integration/todoApiTest.py::TestApi::test_api_gettodo \
                        --junitxml=result-rest.xml
                """
                // show result
                junit 'result-rest.xml'            
            }
        }
     }
    
    post {
        always {
            cleanWs() // clean all agents used
        }
    }  
    
}
