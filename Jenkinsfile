pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                #This is to allow me to use shell command as the code does not support delarative pipeline
                script {
                    app = docker.build("robinodocker/train-schedule") # build image from my docker hub
                    app.inside {
                        sh 'echo $(curl localhost:8080)' #Test the image with curl command
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                #This is to allow me to use shell command as the code does not support delarative pipeline
                script {
                    #using default docker registry to push the image with the credentials created in Jenkins
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login_new') {
                        app.push("${env.BUILD_NUMBER}")#versioning the build
                        app.push("latest")#latest will be appended to the end of the image name
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'ubuntu-ansible-login', usernameVariable: 'USER', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USER@$prod_ip \"docker pull robinodocker/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USER@$prod_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USER@$prod_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USER@$prod_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d robinodocker/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
