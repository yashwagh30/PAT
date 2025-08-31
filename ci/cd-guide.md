# CI/CD Integration Guide

This guide provides examples for running Hydrophone in various CI/CD environments to automate Conformance Report generation.

## General Principles

When integrating Hydrophone into CI/CD, keep these points in mind:
*   **Authentication:** The Hydrophone container needs a `kubeconfig` file or token with sufficient permissions to access the target cluster.
*   **Secrets Management:** Never hardcode credentials. Use your CI/CD system's built-in secrets store (e.g., GitHub Secrets, GitLab CI Variables) to securely pass the `kubeconfig` or its contents.
*   **Output:** By default, Hydrophone saves the report to `./conformance-report.html`. You will likely want to add a step to upload this artifact.

## Examples

### GitHub Actions

This workflow example assumes you have stored a base64-encoded kubeconfig file as a secret in your GitHub repository named `KUBECONFIG_B64`.

1.  Create a secret in your GitHub repository settings named `KUBECONFIG_B64` containing the output of `cat ~/.kube/config | base64 -w 0`.
2.  Create a workflow file (e.g., `.github/workflows/conformance.yaml`):

```yaml
name: Kubernetes Conformance Test
on:
  workflow_dispatch: # Allows manual triggering
  schedule:
    - cron: '0 0 * * 0' # Run weekly on Sunday

jobs:
  conformance:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Decode Kubeconfig
        run: |
          echo "${{ secrets.KUBECONFIG_B64 }}" | base64 -d > kubeconfig.yaml
        shell: bash

      - name: Run Hydrophone
        uses: docker://gcr.io/k8s-staging-conformance/hydrophone:latest
        with:
          args: --kubeconfig=/kubeconfig.yaml
        env:
          # Mount the decoded kubeconfig into the container
          KUBECONFIG: /kubeconfig.yaml

      - name: Upload Conformance Report
        uses: actions/upload-artifact@v4
        with:
          name: conformance-report
          path: conformance-report.html
```

### GitLab CI

This example uses a GitLab CI variable to store the entire `kubeconfig` content.

1.  In your GitLab project, go to **Settings > CI/CD > Variables**. Add a variable named `KUBECONFIG_CONTENT` and paste the entire contents of your `~/.kube/config` file. **Mask the variable** if possible.
2.  Create a job in your `.gitlab-ci.yml` file:

```yaml
generate_conformance_report:
  image: docker:latest
  services:
    - docker:dind
  variables:
    # Mount the kubeconfig file from the variable
    KUBECONFIG: /kubeconfig.yaml
  before_script:
    - echo "$KUBECONFIG_CONTENT" > kubeconfig.yaml
  script:
    - docker run -v $(pwd)/kubeconfig.yaml:/.kube/config gcr.io/k8s-staging-conformance/hydrophone:latest --kubeconfig=/.kube/config
  artifacts:
    paths:
      - conformance-report.html
    expire_in: 1 week
```

### Jenkins Pipeline

This example uses the Jenkins `withCredentials` directive to bind a credential containing the kubeconfig file.

1.  In Jenkins, add a credential of type "Secret file" (e.g., with an ID like `cluster-kubeconfig`).
2.  Create a `Jenkinsfile` in your project:

```groovy
pipeline {
    agent any
    environment {
        KUBECONFIG = "${WORKSPACE}/kubeconfig.yaml"
    }
    stages {
        stage('Generate Conformance Report') {
            steps {
                // This step copies the credential file to the workspace
                withCredentials([file(credentialsId: 'cluster-kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh 'cp $KUBECONFIG_FILE ./kubeconfig.yaml'
                }
                sh 'docker run -v $(pwd)/kubeconfig.yaml:/.kube/config gcr.io/k8s-staging-conformance/hydrophone:latest --kubeconfig=/.kube/config'
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'conformance-report.html', fingerprint: true
        }
    }
}
```

### Prow

(Prow is heavily used in Kubernetes core development. An example might look more complex and involve a `PodSpec`.)

```yaml
# This is a basic example of a prow job config (prowjob.yaml).
# It would typically be configured by a cluster administrator.
apiVersion: batch/v1
kind: Job
metadata:
  name: hydrophone-conformance-test
spec:
  template:
    spec:
      containers:
      - name: hydrophone
        image: gcr.io/k8s-staging-conformance/hydrophone:latest
        args: ["--kubeconfig=/path/to/incluster/kubeconfig"]
        # ... other required volumes and mounts for in-cluster config
      restartPolicy: Never
```

**Note for Prow:** In-cluster configuration is often handled automatically. This is a advanced example and users may need to consult their cluster administrator.

## Need Help?

If your CI system isn't listed here, or you run into problems, please open an issue on GitHub or reach out to the community on the [Kubernetes Slack](https://slack.k8s.io/) in the `#sig-testing` or `#sig-architecture` channels.
