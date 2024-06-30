pipeline {
    agent any

    environment {
        STACK_NAME = 'staging-todo-list-aws'
        AWS_REGION = 'us-east-1'
        S3_BUCKET = 'aws-sam-cli-managed-default-samclisourcebucket-tpbrtihbizum'
        S3_PREFIX = 'staging'
        STAGE = 'production'

        GITHUB_TOKEN = credentials('git-token-id')
    }

    stages {
        stage('Get Code') {
            steps {
                echo 'Inicio de stage Get Code!!!'
                git branch: 'master', url: 'https://github.com/AlbertoCrisostomo/todo-list-aws.git', credentialsId: 'git-token-id'
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
                            pytest --junitxml=result-rest.xml test/integration/todoApiTest.py -m smoke
                        '''
                        
                        // Muestra el resultado
                        junit 'result*.xml'
                    }
                }
            }
        }
        
    }
}
