pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1'
        SONARQUBE_URL = "http://localhost:9000"
        SONAR_SCANNER_HOME = "/opt/sonar-scanner"
        TRUFFLEHOG_PATH = "/usr/local/bin/trufflehog3"
        JIRA_SITE = "https://daquietstorm22.atlassian.net/"
        JIRA_PROJECT = "JENKINS"
    }

    stages {
        stage('Set AWS Credentials') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'Jenkins3'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/tiqsclass6/jenkins-test3'
            }
        }

        stage('Static Code Analysis (SAST)') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'SonarQube_Jenkins', variable: 'SONAR_TOKEN')]) {
                        def scanStatus = sh(script: '''
                            ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=tiqsclass6_jenkins-test3 \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.login=''' + SONAR_TOKEN, returnStatus: true)

                        if (scanStatus != 0) {
                            createJiraTicket("Static Code Analysis Failed", "SonarQube scan detected issues in your code.")
                            error("SonarQube found security vulnerabilities!")
                        }
                    }
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                        sh "snyk auth ${SNYK_TOKEN}"
                        sh "snyk test --severity-threshold=high || exit 1"
                        sh "snyk monitor || echo 'No supported files found, monitoring skipped.'"
                    }
                }
            }
        }

        stage('Validate Terraform') {
            steps {
                sh 'terraform validate'
            }
        }

        stage('Plan Terraform') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'Jenkins3'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Apply Terraform') {
            steps {
                input message: "Approve Terraform Apply?", ok: "Deploy"
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'Jenkins3'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }
    }

    post {
        success { echo 'Terraform deployment completed successfully!' }
        failure { echo 'Terraform deployment failed!' }
    }
}