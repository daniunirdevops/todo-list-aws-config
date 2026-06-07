pipeline {
    agent any
    environment {
        AWS_REGION  = "us-east-1"
        REPO_URL    = 'https://github.com/daniunirdevops/todo-list-aws.git'
        REPO_URL_CONFIG = 'https://github.com/daniunirdevops/todo-list-aws-config.git'
        BRANCH      = "develop"
        BRANCH_CONFIG = "staging"
        ENVIRONMENT = "staging"
        // to create a temporal bucket for deploy
        TEMP_BUCKET = "sam-temp-${env.BUILD_ID}-${ENVIRONMENT}"
    }
    stages {
        stage('Get Code') {
            agent {label 'principal'}
            steps {
                sh 'whoami ; hostname'
                git branch: env.BRANCH,
                    url: env.REPO_URL                
                // obtain samconfig.toml from config 
                sh '''
                rm -f samconfig.toml
                curl -L -o samconfig.toml \
                https://raw.githubusercontent.com/daniunirdevops/todo-list-aws-config/${BRANCH_CONFIG}/samconfig.toml
                '''                
                
                stash name: 'code', includes: '**'
            }
        }
        stage('Checkout Config') {
            steps {
                sh 'whoami ; hostname'
                unstash 'code'     
                sh 'cat samconfig.toml'
            }
        }        
        stage('Static Test') {
            agent {label 'static'}
            steps {
                sh 'whoami ; hostname'
                unstash 'code'     
                sh '''
                    export PYTHONPATH=$WORKSPACE
                    bandit \
                            -r src \
                            -f custom \
                            -o bandit.out \
                            --msg-template "{abspath}:{line}: {severity}: {msg}"
                '''                       
                sh '''
                    export PYTHONPATH=$WORKSPACE
                    flake8 --format=pylint --exit-zero --output-file=flake8.out src
                '''
                recordIssues(
                    tools: [
                        pyLint(name: 'Bandit', pattern: 'bandit.out'),
                        flake8(name: 'Flake8', pattern: 'flake8.out')
                    ]
                )
            }
            post {
                always {
                    sh '''
                        rm -rf bandit.out || true
                        rm -rf flake8.out || true
                    '''
                }
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
                sh """
                    export BASE_URL="${env.BASE_URL}"
                    pytest -s test/integration/todoApiTest.py --junitxml=result-rest.xml
                """
                // show result
                junit 'result-rest.xml'            
            }
        }
        stage ('Promote') {
            agent { label 'principal' }
            steps {
                sh 'whoami ; hostname'
                unstash 'code'     
                sshagent(credentials: ['github-ssh']) {
                    sh '''
                        set -e
                        git config user.email "jenkins@ci"
                        git config user.name "Jenkins CI"
        
                        git fetch origin    
                        
                        git checkout -B master origin/master
                        git pull origin master
                        
                        git merge develop --no-ff|| true
                        
                        git ls-files -u
                        
                        for f in $(git ls-files -u | awk '{print $4}' | grep -E '^Jenkinsfile'); do
                        #for f in Jenkinsfile Jenkinsfile_agentes; do
                            if git ls-files -u | grep -q "$f"; then
                                git checkout --ours "$f"
                                git add "$f"
                            fi
                        done

                        if ! git diff --quiet; then
                            git commit -m "Merge develop into master (auto-resolved Jenkinsfile)"
                        fi
                        #force ssh
                        git remote set-url origin git@github.com:daniunirdevops/todo-list-aws.git                        
                        git push origin master
                    '''
            }
            }
        }
     }
    post {
        always {
            cleanWs() // clean all agents used
        }
    }    
}
