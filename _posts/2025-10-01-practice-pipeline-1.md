---
title: 'Practice - Pipeline with Juice Shop'
date: 2025-10-01
permalink: /posts/practice-pipeline-1/
tags:
  - infra
---

# Practice - DevSecOps Pipeline with OWASP Juice Shop

### Objective

A simple/initial CI/CD pipeline for the OWASP Juice Shop application that automatically performs various security checks on every code push and pull request. This pipeline will serve as a practical example of integrating security into the development lifecycle (DevSecOps).

The pipeline will include the following stages:

*   **Secret Scanning**: To find any hardcoded secrets in the codebase.
*   **Static Application Security Testing (SAST)**: To analyze the source code for potential security vulnerabilities.
*   **Software Composition Analysis (SCA)**: To identify known vulnerabilities in the application's dependencies.
*   **Container Scanning**: To scan the application's Docker image for vulnerabilities.
*   **Dynamic Application Security Testing (DAST)**: To scan the running application for vulnerabilities at runtime.

### Step 1: Fork the Repository

Fork the official [OWASP Juice Shop repository](https://github.com/juice-shop/juice-shop) to your own GitHub account. This will give you a copy of the project where you can create your own GitHub Actions workflows.

### Step 2: Create the GitHub Actions Workflow File

In your forked repository, create a new file in the following directory: `.github/workflows/devsecops-pipeline.yml`. This YAML file will contain the complete configuration for your pipeline.

Copy and paste the following code into the `devsecops-pipeline.yml` file. Each job is explained in detail below.

```
name: DevSecOps Pipeline

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Fetches all history for a full scan

      - name: Secret scan with TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.before }}
          head: ${{ github.sha }}
          extra_args: --only-verified # Reduces false positives by verifying secrets
```

**`secret-scan`**:

*   **Tool**: TruffleHog.
*   **Purpose**: Scans the entire Git history for any secrets like API keys or passwords that may have been accidentally committed.
*   **Configuration**:
    *   `fetch-depth: 0` is used to check out the full git history.
    *   `--only-verified` reduces noise by only reporting secrets that TruffleHog can confirm are active.

```
  sast:
    name: SAST Analysis
    runs-on: ubuntu-latest
    needs: secret-scan # This job runs after the secret-scan job
    permissions:
      actions: read
      contents: read
      security-events: write # Required to upload results to GitHub's Security tab
    strategy:
      fail-fast: false
      matrix:
        language: [ 'javascript' ]

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{ matrix.language }}"
```

**`sast` (Static Application Security Testing)**:

*   **Tool**: GitHub CodeQL.
*   **Purpose**: Analyzes your source code to find security flaws like SQL injection, cross-site scripting (XSS), and other common vulnerabilities.
*   **Configuration**:
    *   It uses a `matrix` strategy to easily add more languages in the future.
    *   The necessary `permissions` are set to allow the action to upload findings to the "Security" > "Code scanning alerts" tab in your repository.

```
  container-scan:
    name: Container Scanning
    runs-on: ubuntu-latest
    needs: secret-scan # This job runs after the secret-scan job
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t juice-shop:latest .

      - name: Scan Local Image With Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'juice-shop:latest'
          format: 'table'
          exit-code: '0' # Set to '1' to fail the build on vulnerabilities
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
```

**`container-scan`**:

*   **Tool**: Aqua Security Trivy.
*   **Purpose**: Builds a Docker image of the Juice Shop application and then scans it for vulnerabilities within the base operating system layers and application libraries.
*   **Configuration**:
    *   `exit-code: '0'` ensures the pipeline continues even if vulnerabilities are found.
    *   `ignore-unfixed: true` filters out vulnerabilities that don't have a fix available yet.
    *   The scan focuses on `CRITICAL` and `HIGH` severity vulnerabilities.

```
  sca:
    name: SCA Analysis
    runs-on: ubuntu-latest
    needs: secret-scan # This job runs after the secret-scan job
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install dependencies
        run: npm install

      - name: OWASP Dependency-Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'juice-shop'
          scanPath: '.'
          format: 'HTML'

      - name: Upload Dependency-Check report
        uses: actions/upload-artifact@v4
        with:
          name: dependency-check-report
          path: reports/dependency-check-report.html

```

**`sca` (Software Composition Analysis)**:

*   **Tool**: OWASP Dependency-Check.
*   **Purpose**: Scans the project's third-party libraries (dependencies) for known vulnerabilities (CVEs). Juice Shop has many intentional vulnerabilities in its dependencies, so expect this to find many issues.
*   **Configuration**:
    *   `npm install` is run first to download all dependencies.
    *   The action generates an HTML report, which is then uploaded as a build artifact named `dependency-check-report`.

```yaml
  dast:
    name: DAST Scan
    runs-on: ubuntu-latest
    needs: secret-scan # This job runs after the secret-scan job
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Build and Run Juice Shop in a container
        run: |
          docker build -t juice-shop:latest .
          docker run -d -p 3000:3000 --name juice-shop-dast juice-shop:latest

      - name: Wait for application to start
        run: |
          echo "Waiting for application to start..."
          timeout 120s bash -c 'until curl -s http://localhost:3000 > /dev/null; do echo "Waiting for Juice Shop..."; sleep 5; done'
          echo "Application started!"

      - name: OWASP ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: 'http://localhost:3000'
          # The action automatically creates 'zap_report.html' in the workspace
          
      - name: Upload ZAP report
        if: always() # Ensures the report is uploaded even if the scan finds issues
        uses: actions/upload-artifact@v4
        with:
          name: zap-baseline-report
          path: zap_report.html
```

**`dast` (Dynamic Application Security Testing)**:

*   **Tool**: OWASP ZAP.
*   **Purpose**: Starts the application in a Docker container and then actively probes it from the outside (like a real attacker) to find runtime vulnerabilities.
*   **Configuration**:
    *   A robust wait mechanism is included to ensure the application is fully running before the scan begins.
    *   The ZAP baseline scan checks for a set of common web application vulnerabilities against the running instance at `http://localhost:3000`.
    *   The final HTML report is uploaded as a build artifact named `zap-baseline-report`.

### Step 3: Commit and Push the Workflow

Commit the new `devsecops-pipeline.yml` file to your repository's main branch.

```bash
git add .github/workflows/devsecops-pipeline.yml
git commit -m "Add DevSecOps pipeline"
git push
```

Pushing this file will automatically trigger the pipeline. You can view its progress under the "Actions" tab in your GitHub repository.

### Step 4: Viewing the Results

Once the pipeline run is complete, you can review the security findings:

*   **Logs**: The output of each job, including the list of vulnerabilities found by Trivy, can be viewed directly in the GitHub Actions logs.
*   **Artifacts**: The detailed HTML reports from OWASP Dependency-Check and OWASP ZAP can be downloaded from the "Summary" page of the completed workflow run.
*   **Security Tab**: SAST results from CodeQL will appear in your repository's **Security > Code scanning** section, providing detailed information about each finding and where it is in the code.