
# **GitHub Actions for SAST: Gitleaks & Semgrep**

## **Overview**
This documentation provides a setup for **SAST** (Static Application Security Testing) using two GitHub Actions workflows:
1. **Gitleaks**: Scans the repository for hardcoded secrets.
2. **Semgrep**: Performs static analysis on the source code to detect security vulnerabilities.

Both workflows are set up to trigger on different events and can be configured to run on-demand, on pull requests, or on a scheduled basis.

## **1. GitHub Action: Gitleaks**

### **Purpose**
This workflow uses the [Gitleaks GitHub Action](https://github.com/gitleaks/gitleaks-action) to scan the repository for hardcoded secrets such as passwords, API keys, and other sensitive data.

### **Workflow File: `sast-gitleaks.yml`**

```yaml
name: gitleaks
on: [push, workflow_dispatch]
jobs:
  scan:
    name: gitleaks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

### **How to Use**
1. **Create the Workflow**:
   - Create a file in your repository: `.github/workflows/sast-gitleaks.yml`.
   - Paste the content above into this file.

2. **Secrets Configuration**:
   - Set up the following GitHub secrets in your repository:
     - `GITHUB_TOKEN`: This is automatically available in GitHub Actions, but ensure your repository permissions are set correctly.
     - `GITLEAKS_LICENSE`: If you're using a paid version of Gitleaks, you'll need to configure your license key here.

3. **Events**:
   - This workflow will run on:
     - `push` to any branch.
     - `workflow_dispatch` to manually trigger the workflow from the GitHub Actions interface.

4. **Scan Output**:
   - Gitleaks will output the findings as a report in the GitHub Actions logs. You can further modify the workflow to save the results to a file if needed.

---

## **2. GitHub Action: Semgrep**

### **Purpose**
This workflow uses [Semgrep](https://github.com/semgrep/semgrep) for static code analysis to detect security vulnerabilities in the code. It supports a wide range of languages and security patterns.

### **Workflow File: `sast-semgrep.yml`**

```yaml
# Name of this GitHub Actions workflow.
name: Semgrep

on:
  # Scan changed files in PRs (diff-aware scanning):
  pull_request: {}
  # Scan on-demand through GitHub Actions interface:
  workflow_dispatch: {}
  # Scan mainline branches if there are changes to .github/workflows/semgrep.yml:
  push:
    branches:
      - main
      - master
    paths:
      - .github/workflows/semgrep.yml
  # Schedule the CI job (this method uses cron syntax):
  schedule:
    - cron: '20 17 * * *' # Sets Semgrep to scan every day at 17:20 UTC.
    # It is recommended to change the schedule to a random time.

jobs:
  semgrep:
    name: semgrep/ci
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        run: |
          # Copy the event.json file into the current working directory 
          # so it gets mounted when we run docker and semgrep can refer 
          # to it when setting git metadata.
          cp $GITHUB_EVENT_PATH ./.github_event_path.json
          # Copy over all GitHub Actions related env vars from the runner
          # environment into a file that will get passed into docker so 
          # we retain all necessary env vars that semgrep uses for 
          # setting git metadata.
          echo "Saving these env vars to file, then passing into docker container."
          printenv | sort | egrep "^GITHUB_" | tee .env
          printenv | sort | egrep "^RUNNER_" | tee -a .env
          # Recursively set the owner:group of all files in the current
          # working directory to the UID and GID of the semgrep user
          # that the non-root semgrep docker image runs as. When we bind
          # mount the directory to our docker container, it retains the
          # permissions of the host.
          sudo chown -R 1000:1000 . 
          # Finally, we pass in all the env vars from our file along with
          # setting the SEMGREP_APP_TOKEN and updated GITHUB_EVENT_PATH.
          docker run --rm -v "${PWD}:/src" \
            --env-file .env \
            -e SEMGREP_APP_TOKEN=${{ secrets.SEMGREP_APP_TOKEN }} \
            -e GITHUB_EVENT_PATH="/src/.github_event_path.json" \
            semgrep/semgrep:latest-nonroot \
            semgrep ci
```

### **How to Use**
1. **Create the Workflow**:
   - Create a file in your repository: `.github/workflows/sast-semgrep.yml`.
   - Paste the content above into this file.

2. **Secrets Configuration**:
   - Set up the following GitHub secrets in your repository:
     - `SEMGREP_APP_TOKEN`: This is required to authenticate with Semgrep's API. You can get your token from [Semgrep's website](https://semgrep.dev/docs/get-started/#get-an-api-token).

3. **Events**:
   - This workflow will run on:
     - `pull_request` to scan changes in PRs.
     - `workflow_dispatch` to manually trigger the workflow.
     - `push` when changes are made to `.github/workflows/semgrep.yml` on the `main` or `master` branches.
     - A scheduled scan that runs daily at `17:20 UTC` (customizable).

4. **Scan Output**:
   - The output of the scan will be visible in the GitHub Actions logs. It will analyze the code using Semgrep rules, detecting security vulnerabilities, and issues in the code.

---

## **Steps to Make Both Actions Work**

### **1. Add the Workflow Files**:
   - Add the `sast-gitleaks.yml` and `sast-semgrep.yml` files to the `.github/workflows/` directory in your GitHub repository.

### **2. Configure Secrets**:
   - Navigate to the **Settings** page of your GitHub repository.
   - Under **Secrets and variables**, choose **Actions** and add the required secrets:
     - `GITHUB_TOKEN`: This is automatically available, but ensure you have the correct permissions.
     - `GITLEAKS_LICENSE`: If you are using a Gitleaks paid license.
     - `SEMGREP_APP_TOKEN`: Obtain this token from the [Semgrep website](https://semgrep.dev/).
   
### **3. Triggering the Actions**:
   - You can trigger the workflows manually from the **Actions** tab in GitHub or configure them to run on push, pull requests, or according to the schedule.

### **4. Monitor Results**:
   - After the scans are completed, you can view the results in the GitHub Actions logs. You may also choose to send the findings to a file or external service for further action.

---

This setup ensures that your codebase is regularly scanned for security vulnerabilities both in the code itself (using **Gitleaks** and **Semgrep**) and in dependencies (using the earlier Docker Compose setup for SCA).
