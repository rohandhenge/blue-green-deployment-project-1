 pipeline {
    agent any

    environment {
        AWS_REGION = "ap-southeast-1"

        BLUE_TG = "blue-target"
        GREEN_TG = "green-target"

        ALB_LISTENER_ARN = "arn:aws:elasticloadbalancing:ap-southeast-1:427617722045:listener/app/blue-green-alb/a9a6c7bee070b638/bbe59b8a7bbb918a"

        BLUE_INSTANCE = "ec2-user@172.31.31.175"
        GREEN_INSTANCE = "ec2-user@47.129.216.120"

        APP_PATH = "/var/www/html"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/rohandhenge/blue-green-deployment-project-1.git'
            }
        }

        stage('Detect Active Environment') {
            steps {
                script {

                    def ACTIVE_TG = sh(
                        script: "aws elbv2 describe-listeners --listener-arns $ALB_LISTENER_ARN --query 'Listeners[0].DefaultActions[0].TargetGroupArn' --output text",
                        returnStdout: true
                    ).trim()

                    if (ACTIVE_TG.contains("blue")) {
                        env.DEPLOY_SERVER = env.GREEN_INSTANCE
                        env.NEW_TG = env.GREEN_TG
                    } else {
                        env.DEPLOY_SERVER = env.BLUE_INSTANCE
                        env.NEW_TG = env.BLUE_TG
                    }

                    echo "Deploying to ${env.DEPLOY_SERVER}"
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                scp -o StrictHostKeyChecking=no -r * ${DEPLOY_SERVER}:${APP_PATH}

                ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} '
                sudo systemctl restart nginx
                '
                """
            }
        }

        stage('Switch Traffic') {
            steps {
                sh """
                aws elbv2 modify-listener \
                --listener-arn ${ALB_LISTENER_ARN} \
                --default-actions Type=forward,TargetGroupArn=\$(aws elbv2 describe-target-groups --names ${NEW_TG} --query 'TargetGroups[0].TargetGroupArn' --output text)
                """
            }
        }
    }
}
