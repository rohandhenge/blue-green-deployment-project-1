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
                git 'https://github.com/rohandhenge/blue-green-deployment-project-1.git'
            }
        }

        stage('Detect Active Environment') {
            steps {
                script {
                    ACTIVE_TG = sh(
                        script: "aws elbv2 describe-listeners --listener-arns $arn:aws:elasticloadbalancing:ap-southeast-1:427617722045:listener/app/blue-green-alb/a9a6c7bee070b638/bbe59b8a7bbb918a --query 'Listeners[0].DefaultActions[0].TargetGroupArn' --output text",
                        returnStdout: true
                    ).trim()

                    if (ACTIVE_TG.contains("blue")) {
                        env.INACTIVE_ENV = "green"
                        env.DEPLOY_SERVER = env.GREEN_INSTANCE
                        env.NEW_TG = env.GREEN_TG
                        env.OLD_TG = env.BLUE_TG
                    } else {
                        env.INACTIVE_ENV = "blue"
                        env.DEPLOY_SERVER = env.BLUE_INSTANCE
                        env.NEW_TG = env.BLUE_TG
                        env.OLD_TG = env.GREEN_TG
                    }

                    echo "Deploying to ${env.INACTIVE_ENV}"
                }
            }
        }

        stage('Deploy to Inactive Environment') {
            steps {
                sh """
                scp -o StrictHostKeyChecking=no -r * ${DEPLOY_SERVER}:${/var/www/html}

                ssh ${DEPLOY_SERVER} '
                sudo systemctl restart nginx
                '
                """
            }
        }

        stage('Health Check') {
            steps {
                script {
                    sleep 20

                    def status = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://${DEPLOY_SERVER}",
                        returnStdout: true
                    ).trim()

                    if (status != "200") {
                        error("Health check failed")
                    }

                    echo "Health check passed"
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                sh """
                aws elbv2 modify-listener \
                --listener-arn ${arn:aws:elasticloadbalancing:ap-southeast-1:427617722045:listener/app/blue-green-alb/a9a6c7bee070b638/bbe59b8a7bbb918a} \
                --default-actions Type=forward,TargetGroupArn=$(aws elbv2 describe-target-groups --names ${NEW_TG} --query 'TargetGroups[0].TargetGroupArn' --output text)
                """
            }
        }

        stage('Post Switch Validation') {
            steps {
                script {
                    sleep 15

                    def status = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' http://blue-green-alb-1890572643.ap-southeast-1.elb.amazonaws.com",
                        returnStdout: true
                    ).trim()

                    if (status != "200") {

                        echo "Deployment Failed. Rolling back..."

                        sh """
                        aws elbv2 modify-listener \
                        --listener-arn ${arn:aws:elasticloadbalancing:ap-southeast-1:427617722045:listener/app/blue-green-alb/a9a6c7bee070b638/bbe59b8a7bbb918a} \
                        --default-actions Type=forward,TargetGroupArn=$(aws elbv2 describe-target-groups --names ${OLD_TG} --query 'TargetGroups[0].TargetGroupArn' --output text)
                        """

                        error("Rollback executed")
                    }

                    echo "Deployment Successful 🚀"
                }
            }
        }
    }
}
