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
                script {
                    // ACK's lambda-controller diffs spec against what AWS returns. S3Bucket/S3Key
                    // are write-only on Lambda's API and never come back from DescribeFunction, so
                    // bumping s3Key alone is invisible to the controller's reconcile diff. sha256 IS
                    // diffable (compared against the AWS-reported CodeSha256), so it has to travel
                    // alongside s3Key on every deploy or the CR silently never redeploys.
                    env.ZIP_SHA256 = sh(
                        script: 'python3 -c "import hashlib,base64; print(base64.b64encode(hashlib.sha256(open(\'function.zip\',\'rb\').read()).digest()).decode())"',
                        returnStdout: true
                    ).trim()
                }
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
        stage('Open and auto-merge gitops PR') {
            steps {
                withCredentials([string(credentialsId: 'gitops-pr-pat', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        rm -rf gitops-checkout
                        git clone https://x-access-token:${GITHUB_TOKEN}@github.com/${GITOPS_REPO}.git gitops-checkout
                        cd gitops-checkout
                        git checkout -b bump-${ZIP_KEY}
                        sed -i "s|s3Key: .*|s3Key: ${ZIP_KEY}|" apps/sample-function/function.yaml
                        sed -i "s|sha256: .*|sha256: ${ZIP_SHA256}|" apps/sample-function/function.yaml
                        git config user.email "jenkins-poc@example.com"
                        git config user.name "jenkins-poc"
                        git commit -am "Bump sample-function to ${ZIP_KEY}"

                        # Safety guard: this PR auto-merges with no human review, so refuse
                        # to proceed unless the diff is exactly the two-line s3Key+sha256 bump
                        # we expect. Anything else (a bug, a bad rebase, tampering) stops here
                        # loudly instead of silently auto-merging.
                        CHANGED_FILES=$(git diff HEAD~1 HEAD --name-only)
                        if [ "$CHANGED_FILES" != "apps/sample-function/function.yaml" ]; then
                          echo "ABORT: unexpected files changed: $CHANGED_FILES"
                          exit 1
                        fi
                        DIFF_LINES=$(git diff HEAD~1 HEAD -- apps/sample-function/function.yaml | grep -E "^[+-]" | grep -v "^[+-][+-][+-]")
                        if [ "$(echo "$DIFF_LINES" | wc -l)" != "4" ] || echo "$DIFF_LINES" | grep -qvE "s3Key:|sha256:"; then
                          echo "ABORT: unexpected diff content:"
                          echo "$DIFF_LINES"
                          exit 1
                        fi

                        git push origin bump-${ZIP_KEY}

                        PR_NUMBER=$(curl -sf -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                          -H "Accept: application/vnd.github+json" \
                          "https://api.github.com/repos/${GITOPS_REPO}/pulls" \
                          -d "{\\"title\\":\\"Bump sample-function to ${ZIP_KEY}\\",\\"head\\":\\"bump-${ZIP_KEY}\\",\\"base\\":\\"main\\"}" \
                          | python3 -c "import json,sys; print(json.load(sys.stdin)[\\"number\\"])")
                        echo "Opened PR #${PR_NUMBER}"

                        # mergeable is computed asynchronously right after creation; poll briefly.
                        for i in 1 2 3 4 5; do
                          MERGEABLE=$(curl -sf -H "Authorization: token ${GITHUB_TOKEN}" \
                            "https://api.github.com/repos/${GITOPS_REPO}/pulls/${PR_NUMBER}" \
                            | python3 -c "import json,sys; print(json.load(sys.stdin)[\\"mergeable\\"])")
                          if [ "$MERGEABLE" = "True" ]; then break; fi
                          sleep 2
                        done

                        curl -sf -X PUT -H "Authorization: token ${GITHUB_TOKEN}" \
                          -H "Accept: application/vnd.github+json" \
                          "https://api.github.com/repos/${GITOPS_REPO}/pulls/${PR_NUMBER}/merge" \
                          -d "{\\"merge_method\\":\\"squash\\"}"

                        curl -sf -X DELETE -H "Authorization: token ${GITHUB_TOKEN}" \
                          "https://api.github.com/repos/${GITOPS_REPO}/git/refs/heads/bump-${ZIP_KEY}"
                    '''
                }
            }
        }
    }
}
