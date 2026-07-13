pipeline {
    agent any

    // Jenkins runs locally and can't receive a GitHub webhook, so poll instead of
    // waiting for a push notification. This is the local-POC substitution for what
    // would be a webhook-triggered build in the real environment.
    triggers {
        pollSCM('* * * * *')
    }

    environment {
        AWS_ENDPOINT_URL      = 'http://localstack:4566'
        AWS_ACCESS_KEY_ID     = 'test'
        AWS_SECRET_ACCESS_KEY = 'test'
        AWS_DEFAULT_REGION    = 'us-east-1'
        S3_BUCKET             = 'poc-lambda-artifacts'
        GITOPS_REPO           = 'luis-pabon-tf/argo-ack-poc-gitops'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Package') {
            steps {
                script {
                    env.GIT_SHORT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.ZIP_KEY = "handler-${env.GIT_SHORT_SHA}.zip"
                }
                sh 'zip function.zip handler.py'
            }
        }

        stage('Upload to LocalStack S3') {
            steps {
                sh 'aws --endpoint-url $AWS_ENDPOINT_URL s3 cp function.zip s3://$S3_BUCKET/$ZIP_KEY'
            }
        }

        // No container registry / Argo CD Image Updater in this POC: LocalStack's free
        // Hobby plan doesn't cover ECR, and container-image-packaged Lambda functions
        // are a Pro-only feature (see NOTES.md). This stage is the substitution called
        // out in the build plan's Phase 2/7: Jenkins opens the gitops PR directly.
        stage('Open gitops PR') {
            steps {
                withCredentials([string(credentialsId: 'gitops-pr-pat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        rm -rf gitops-checkout
                        git clone https://x-access-token:${GITHUB_TOKEN}@github.com/${GITOPS_REPO}.git gitops-checkout
                        cd gitops-checkout
                        git checkout -b bump-${ZIP_KEY}
                        sed -i "s|s3Key: .*|s3Key: ${ZIP_KEY}|" apps/sample-function/function.yaml
                        git config user.email "jenkins-poc@example.com"
                        git config user.name "jenkins-poc"
                        git commit -am "Bump sample-function to ${ZIP_KEY}"
                        git push origin bump-${ZIP_KEY}
                        curl -sf -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                          -H "Accept: application/vnd.github+json" \
                          "https://api.github.com/repos/${GITOPS_REPO}/pulls" \
                          -d "{\\"title\\":\\"Bump sample-function to ${ZIP_KEY}\\",\\"head\\":\\"bump-${ZIP_KEY}\\",\\"base\\":\\"main\\"}"
                    '''
                }
            }
        }
    }
}
