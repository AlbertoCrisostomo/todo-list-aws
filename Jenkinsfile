pipeline {
    agent any

    environment {
        STACK_NAME = 'staging-todo-list-aws'
        AWS_REGION = 'us-east-1'
        S3_BUCKET = 'aws-sam-cli-managed-default-samclisourcebucket-tpbrtihbizum'
        S3_PREFIX = 'staging'
        STAGE = 'staging'

        GITHUB_TOKEN = credentials('git-token-id')
    }

    stages {
        stage('Get Code') {
            steps {
                echo 'Inicio de stage Get Code!!!'
                git branch: 'develop', url: 'https://github.com/AlbertoCrisostomo/todo-list-aws.git', credentialsId: 'git-token-id'
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
                echo "Inicio de stage Promote"
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    withCredentials([string(credentialsId: 'git-token-id', variable: 'MIGIT')]) {
                        script {
                            // Configurando git
                            sh "git config --global user.email 'acrisostomop@gmail.com'"
                            sh "git config --global user.name 'AlbertoCrisostomo'"
        
                            // Eliminando cualquier cambio en el directorio de trabajo y asegurando de estar en una rama limpia
                            sh "git reset --hard"
                            sh "git clean -fd"
        
                            // Obteninendo la última versión desde el origen
                            sh "git fetch https://\$MIGIT@github.com/AlbertoCrisostomo/todo-list-aws.git"
        
                            // Haciendo checkout a master y merge de develop
                            sh "git checkout master"
                            sh "git pull origin master"
                            sh "git checkout develop"
                            sh "git pull origin develop"
                            sh "git checkout master"
                            
                            // Intentando el merge
                            def mergeStatus = sh(script: "git merge develop", returnStatus: true)
                            
                            // Resolviendo conflictos y asegurando que Jenkinsfile no se actualice
                            if (mergeStatus != 0) {
                                sh "echo 'Merge con conflictos detectados. Resolviendo conflictos.'"
                                // Restaurar el archivo Jenkinsfile con la versión del master
                                sh """
                                    git merge --abort
                                    git merge develop -X ours --no-commit
                                    git checkout --ours Jenkinsfile
                                    git add Jenkinsfile
                                    git commit -m 'Merge de la rama develop a la rama master con Jenkinsfile de master'
                                """
                            } else {
                                sh "echo 'Merge completado sin conflictos.'"
                                // Asegurando que el archivo Jenkinsfile no se actualice
                                sh """
                                    git checkout --ours Jenkinsfile
                                    git add Jenkinsfile
                                    git commit -m 'Jenkinsfile es de master'
                                """
                            }
        
                            // Push del resultado del merge a master
                            sh "git push https://\$MIGIT@github.com/AlbertoCrisostomo/todo-list-aws.git master"
                        }
                    }
                }
            }
        }

        stage('Promote 3') {
            steps {
                echo "Inicio de stage Promote"
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    withCredentials([string(credentialsId: 'git-token-id', variable: 'PAT')]) {
                        script {
                            // Configuración de git
                            sh "git config --global user.email 'acrisostomop@gmail.com'"
                            sh "git config --global user.name 'AlbertoCrisostomo'"
        
                            // Eliminar cualquier cambio en el directorio de trabajo y asegurar estar en una rama limpia
                            sh "git reset --hard"
                            sh "git clean -fd"
        
                            // Obtener la última versión desde el origen
                            sh "git fetch https://\$PAT@github.com/AlbertoCrisostomo/todo-list-aws.git"
        
                            // Hacer checkout a master y merge de develop
                            sh "git checkout master"
                            sh "git pull origin master"
                            sh "git checkout develop"
                            sh "git pull origin develop"
                            sh "git checkout master"
                            sh "git merge develop"
        
                            // Resolver conflictos y asegurar que Jenkinsfile no se actualice
                            def mergeStatus = sh(script: "git status --porcelain", returnStdout: true).trim()
        
                            if (mergeStatus) {
                                sh "echo 'Merge conflict detected. Resolving conflicts.'"
                                // Restaurar el archivo Jenkinsfile con la versión del master
                                sh """
                                    git checkout --ours Jenkinsfile
                                    git add Jenkinsfile
                                    git commit -m 'Merge develop into master with Jenkinsfile from master'
                                """
                            } else {
                                sh "echo 'Merge completed successfully without conflicts.'"
                                // Asegurar que el archivo Jenkinsfile no se actualice
                                sh """
                                    git checkout --ours Jenkinsfile
                                    git add Jenkinsfile
                                    git commit -m 'Ensure Jenkinsfile from master'
                                """
                            }
        
                            // Push del resultado del merge a master
                            sh "git push https://\$PAT@github.com/AlbertoCrisostomo/todo-list-aws.git master"
                        }
                    }
                }
            }
        }
        
        
    }
}
