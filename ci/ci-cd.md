# Hydrophone CI/CD Pipeline Examples

This document provides practical examples for running Hydrophone in various CI/CD environments. These snippets are intended to help contributors and users integrate Hydrophone into their automated workflows for testing, building, and deployment.

## Why are we solving this issue?

Many users and contributors run Hydrophone in different environments and pipelines. Sharing example configurations for popular CI/CD systems will:
- Help new users get started quickly
- Promote best practices for testing and deployment
- Foster collaboration and knowledge sharing in the Hydrophone community

---

## Community Examples

If you use Hydrophone in your own CI/CD environment (Prow, GitHub Actions, GitLab, Jenkins, etc.), please consider sharing your configuration with the community! You can reach out via:

- [Kubernetes Slack #sig-testing channel](https://kubernetes.slack.com)
- [Kubernetes SIG Testing mailing list](https://groups.google.com/g/kubernetes-sig-testing)
- Opening a Pull Request with your example in this document

---

## Prow Example

Prow is the Kubernetes project's CI/CD system. To run Hydrophone tests with Prow, add a job to your `prow/config.yaml`:

```yaml
presubmits:
  kubernetes-sigs/hydrophone:
  - name: hydrophone-presubmit
    always_run: true
    decorate: true
    spec:
      containers:
      - image: golang:1.21
        command:
        - bash
        - -c
        - |
          go test ./...
```

Learn more: [Prow Documentation](https://github.com/kubernetes/test-infra/tree/master/prow)

---

## GitHub Actions Example

To run Hydrophone tests on every push and pull request, add the following workflow to `.github/workflows/ci.yaml`:

```yaml
name: Hydrophone CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'

    - name: Install dependencies
      run: go mod tidy

    - name: Run tests
      run: go test ./...
```

---

## GitLab CI Example

To run Hydrophone tests in GitLab, add the following to your `.gitlab-ci.yml`:

```yaml
stages:
  - test

test_hydrophone:
  image: golang:1.21
  stage: test
  script:
    - go mod tidy
    - go test ./...
```

---

## Jenkins Pipeline Example

To run Hydrophone tests in a Jenkins pipeline, use the following in your `Jenkinsfile`:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Setup Go') {
            steps {
                sh 'wget https://golang.org/dl/go1.21.linux-amd64.tar.gz'
                sh 'sudo tar -C /usr/local -xzf go1.21.linux-amd64.tar.gz'
                env.GOPATH = "${WORKSPACE}/go"
                env.PATH = "/usr/local/go/bin:$PATH"
            }
        }
        stage('Test') {
            steps {
                sh 'go mod tidy'
                sh 'go test ./...'
            }
        }
    }
}
```

---

## Need Help or Want to Share?

- If you have questions or would like to share your own Hydrophone CI/CD example, join the [#sig-testing Slack channel](https://kubernetes.slack.com) or [mailing list](https://groups.google.com/g/kubernetes-sig-testing).
- You can also open an issue or pull request in the Hydrophone repository.

---

## References

- [Prow Documentation](https://github.com/kubernetes/test-infra/tree/master/prow)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI Documentation](https://docs.gitlab.com/ee/ci/)
- [Jenkins Documentation](https://www.jenkins.io/doc/)
