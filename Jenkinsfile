pipeline {
    agent any
    stages {
        stage('Clean Caches') {
            steps {
                echo 'Cleaning Gradle and npm caches'
                sh '''
                    ./gradlew --stop || true
                    ./gradlew clean || true
                    rm -rf .gradle
                    rm -rf ~/.gradle
                    rm -rf ~/.npm
                    rm -rf node_modules
                    rm -rf /var/lib/jenkins/.npm
                    rm -rf /var/lib/jenkins/.gradle
                '''
            }
        }
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
                script {
                    app = docker.build("facundoforcada/train-schedule")
                    app.inside {
                        sh 'echo $(curl localhost:8080)'
                    }
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
        stage ('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production'
                milestone(1)
                withCredentials ([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@${env.prod_ip} \"docker pull facundoforcada/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                           sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@${env.prod_ip} \"docker stop train-schedule\""
                           sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@${env.prod_ip} \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@${env.prod_ip} \"docker run --restart always --name train-schedule -p 8080:8080 -d facundoforcada/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }   
}
