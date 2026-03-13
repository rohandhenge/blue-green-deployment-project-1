pipeline {
    agent any

    stages {

        stage('Clone Code') {
            steps {
                git ' https://github.com/rohandhenge/blue-green-deployment-project-1.git'
            }
        }

        stage('Deploy to Green Server') {
            steps {
                sh '''
                scp -o StrictHostKeyChecking=no index.html ec2-user@172.31.22.25:/tmp/

                ssh -o StrictHostKeyChecking=no ec2-user@172.31.22.25 "
                sudo mv /tmp/index.html /usr/share/nginx/html/index.html
                sudo systemctl restart nginx
                "
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh 'curl -f http://172.31.22.25'
            }
        }

        stage('Deployment Successful') {
            steps {
                echo "Green Environment Ready"
            }
        }

    }
}
