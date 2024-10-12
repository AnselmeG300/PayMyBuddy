pipeline {
    agent {
        dockerfile {
            filename 'agent/Dockerfile'
            args '-v /root/.m2:/root/.m2 -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

                
    parameters {
        string(name: 'IMAGE_NAME_PROD', defaultValue: 'paymybuddy')
        string(name: 'IMAGE_TAG_PROD', defaultValue: 'latest')
    }


    environment {
        DOCKERHUB_AUTH = credentials('DockerHubCredentials')
        MYSQL_AUTH= credentials('MYSQL_AUTH')
        HOSTNAME_DEPLOY_PROD = "54.197.76.44"
        HOSTNAME_DEPLOY_STAGING = "18.208.156.10"
        IMAGE_NAME= 'paymybuddy'
        IMAGE_TAG= '1.0'
        AWS_REGION = 'us-east-1' // Remplacez par votre région AWS préférée
        INSTANCE_TYPE = 't2.medium'
        AMI_ID = 'ami-0866a3c8686eaeeba'
        KEY_NAME = 'deploy'
        SECURITY_GROUP = 'sg-07fadf1d01a417de7' // Remplacez par votre groupe de sécurité
        STORAGE = 100
        SUBNET_ID = 'subnet-059ab491ba44695d7'
        VPC_ID = 'vpc-012d419c1a2bb6ee5'
    }

    stages {

        stage('Test') {
            when {
                expression { GIT_BRANCH != 'origin/prod' }
            }
            steps {
                sh 'mvn clean test'
            }

            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarCloud analysis') {
            when {
                expression { GIT_BRANCH != 'origin/prod' }
            }
            steps {
                withSonarQubeEnv('SonarCloudServer') {
                    sh 'mvn sonar:sonar -s .m2/settings.xml'
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { GIT_BRANCH != 'origin/prod' }
            }
            steps {
                timeout(time: 60, unit: 'SECONDS') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Package') {
            when {
                expression { GIT_BRANCH != 'origin/prod' }
            }
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build and push IMAGE to docker registry') {
            when {
                expression { GIT_BRANCH != 'origin/prod' }
            }
            steps {
                sh """
                    docker build -t ${DOCKERHUB_AUTH_USR}/${IMAGE_NAME}:${IMAGE_TAG} .
                    echo ${DOCKERHUB_AUTH_PSW} | docker login -u ${DOCKERHUB_AUTH_USR} --password-stdin
                    docker push ${DOCKERHUB_AUTH_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('IAC Review') {
            steps {
                withCredentials([aws(credentialsId: 'credentialsId', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    script {
                        // Installer AWS CLI
                        sh '''
                            apk add --no-cache python3 py3-pip
                            pip3 install awscli
                        '''

                        // Configurer AWS CLI avec les credentials
                        sh "aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}"
                        sh "aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}"
                        sh "aws configure set region ${AWS_REGION}"

                        // Construire le tag à partir du nom de la branche
                        def branchName = GIT_BRANCH // Nom de la branche
                        def tag = "review-${branchName}"

                        // Vérifier si une instance avec le tag existe déjà
                        def instancesCheckCommand = """
                            aws ec2 describe-instances \
                            --filters "Name=tag:Name,Values=${tag}" \
                            --query "Reservations[*].Instances[*].InstanceId" \
                            --output text
                        """

                        def existingInstance = sh(script: instancesCheckCommand, returnStdout: true).trim()

                        if (existingInstance) {
                            echo "Une instance avec le tag '${tag}' existe déjà : ${existingInstance}"
                        } else {
                            // User Data pour l'installation de Docker
                            def userData = """#!/bin/bash
                            curl -fsSL https://get.docker.com -o install-docker.sh
                            sh install-docker.sh --dry-run
                            sudo sh install-docker.sh
                            sudo usermod -aG docker ubuntu
                            """

                            // Commande pour créer l'instance EC2
                            def createInstanceCommand = """
                                aws ec2 run-instances \
                                --image-id ${AMI_ID} \
                                --count 1 \
                                --instance-type ${INSTANCE_TYPE} \
                                --key-name ${KEY_NAME} \
                                --security-group-ids ${SECURITY_GROUP} \
                                --block-device-mappings DeviceName=/dev/sda1,Ebs={VolumeSize=${STORAGE}} \
                                --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=${tag}}]' \
                                --user-data '${userData}'
                            """

                            // Exécuter la commande
                            sh createInstanceCommand
                            echo "Instance EC2 créée avec le tag '${tag}'."
                        }
                    }
                }
            }
        }

        stage('Deploy Review') {
    steps {
        withCredentials([aws(credentialsId: 'credentialsId', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
            script {
                // Configurer AWS CLI avec les credentials
                sh "aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}"
                sh "aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}"
                sh "aws configure set region ${AWS_REGION}"

                // Construire le tag à partir du nom de la branche
                def branchName = GIT_BRANCH // Nom de la branche
                def tag = "review-${branchName}"

                // Récupérer l'IP publique de l'instance en fonction du tag
                def publicIpCommand = """
                    aws ec2 describe-instances \
                    --filters "Name=tag:Name,Values=${tag}" \
                    --query "Reservations[*].Instances[*].PublicIpAddress" \
                    --output text
                """

                def publicIp = sh(script: publicIpCommand, returnStdout: true).trim()

                if (!publicIp) {
                    error "Aucune instance trouvée avec le tag '${tag}'."
                }

                // Stocker l'IP dans une variable
                env.HOSTNAME_DEPLOY_REVIEW = publicIp
                echo "IP publique de l'instance : ${HOSTNAME_DEPLOY_REVIEW}"

                // Préparer les commandes de déploiement
                def deployCommands = """
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa ${HOSTNAME_DEPLOY_REVIEW} >> ~/.ssh/known_hosts
                    scp -r deploy ubuntu@${HOSTNAME_DEPLOY_REVIEW}:/home/ubuntu/
                    command1="cd deploy && echo ${DOCKERHUB_AUTH_PSW} | docker login -u ${DOCKERHUB_AUTH_USR} --password-stdin"
                    command2="echo 'IMAGE_VERSION=${DOCKERHUB_AUTH_USR}/${IMAGE_NAME}:${IMAGE_TAG}' > .env && echo ${MYSQL_AUTH_PSW} > secrets/db_password.txt && echo ${MYSQL_AUTH_USR} > secrets/db_user.txt"
                    command3="echo 'SPRING_DATASOURCE_URL=jdbc:mysql://paymybuddydb:3306/db_paymybuddy' > env/paymybuddy.env && echo 'SPRING_DATASOURCE_PASSWORD=${MYSQL_AUTH_PSW}' >> env/paymybuddy.env && echo 'SPRING_DATASOURCE_USERNAME=${MYSQL_AUTH_USR}' >> env/paymybuddy.env"
                    command4="docker compose down && docker pull ${DOCKERHUB_AUTH_USR}/${IMAGE_NAME}:${IMAGE_TAG}"
                    command5="docker compose up -d"
                    ssh -t ubuntu@${HOSTNAME_DEPLOY_REVIEW} \
                        -o SendEnv=IMAGE_NAME \
                        -o SendEnv=IMAGE_TAG \
                        -o SendEnv=DOCKERHUB_AUTH_USR \
                        -o SendEnv=DOCKERHUB_AUTH_PSW \
                        -o SendEnv=MYSQL_AUTH_USR \
                        -o SendEnv=MYSQL_AUTH_PSW \
                        -C "$command1 && $command2 && $command3 && $command4 && $command5"
                """

                // Exécuter les commandes de déploiement
                sh deployCommands
            }
        }
    }
}


        stage('Destroy Review') {
    steps {
        script {
            // Demander une confirmation manuelle avant de continuer
            def userInput = input(
                id: 'userInput', 
                message: 'Êtes-vous sûr de vouloir supprimer l\'instance EC2 ?',
                parameters: [
                    booleanParam(name: 'Confirmer la suppression', defaultValue: false)
                ]
            )

            if (userInput) {
                withCredentials([aws(credentialsId: 'credentialsId', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    // Configurer AWS CLI avec les credentials
                    sh "aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}"
                    sh "aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}"
                    sh "aws configure set region ${AWS_REGION}"

                    // Construire le tag à partir du nom de la branche
                    def branchName = GIT_BRANCH // Nom de la branche
                    def tag = "review-${branchName}"

                    // Vérifier si une instance avec le tag existe déjà
                    def instancesCheckCommand = """
                        aws ec2 describe-instances \
                        --filters "Name=tag:Name,Values=${tag}" \
                        --query "Reservations[*].Instances[*].InstanceId" \
                        --output text
                    """

                    def existingInstance = sh(script: instancesCheckCommand, returnStdout: true).trim()

                    if (existingInstance) {
                        echo "Suppression de l'instance avec le tag '${tag}': ${existingInstance}"

                        // Commande pour supprimer l'instance EC2
                        def deleteInstanceCommand = """
                            aws ec2 terminate-instances \
                            --instance-ids ${existingInstance}
                        """

                        // Exécuter la commande de suppression
                        sh deleteInstanceCommand
                        echo "Instance EC2 supprimée avec le tag '${tag}'."
                    } else {
                        echo "Aucune instance avec le tag '${tag}' à supprimer."
                    }
                }
            } else {
                echo "Suppression annulée par l'utilisateur."
            }
        }
    }
}




        stage ('Deploy in staging') {
            when {
                expression { GIT_BRANCH == 'origin/staging' }
            }
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) { 
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa,dsa ${HOSTNAME_DEPLOY_REVIEW} >> ~/.ssh/known_hosts
                        scp -r deploy ubuntu@${HOSTNAME_DEPLOY_REVIEW}:/home/ubuntu/
                        command1="cd deploy && echo ${DOCKERHUB_AUTH_PSW} | docker login -u ${DOCKERHUB_AUTH_USR} --password-stdin"
                        command2="echo 'IMAGE_VERSION=${DOCKERHUB_AUTH_USR}/${IMAGE_NAME}:${IMAGE_TAG}' > .env && echo ${MYSQL_AUTH_PSW} > secrets/db_password.txt && echo ${MYSQL_AUTH_USR} > secrets/db_user.txt"
                        command3="echo 'SPRING_DATASOURCE_URL=jdbc:mysql://paymybuddydb:3306/db_paymybuddy' > env/paymybuddy.env && echo 'SPRING_DATASOURCE_PASSWORD=${MYSQL_AUTH_PSW}' >> env/paymybuddy.env && echo 'SPRING_DATASOURCE_USERNAME=${MYSQL_AUTH_USR}' >> env/paymybuddy.env"
                        command4="docker compose down && docker pull ${DOCKERHUB_AUTH_USR}/${IMAGE_NAME}:${IMAGE_TAG}"
                        command5="docker compose up -d"
                        ssh -t ubuntu@${HOSTNAME_DEPLOY_REVIEW} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -o SendEnv=MYSQL_AUTH_USR \
                            -o SendEnv=MYSQL_AUTH_PSW \
                            -C "$command1 && $command2 && $command3 && $command4 && $command5"
                    '''
                }
            }
        }

        stage('Test Staging') {
            when {
                expression { GIT_BRANCH == 'origin/staging' }
            }
            steps {
                sh '''
                    sleep 30
                    apk add --no-cache curl
                    curl ${HOSTNAME_DEPLOY_STAGING}
                '''
            }
        }

        stage ('Deploy in prod') {
            when {
                expression { GIT_BRANCH == 'origin/prod' }
            }
            
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) { 
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        ssh-keyscan -t rsa,dsa ${HOSTNAME_DEPLOY_PROD} >> ~/.ssh/known_hosts
                        scp -r deploy ubuntu@${HOSTNAME_DEPLOY_PROD}:/home/ubuntu/
                        command1="cd deploy && echo ${DOCKERHUB_AUTH_PSW} | docker login -u ${DOCKERHUB_AUTH_USR} --password-stdin"
                        command2="echo 'IMAGE_VERSION=${DOCKERHUB_AUTH_USR}/${IMAGE_NAME_PROD}:${IMAGE_TAG_PROD}' > .env && echo ${MYSQL_AUTH_PSW} > secrets/db_password.txt && echo ${MYSQL_AUTH_USR} > secrets/db_user.txt"
                        command3="echo 'SPRING_DATASOURCE_URL=jdbc:mysql://paymybuddydb:3306/db_paymybuddy' > env/paymybuddy.env && echo 'SPRING_DATASOURCE_PASSWORD=${MYSQL_AUTH_PSW}' >> env/paymybuddy.env && echo 'SPRING_DATASOURCE_USERNAME=${MYSQL_AUTH_USR}' >> env/paymybuddy.env"
                        command4="docker compose down && docker pull ${DOCKERHUB_AUTH_USR}/${IMAGE_NAME_PROD}:${IMAGE_TAG_PROD}"
                        command5="docker compose up -d"
                        ssh -t ubuntu@${HOSTNAME_DEPLOY_PROD} \
                            -o SendEnv=IMAGE_NAME_PROD \
                            -o SendEnv=IMAGE_TAG_PROD \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -o SendEnv=MYSQL_AUTH_USR \
                            -o SendEnv=MYSQL_AUTH_PSW \
                            -C "$command1 && $command2 && $command3 && $command4 && $command5"
                    '''
                }
            }
        }

        stage('Test Prod') {
            when {
                expression { GIT_BRANCH == 'origin/prod' }
            }
            steps {
                sh '''
                    sleep 30
                    apk add --no-cache curl
                    curl ${HOSTNAME_DEPLOY_PROD}
                '''
            }
        }
    }

    post {
        success {
            slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
        failure {
            slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
    }

}
