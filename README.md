# Auto-comment on PR GitHub Action

This GitHub Action automatically adds a comment to newly opened or synchronized pull requests. It gathers details like the PR title, body, labels, file changes, and also **detects the project type** (Node, Maven, Gradle, Python, or Yarn-based Node) to run generic build, test, and lint steps before commenting on the PR. If no recognized project type is found, those steps are skipped.

## Features

- Automatically triggered when a pull request is opened or updated.
- **Auto-detection** of the project type based on characteristic files (e.g., `yarn.lock`, `package.json`, `pom.xml`, `build.gradle`, `requirements.txt`).
- Runs a configurable build, test, and lint process depending on the detected technology.
- **Dependency caching**: uses `actions/cache@v3` to speed up builds (e.g., caching `node_modules`, `~/.m2/repository`, `~/.gradle`, `~/.cache/pip`, etc.).
- **Retry logic** for build and test commands, to gracefully handle transient failures by re-attempting.
- **Optional coverage parsing**: for Node projects with Jest, it can parse `coverage/coverage-summary.json` to post real coverage info.
- Collects PR metadata via the GitHub API (labels, changed files, etc.).
- Writes a consolidated comment back to the PR, including build/test/lint summaries and file-level details.
- Retries API calls to handle transient errors.
- Leverages `jq` for JSON parsing and `curl` for HTTP requests.

## Prerequisites

- A GitHub Token (`GITHUB_TOKEN`) with permission to write to pull requests.
- The runner must have `jq` and `curl` installed (the workflow itself installs them, but note it uses `sudo apt-get`, which assumes `ubuntu-latest` or similar).
- For auto-detection to work:
  - If the project uses **Yarn**, it should have a `yarn.lock`.
  - If the project uses **Node**, it should have a `package.json`.
  - If the project uses **Maven**, it should have a `pom.xml`.
  - If the project uses **Gradle**, it should have a `build.gradle`.
  - If the project uses **Python**, it should have a `requirements.txt`.
  - Otherwise, the workflow will classify the project type as `unknown` and skip build/test/lint.

## Workflow Configuration

### Environment Variables

- **`GITHUB_TOKEN`**: Used to authenticate GitHub API calls.
- **`REPO`**: The `owner/repo` string for the repository where the action runs.
- **`PR_AUTHOR`**: The GitHub username of the PR’s author.
- **`PR_ID`**: The pull request number.
- **`PR_BRANCH`**: The branch from which the pull request was created.
- **`PR_TITLE`**: The title of the pull request.
- **`PR_URL`**: The direct URL of the pull request.
- **`PR_BODY`**: The description/body text of the PR.
- **`DEBUG`**: Controls debug logs (default is `false`). Set to `true` to see extra logs.

### Steps Overview

1. **🛠️ Prepare Dependencies**: Installs `jq` and `curl` if not present.  
2. **🛡️ Validate Input Data**: Ensures critical variables are present (`REPO`, `PR_ID`, `PR_BRANCH`, and `GITHUB_TOKEN`).  
3. **📥 Checkout Code**: Uses `actions/checkout@v3` to fetch the repository code.  
4. **🔍 Detect Project Type**: Looks for specific files to set `BUILD_COMMAND`, `TEST_COMMAND`, `LINT_COMMAND`, and defines caching paths.  
5. **🔑 Prepare & Cache Dependencies**: Dynamically computes a cache key from a dependency file (e.g. `package-lock.json` or `pom.xml`), then caches the relevant directory.  
6. **🛡️ Define Retry Function**: Creates a small shell script to allow up to 3 retries for build/test commands if needed.  
7. **⚙️ Build & Test**: Dynamically runs the build/test commands. Skips if the project type is unknown. Uses the retry logic for transient failures.  
8. **🧹 Lint**: Runs the configured lint command if the project type is recognized, and fails the workflow on lint errors.  
9. **🔎 Gather Commit Info**: Captures the latest commit SHA and message.  
10. **📊 Gather PR Details**: Uses the GitHub API to gather labels and changed files, also with retry logic.  
11. **✔️ / ❌ Workflow Status**: Marks success or failure depending on previous steps.  
12. **📝 Add Comment to PR**: Posts the collected and generated info (lint/test results, coverage, changed files, etc.) as a comment, including optional coverage details for Node projects.  
13. **🐛 Debug Logs**: Prints extra debugging information when `DEBUG` is `true`.

## Usage

1. **Create the Workflow Directory:**  
   If it doesn't already exist, create a folder called `.github/workflows` at the root of your repository.

2. **Copy the Workflow File:**  
   Open the file [./.github/workflows/auto_comments.yml](./.github/workflows/auto_comments.yml) and copy its entire contents into a new file within the `.github/workflows` directory (for example, name it `auto_comments.yml`).

Once these steps are completed, the action will be automatically triggered on every pull request (when opened or updated). The workflow detects your project type (supporting Yarn, Node.js, Maven, Gradle, and Python), caches dependencies, applies retry logic to build and test commands, runs linting, and posts a detailed comment on the pull request with build/test results, lint output, coverage information, and a list of changed files with clickable links.

Simply customize the workflow if needed, and commit it to your repository.

## Debugging

If you need extra logs for troubleshooting, set `DEBUG` to `true`. The “Debug Logs” step at the end will print out additional environment variables and data.

## Limitations

- The comment body cannot exceed **65,536 characters** (GitHub’s limit). If it does, it will be truncated.
- This workflow only auto-detects Yarn, Node.js, Maven, Gradle, or Python. If your project uses a different tech stack, please update the detection logic accordingly.
- Lint is configured to fail the pipeline by default (using `set -e`). If you prefer warnings instead of failing the entire workflow, remove or adjust that behavior in the lint step.

## Contributing

Issues and pull requests are welcome. Feel free to propose enhancements, bug fixes, or additional features to make this action even more versatile.
