pipeline {
    agent any

    stages {

        stage('Build') {
            agent {
                docker {
                    image 'gradle:6.9.4-jdk11'
                    reuseNode true
                }
            }
            environment {
                HOME = "${WORKSPACE}"
                NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
                NPM_CONFIG_UPDATE_NOTIFIER = "false"
                GRADLE_USER_HOME = "${WORKSPACE}/.gradle"
            }
            steps {
                echo 'Running build automation'
                sh 'mkdir -p $HOME $NPM_CONFIG_CACHE $GRADLE_USER_HOME'
                sh 'chmod +x gradlew'
                sh './gradlew build --no-daemon -x npm_test'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }

        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build("cloudusers/train-schedule")
                }
            }
        }

        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }

        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production'
                withCredentials([usernamePassword(
                    credentialsId: 'webserver_login',
                    usernameVariable: 'USERNAME',
                    passwordVariable: 'USERPASS'
                )]) {
                    script {
                        sh """
                        sshpass -p '$USERPASS' ssh -o StrictHostKeyChecking=no $USERNAME@${env.prod_ip} \
                        "docker pull cloudusers/train-schedule:${env.BUILD_NUMBER}"
                        """

                        sh """
                        sshpass -p '$USERPASS' ssh -o StrictHostKeyChecking=no $USERNAME@${env.prod_ip} \
                        "docker rm -f train-schedule || true"
                        """

                        sh """
                        sshpass -p '$USERPASS' ssh -o StrictHostKeyChecking=no $USERNAME@${env.prod_ip} \
                        "docker run -d --name train-schedule -p 8080:8080 --restart always cloudusers/train-schedule:${env.BUILD_NUMBER}"
                        """
                    }
                }
            }
        }
    }
}
