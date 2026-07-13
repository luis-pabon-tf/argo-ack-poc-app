# argo-ack-poc-app

Trivial Lambda handler used as the app repo for the Argo CD + ACK GitOps POC. On merge
to `main`, Jenkins zips `handler.py`, uploads it to the LocalStack-emulated S3 bucket
`poc-lambda-artifacts`, and opens a PR against
[argo-ack-poc-gitops](https://github.com/luis-pabon-tf/argo-ack-poc-gitops) bumping the
`Function` CR's `S3Key` to the new zip.
