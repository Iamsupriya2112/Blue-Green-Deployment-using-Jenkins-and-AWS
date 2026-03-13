pipeline {
    agent any

    environment {
        BLUE_IP      = '65.0.6.146'
        GREEN_IP     = '3.110.168.77'
        BLUE_TG      = 'arn:aws:elasticloadbalancing:ap-south-1:015442728986:targetgroup/Blue-TG/9de93c7cc35e114a'
        GREEN_TG     = 'arn:aws:elasticloadbalancing:ap-south-1:015442728986:targetgroup/Green-TG/d0381963d61072bb'
        LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-south-1:015442728986:listener/app/Blue-Green-ALB/13aff2d8f59ffcac/937da5d9b97ab11c'
        ACTIVE_ENV   = 'blue'
    }

    stages {

        stage('Determine Inactive Environment') {
            steps {
                script {
                    def INACTIVE_ENV = (ACTIVE_ENV == 'blue') ? 'green' : 'blue'
                    env.INACTIVE_ENV = INACTIVE_ENV
                    echo "Deploying to ${INACTIVE_ENV}"
                }
            }
        }

        stage('Deploy Code from GitHub') {
            steps {
                sshagent(['project-2-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.INACTIVE_ENV == 'blue' ? BLUE_IP : GREEN_IP} "sudo rm -rf /var/www/html && sudo git clone https://github.com/Iamsupriya2112/blue-green-app.git /var/www/html"
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {

                    def URL = (env.INACTIVE_ENV == 'blue') ? "http://${BLUE_IP}" : "http://${GREEN_IP}"

                    def status = sh(
                        script: "curl -s -o /dev/null -w \"%{http_code}\" ${URL}",
                        returnStdout: true
                    ).trim()

                    if (status != "200") {
                        error "Health check failed!"
                    }

                }
            }
        }

        stage('Switch Traffic') {
            steps {

                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']]) {

                    sh """
                    aws elbv2 modify-listener \
                    --listener-arn ${LISTENER_ARN} \
                    --default-actions Type=forward,TargetGroupArn=${env.INACTIVE_ENV == 'blue' ? BLUE_TG : GREEN_TG}
                    """

                }

                script {
                    ACTIVE_ENV = env.INACTIVE_ENV
                    echo "Traffic switched to ${ACTIVE_ENV}"
                }
            }
        }

        stage('Check Target Group Health') {
            steps {
                script {

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']]) {

                        def ACTIVE_TG = (ACTIVE_ENV == 'blue') ? BLUE_TG : GREEN_TG
                        def OTHER_TG  = (ACTIVE_ENV == 'blue') ? GREEN_TG : BLUE_TG

                        def result = sh(
                            script: "aws elbv2 describe-target-health --target-group-arn ${ACTIVE_TG} --output text",
                            returnStdout: true
                        ).trim()

                        if (result.contains("unhealthy")) {

                            echo "Active environment unhealthy. Switching traffic."

                            sh """
                            aws elbv2 modify-listener \
                            --listener-arn ${LISTENER_ARN} \
                            --default-actions Type=forward,TargetGroupArn=${OTHER_TG}
                            """

                            ACTIVE_ENV = (ACTIVE_ENV == 'blue') ? 'green' : 'blue'

                            echo "Traffic switched to ${ACTIVE_ENV}"

                        } else {

                            echo "Environment healthy"

                        }

                    }

                }
            }
        }
    }

    post {
        failure {
            script {

                echo "Deployment failed. Rolling back."

                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-cred']]) {

                    def PREV_TG = (ACTIVE_ENV == 'blue') ? BLUE_TG : GREEN_TG

                    sh """
                    aws elbv2 modify-listener \
                    --listener-arn ${LISTENER_ARN} \
                    --default-actions Type=forward,TargetGroupArn=${PREV_TG}
                    """
                }

            }
        }
    }

}