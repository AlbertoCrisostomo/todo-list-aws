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
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    withCredentials([string(credentialsId: 'git-token-id', variable: 'PAT')]) {
                        sh """
                            echo 'STAGE --> Promote merge to master'
                            echo 'Host name:'; hostname
                            echo 'User:'; whoami
                            echo 'Workspace:'; pwd
                        """

                        script {
                            // Configuración de git
                            sh "git config --global user.email 'acrisostomop@gmail.com'"
                            sh "git config --global user.name 'AlbertoCrisostomo'"

                            //Eliminar cualquier cambio en el directorio de trabajo
                            sh "git checkout -- ."

                            //Hacer checkout a master y obtener la última versión desde el origen
                            sh "git checkout master"
                            sh "git pull https://\$PAT@github.com/AlbertoCrisostomo/todo-list-aws.git master"

                            //Hacer checkout a develop y obtener la última versión desde el origen
                            sh "git checkout develop"
                            sh "git pull https://\$PAT@github.com/AlbertoCrisostomo/todo-list-aws.git develop"
                                                    
                            //Checkout master
                            sh "git checkout master"

                            //Merge develop en master
                            def mergeStatus = sh(script: "git merge develop", returnStatus: true)

                            //En caso de conflicto en el Merge o si dió Error el Merge
                            if (mergeStatus){
                                //Mensaje de error para conflicto o error en la ejecución del merge
                                sh "echo 'Error: Merge conflict or other error occurred during git merge.'"
                                //Abort merge
                                sh "git merge --abort"

                                //Lanzar el merge nuevamente y mantener los archivos en master en caso de conflicto
                                sh "git merge develop -X ours --no-commit"
                                //Restaurar el archivo Jenkinsfile con la versión del master
                                sh "git checkout --ours Jenkinsfile"
                                sh "git add Jenkinsfile"
                                sh "git commit -m 'Merged develop into master, excluding Jenkinsfile'"
                            }
                            else {
                                sh "echo 'Merge completed successfully.'"
                            }
                            
                            //Push del resultado del merge result a master
                            sh "git push https://\$PAT@github.com/AlbertoCrisostomo/todo-list-aws.git master"
                                                    
                        }
                    }
                }
            }
        }
        
        stage('Promote 1') {
            steps {
                echo 'Inicio de stage Promote!!!'
                script {
                    withCredentials([string(credentialsId: 'git-token-id', variable: 'GITHUB_TOKEN')]) {
                        def developBranch = 'develop'
                        def masterBranch = 'master'
                        
                        // Cambiar al directorio del repositorio clonado
                        dir('todo-list-aws') {
                            sh '''
                            # Configurar Git
                            git config --global user.email "alberto.crisostomo@gmail.com"
                            git config --global user.name "AlbertoCrisostomo"

                            # Obtener las ramas remotas
                            git fetch origin
                            
                            # Verificar si existe el Jenkinsfile en origin/master
                            if git ls-tree -r --name-only origin/master | grep -q '^Jenkinsfile$'; then
                                # Restaurar el estado actual del Jenkinsfile en master
                                git checkout origin/master -- Jenkinsfile
                            else
                                echo "Jenkinsfile not found in origin/master"
                                exit 1
                            fi

                            # Cambiar a la rama develop
                            git checkout develop

                            # Limpiar archivos no rastreados si es posible
                            git clean -df || echo "Failed to clean untracked files"

                            # Hacer merge con la rama master
                            git merge origin/master
                            
                            # Resolver conflictos automáticamente, dando prioridad a los cambios de develop
                            git merge -X theirs origin/develop
                            
                            # Añadir el Jenkinsfile al índice para mantenerlo sin cambios
                            git checkout --ours Jenkinsfile
                            git add Jenkinsfile

                            # Hacer commit de los cambios y pushear a master
                            git commit -m "Merge develop into master, keeping Jenkinsfile unchanged"
                            git push origin develop:master
                            '''
                        }
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
                    sh 'git config user.email "acrisostomop@gmail.com"'

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
                    echo 'FINNNNNNNNNNNNNNN'
                }
            }
        }
        
    }
}
