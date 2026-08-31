pipeline {
    agent any

    environment {
        TARGET_VM_HOST = "13.233.136.212"
        TARGET_VM_USER = "ec2-user"
        DEPLOY_PATH    = "/var/www/html"
        BACKUP_PATH    = "/var/www/app_backup"
        TMP_PATH       = "/tmp/deploy"
        SSH_CREDS      = "AWS_SSH"
        GIT_REPO       = "https://github.com/anuragdahiya1/practice.git"
        GIT_BRANCH     = "main"
        GIT_CREDS      = "GitHub"
        APP_URL        = "http://ec2-13-233-136-212.ap-south-1.compute.amazonaws.com"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: env.GIT_BRANCH,
                    url: env.GIT_REPO,
                    credentialsId: env.GIT_CREDS
            }
        }

        stage('Backup Current Version') {
            steps {
                sshagent([env.SSH_CREDS]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${env.TARGET_VM_USER}@${env.TARGET_VM_HOST} '
                        sudo rm -rf ${env.BACKUP_PATH} &&
                        sudo cp -r ${env.DEPLOY_PATH} ${env.BACKUP_PATH}
                    '
                    """
                }
            }
        }

        stage('Deploy New Version') {
            steps {
                sshagent([env.SSH_CREDS]) {
                    sh """
                    scp -o StrictHostKeyChecking=no -r * ${env.TARGET_VM_USER}@${env.TARGET_VM_HOST}:${env.TMP_PATH}/
                    ssh -o StrictHostKeyChecking=no ${env.TARGET_VM_USER}@${env.TARGET_VM_HOST} '
                        sudo rm -rf ${env.DEPLOY_PATH}/* &&
                        sudo cp -r ${env.TMP_PATH}/* ${env.DEPLOY_PATH}/ &&
                        sudo rm -rf ${env.TMP_PATH}/* &&
                        sudo systemctl restart httpd.service
                    '
                    """
                }
            }
        }

        stage('Smoke Test') {
            steps {
                script {
                    def response = sh(
                        script: "curl -s -o /dev/null -w '%{http_code}' ${env.APP_URL}",
                        returnStdout: true
                    ).trim()
                    if (response != '200') {
                        error "Smoke test failed! URL did not return 200 OK"
                    } else {
                        echo "Smoke test passed — site is live with new code."
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "Pipeline failed — rolling back to previous version..."
            sshagent([env.SSH_CREDS]) {
                sh """
                ssh -o StrictHostKeyChecking=no ${env.TARGET_VM_USER}@${env.TARGET_VM_HOST} '
                    sudo rm -rf ${env.DEPLOY_PATH} &&
                    sudo cp -r ${env.BACKUP_PATH} ${env.DEPLOY_PATH} &&
                    sudo systemctl restart httpd.service
                '
                """
            }
        }
        success {
            echo "Pipeline completed successfully for MAIN branch!"
        }
    }
}
