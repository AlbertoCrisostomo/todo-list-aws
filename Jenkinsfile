pipeline {
    agent any

    environment {
        STACK_NAME = 'staging-todo-list-aws'
        AWS_REGION = 'us-east-1'
        S3_BUCKET = 'aws-sam-cli-managed-default-samclisourcebucket-tpbrtihbizum'
        S3_PREFIX = 'staging'
        STAGE = 'staging'
    }

    stages {
        stage('Get Code') {
            steps {
                echo 'Inicio de stage Get Code!!!'
                git branch: 'develop', url: 'https://git-token-id@github.com/AlbertoCrisostomo/todo-list-aws.git'
            }
        }
        
        stage('Static Test') {
            steps {
                echo 'Inicio de stage Static Test!!!'
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh'''
                        pwd
                        python3 -m flake8 --exit-zero --format=pylint src > flake8.out
                    '''
                    recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], qualityGates: [[threshold: 0, type: 'TOTAL', unstable: true], [threshold: 1, type: 'TOTAL', unstable: false]]
                }
            }
        }

        stage('Security Test') {
            steps {
                echo 'Inicio de stage Security Test!!!'
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh'''
                        bandit --exit-zero -r . -f custom -o bandit.out --severity-level medium --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                    '''
                    recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')], qualityGates: [[threshold: 20, type: 'TOTAL', unstable: true], [threshold: 40, type: 'TOTAL', unstable: false]]
                }
            }
        }
        
        stage('Build & Deploy'){
            steps {
                echo 'Inicio de stage Build & Deploy!!!'
                sh "sam build"

                sleep(time: 5, unit: 'SECONDS')

                echo 'Inicio del Deploy!!!'
                sh "sam deploy \
                        --template-file template.yaml \
                        --stack-name ${env.STACK_NAME} \
                        --region ${env.AWS_REGION} \
                        --capabilities CAPABILITY_IAM \
                        --parameter-overrides Stage=${env.STAGE} \
                        --no-fail-on-empty-changeset \
                        --s3-bucket ${env.S3_BUCKET} \
                        --s3-prefix ${env.S3_PREFIX} \
                        --no-confirm-changeset"
            }
        }

        stage('Rest Tests') {
            steps {
                script {
                    echo "Inicio de stage Rest Tests"
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        sh '''
                            #!/bin/bash
    
                            # Muestra la salida del Stage y Region
                            echo "get_base_url_api.sh --> Input 1 'stage' value: ${STAGE}"
                            echo "get_base_url_api.sh --> Input 2 'region' value: ${AWS_REGION}"
    
                            # Describe CloudFormation stacks y captura la salida
                            outputs=$(aws cloudformation describe-stacks --stack-name ${STAGE}-todo-list-aws --region ${AWS_REGION}  | jq '.Stacks[0].Outputs')
                        
                            # Extrae el valor de BaseUrlApi usando jq
                            BASE_URL_API=$(echo "$outputs" | jq -r '.[] | select(.OutputKey=="BaseUrlApi") | .OutputValue')
                        
                            # Muestra el valor de BaseUrlApi
                            echo $BASE_URL_API
    
                            # Setea en el entorno la URL
                            export BASE_URL=$BASE_URL_API
    
                            # Ejecuta las pruebas
                            pytest --junitxml=result-rest.xml test/integration/todoApiTest.py
                        '''
                        
                        // Muestra el resultado
                        junit 'result*.xml'
                    }
                }
            }
        }

        stage('Promote') {
            steps {
                echo 'Inicio de stage Promote!!!'
                script {
                    // Configuramos Git
                    sh 'git config user.name "AlbertoCrisostomo"'
                    sh 'git config user.email "alberto.crisostomo@gmail.com"'
                    
                    // Fetch all branches
                    sh 'git fetch origin'

                    // Cambiamos a la rama master
                    sh 'git checkout origin/master'

                    // Merge la rama develop sin hacer commit (para recuperar el archivo Jenkinsfile)
                    sh 'git merge --no-ff --no-commit origin/develop'

                    // Restauramos el archivo Jenkinsfile de la rama master
                    sh 'git checkout HEAD Jenkinsfile'
                    
                    // Commit del merge
                    sh 'git commit -m "Merge de la rama develop a la rama  master, manteniendo el Jenkinsfile de master"'
                    
                    // Push a la rama master
                    withCredentials([string(credentialsId: 'git-token-id', variable: 'GIT_TOKEN')]) {
                        sh 'git push https://${GIT_TOKEN}@https://github.com/AlbertoCrisostomo/todo-list-aws.git HEAD:master'
                    }
                }
            }
        }
        
    }
}
