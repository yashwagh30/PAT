# CI/CD Integration Guide

This guide provides examples for running Hydrophone in various CI/CD environments to automate the generation of Kubernetes Conformance Reports.

## General Principles

When integrating Hydrophone into CI/CD, keep these points in mind:
*   **Authentication:** The Hydrophone container must have a `kubeconfig` file with sufficient permissions to access the resources in the target cluster.
*   **Secrets Management:** Never hardcode credentials. Use your CI/CD system's built-in secrets store (e.g., GitHub Secrets, GitLab CI Variables) to securely pass the `kubeconfig` or its contents.
*   **Output:** Hydrophone writes the report to the path specified by the `--output` flag, which defaults to `./conformance-report.html`. You will need to use a volume mount to make this file available to the CI/CD host system for uploading/archiving.

## Examples

### GitHub Actions

This workflow example assumes you have stored the entire contents of a valid `kubeconfig` file as a secret in your GitHub repository named `KUBECONFIG_CONTENT`.

1.  **Create the Secret:** In your GitHub repository, go to **Settings > Secrets and variables > Actions**. Create a New Repository Secret named `KUBECONFIG_CONTENT` and paste your complete kubeconfig file contents.
2.  **Create the Workflow File:** Create the file `.github/workflows/conformance.yaml` in your repository with the following content:

```yaml
name: Generate Conformance Report
on:
  workflow_dispatch: # Allows manual triggering
  schedule:
    - cron: '0 0 * * 0' # Run weekly on Sunday at 00:00 UTC

jobs:
  conformance:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Write Kubeconfig
        run: |
          # Create the .kube directory and write the secret content to a file
          mkdir -p .kube
          echo "${{ secrets.KUBECONFIG_CONTENT }}" > .kube/config
        shell: bash

      - name: Run Hydrophone
        # Use the Docker container directly as an action.
        # Mount the .kube directory and the current workspace into the container.
        uses: docker://gcr.io/k8s-staging-conformance/hydrophone:latest
        with:
          # The args are passed to the Hydrophone entrypoint inside the container.
          # The kubeconfig is now at the mount path, and the output will be written to the workspace.
          args: --kubeconfig=/.kube/config --quiet
        env:
          # Mount the host's filesystem paths into the container
          KUBECONFIG: /.kube/config
          # The Hydrophone container's WORKDIR is '/app', so we mount the workspace there.
          # This ensures the output file 'conformance-report.html' is written to the GitHub Actions workspace.
          HOST_WORKSPACE: /app

      - name: Upload Conformance Report Artifact
        uses: actions/upload-artifact@v4
        with:
          name: conformance-report
          path: conformance-report.html # This is now in the workspace root
```

### GitLab CI

This example uses a GitLab CI file variable to store the entire `kubeconfig` content.

1.  **Create the Variable:** In your GitLab project, go to **Settings > CI/CD > Variables**. Add a variable named `KUBECONFIG_CONTENT` and paste the entire contents of your kubeconfig file. **Set the variable to be masked and protected** if possible.
2.  **Create the Job:** Add the following job to your `.gitlab-ci.yml` file:

```yaml
generate_conformance_report:
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - mkdir -p .kube
    - echo "$KUBECONFIG_CONTENT" > .kube/config
  script:
    - |
      docker run \
        --volume "$(pwd)/.kube:/.kube" \
        --volume "$(pwd):/app" \
        --workdir /app \
        gcr.io/k8s-staging-conformance/hydrophone:latest \
        --kubeconfig=/.kube/config \
        --quiet
  artifacts:
    paths:
      - conformance-report.html
    expire_in: 1 week
```

### Jenkins Pipeline

This example uses the Jenkins `withCredentials` directive to bind a credential containing the kubeconfig file.

1.  **Create the Credential:** In Jenkins, add a credential of type "Secret file". Upload your kubeconfig file and note its ID (e.g., `cluster-kubeconfig`).
2.  **Create the Pipeline:** Create a `Jenkinsfile` in your project with the following content:

```groovy
pipeline {
    agent any
    stages {
        stage('Generate Conformance Report') {
            steps {
                script {
                    // This step securely copies the credential file to the workspace
                    withCredentials([file(credentialsId: 'cluster-kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            mkdir -p .kube
                            cp "$KUBECONFIG_FILE" .kube/config
                        '''
                    }
                    // Run Hydrophone, mounting the .kube dir and workspace
                    sh '''
                        docker run \\
                          --volume "$(pwd)/.kube:/.kube" \\
                          --volume "$(pwd):/app" \\
                          --workdir /app \\
                          gcr.io/k8s-staging-conformance/hydrophone:latest \\
                          --kubeconfig=/.kube/config \\
                          --quiet
                    '''
                }
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

Integration with Prow typically involves defining a `PodSpec` in a prow job configuration. This is often managed by cluster administrators.

```yaml
# Example snippet from a prow job config (prowjob.yaml)
# This runs Hydrophone with in-cluster authentication (e.g., a service account).
spec:
  containers:
  - name: hydrophone
    image: gcr.io/k8s-staging-conformance/hydrophone:latest
    args:
      - "--kubeconfig=/etc/kubeconfig/config" # Path if using a mounted in-cluster config
      - "--output=/logs/artifacts/conformance-report.html" # Prow expects artifacts in /logs/artifacts/
    volumeMounts:
    - name: kubeconfig
      mountPath: /etc/kubeconfig
      readOnly: true
    # ... other required volumes
  volumes:
  - name: kubeconfig
    secret:
      secretName: my-cluster-kubeconfig # Secret created by an admin
```

**Note for Prow:** This is an advanced example. The exact setup depends heavily on how the Prow cluster is configured. Users should consult their cluster administrator.

## Need Help?

If your CI system isn't listed here, or you run into problems, please open an issue on GitHub or reach out to the community on the [Kubernetes Slack](https://slack.k8s.io/) in the `#sig-testing` or `#sig-architecture` channels.
