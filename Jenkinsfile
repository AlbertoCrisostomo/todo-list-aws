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
                echo 'Inicio de stage Promote!!!'
                script {
                    withCredentials([string(credentialsId: 'git-token-id', variable: 'GITHUB_TOKEN')]) {
                        def repoUrl = "https://AlbertoCrisostomo:${GITHUB_TOKEN}@github.com/AlbertoCrisostomo/todo-list-aws.git"
                        def developBranch = 'develop'
                        def masterBranch = 'master'
                        
                        sh '''
                        # Configurar Git
                        git config --global user.email "alberto.crisostomo@gmail.com"
                        git config --global user.name "AlbertoCrisostomo"

                        # Si el directorio ya existe, eliminarlo
                        if [ -d "todo-list-aws" ]; then
                            rm -rf todo-list-aws
                        fi
                        '''
                        
                        // Clonar el repositorio y realizar las operaciones de merge
                        sh "git clone https://${env.GITHUB_TOKEN}@github.com/AlbertoCrisostomo/todo-list-aws.git todo-list-aws"
                        sh '''
                        cd todo-list-aws

                        # Obtener las ramas remotas
                        git fetch origin

                        # Cambiar a la rama master
                        git checkout master

                        # Hacer merge con la rama develop, resolviendo conflictos automáticamente
                        git merge -X theirs origin/develop

                        # Revertir los cambios en el Jenkinsfile si hubo conflictos
                        git checkout --ours Jenkinsfile

                        # Añadir el Jenkinsfile al índice
                        git add Jenkinsfile

                        # Hacer commit de los cambios y pushear
                        git commit -m "Merged develop into master, resolving conflicts and keeping Jenkinsfile unchanged"
                        git push origin master
                        '''
                    } 
                }
            }
        }
        
        stage('Promote 2') {
            steps {
                echo 'Inicio de stage Promote 2!!!'
                script {
                    // Configuramos Git
                    sh 'git config user.name "AlbertoCrisostomo"'
                    sh 'git config user.email "alberto.crisostomo@gmail.com"'

                    //Eliminamon cualquier cambio en el directorio de trabajo
                    sh "git checkout -- ."
                    
                    // Agregamos el remoto y actualizamos
                    sh 'git remote set-url origin https://AlbertoCrisostomo:${GITHUB_TOKEN}@github.com/AlbertoCrisostomo/todo-list-aws.git'
                    sh 'git fetch origin master'

                     // Eliminamos la rama temporal si ya existe
                    sh 'git branch -D temp-merge || true'
                    
                    // Creamos una rama temporal para manejar el merge
                    sh 'git checkout -b temp-merge'

                    // Hacemos el merge excluyendo el Jenkinsfile
                    sh 'git merge origin/develop --no-commit --no-ff'
                    sh 'git reset HEAD Jenkinsfile'
                    sh 'git checkout -- Jenkinsfile'

                    // Verificamos si hay cambios para commit
                    def status = sh script: 'git status --porcelain', returnStdout: true
                    if (status) {
                        // Hay cambios para commit
                        sh 'git add -A'
                        sh 'git commit -m "Promoviendo develop a master excluyendo Jenkinsfile"'
                        sh 'git checkout master'
                        sh 'git merge temp-merge'
                        sh 'git push origin master'
                    } else {
                        echo 'No changes to commit.'
                    }
                    
                    // Limpiamos la rama temporal
                    sh 'git branch -d temp-merge'
                }
            }
        }
        
    }
}
