pipeline {
    agent { label 'aws-ready' }

    environment {
        commit = sh(returnStdout: true, script: "git rev-parse --short=8 HEAD").trim()
        aws_region = "${sh(script:'aws configure get region', returnStdout: true).trim()}"
        terraform_directory = "terraform"
        resource_directory = "/var/lib/jenkins-worker-node/AM-resources"
    }

    parameters {
        booleanParam(name: "APPLY", defaultValue: false)
    }

    stages {

        stage('Terraform Plan') {
            steps {
                echo 'Planning terraform infrastructure'
                dir("${terraform_directory}") {
                    sh 'mkdir -p plans'
                    sh 'terraform init -no-color'
                    sh 'terraform plan -out plans/plan-${commit}.tf -no-color > plans/plan-${commit}.txt'
                }
            }
        }

        stage('Terraform Apply') {
            when { expression { params.APPLY } }
            steps {
                echo 'Applying Terraform objects'
                dir("${terraform_directory}") {
                    sh 'terraform apply -no-color -auto-approve plans/plan-${commit}.tf'
                }
            }
        }

        stage('Terraform Output') {
            when { expression { params.APPLY } }
            steps {
                echo 'Exporting outputs as secrets'
                dir("${terraform_directory}") {
                    sh 'terraform refresh'
                    sh 'terraform output | tr -d \'\\\"\\ \' > ${resource_directory}/env.tf'
                }
                script {
                    withCredentials([
                        string (credentialsId: 'dev/AM/utopia-secrets',
                        variable: 'DB_CREDS')
                    ]) {
                        // json objects
                        def creds = readJSON text: DB_CREDS
                        def outputs = readProperties file: env.resource_directory + '/env.tf'

                        // secret keys
                        creds.AWS_VPC_ID          = outputs.AWS_VPC_ID
                        creds.AWS_RDS_ENDPOINT    = outputs.AWS_RDS_ENDPOINT
                        creds.AWS_ALB_ID          = outputs.AWS_ALB_ID

                        // rewrite secret
                        String jsonOut = writeJSON returnText: true, json: creds
                        sh "aws secretsmanager update-secret --secret-id 'arn:aws:secretsmanager:us-west-2:026390315914:secret:dev/AM/utopia-secrets-NE4x9z' --secret-string '${jsonOut}'"
                    }
                }
            }
        }

        //end
    }
}
