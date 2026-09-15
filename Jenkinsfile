
pipeline {
    agent { label "agent-01" }

    stages {

        stage('Clone') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sagarpyakurel/jenkins_project_django-notes-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'this it building docker image stage'
                sh 'docker compose build'
                sh 'docker images'
            }
        }

          stage('DockerHub login and push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker_token',
                        usernameVariable: 'docker_username',
                        passwordVariable: 'docker_password'
                        )
                ]) {
                    sh 'echo "$docker_password" | docker login -u "$docker_username" --password-stdin'
                    sh "docker image tag django_app:latest $docker_username/django_app:latest"
                    sh "docker push $docker_username/django_app:latest"
                    }
                
                    echo "login successful and pushing successful"
                }
            } 

            stage('Deploy') {
            steps {
                sh '''
                    docker compose -f docker-compose-deploy.yml pull
                    docker compose -f docker-compose-deploy.yml up -d
                    docker ps
                '''
                echo "------->>  Deployment successful <<------"
            }
        }

    }
}










