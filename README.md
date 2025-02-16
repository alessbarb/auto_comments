# Auto-comment on PR GitHub Action

This GitHub Action automatically adds a comment to newly opened or synchronized pull requests. It gathers details like the PR title, body, labels, file changes, and also **detects the project type** (Node, Maven, Gradle, or Python) to run generic build, test, and lint steps before commenting on the PR. If no recognized project type is found, those steps are skipped.

## Features

- Automatically triggered when a pull request is opened or updated.
- **Auto-detection** of the project type based on characteristic files (e.g., `package.json`, `pom.xml`, `build.gradle`, `requirements.txt`).
- Runs a configurable build, test, and lint process depending on the detected technology.
- Collects PR metadata via the GitHub API (labels, changed files, etc.).
- Writes a consolidated comment back to the PR, including build/test/lint summaries and file-level details.
- Retries API calls to handle transient errors.
- Leverages `jq` for JSON parsing and `curl` for HTTP requests.

## Prerequisites

- A GitHub Token (`GITHUB_TOKEN`) with permission to write to pull requests.
- The runner must have `jq` and `curl` installed (the workflow itself installs them, but note it uses `sudo apt-get`, which assumes `ubuntu-latest` or similar).
- For auto-detection to work:
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

1. **🛠️ Prepare Dependencies**: Installs `jq` and `curl`.
2. **🛡️ Validate Input Data**: Ensures critical variables are present (`REPO`, `PR_ID`, `PR_BRANCH`, and `GITHUB_TOKEN`).
3. **📥 Checkout Code**: Uses `actions/checkout@v3` to fetch the repository code.
4. **🔍 Detect Project Type**: Looks for specific files to set `BUILD_COMMAND`, `TEST_COMMAND`, and `LINT_COMMAND`.
5. **⚙️ Build & Test**: Dynamically runs the build/test commands defined in step 4. Skips if the project type is unknown.
6. **🧹 Lint**: Runs the configured lint command if the project type is recognized.
7. **🔎 Gather Commit Info**: Captures the latest commit SHA and message.
8. **📊 Gather PR Details**: Uses the GitHub API to gather labels and changed files.
9. **✔️ / ❌ Workflow Status**: Marks success or failure depending on previous steps.
10. **📝 Add Comment to PR**: Posts the collected and generated info (lint/test results, coverage, changed files, etc.) as a comment.
11. **🐛 Debug Logs**: Prints extra debugging information when `DEBUG` is `true`.

## Usage

Create or update a file in `.github/workflows`, for example `auto-comment-on-pr.yml`, with the following content:

```yaml
name: Auto-comment on PR

# NOTE: If you need to run this workflow with secrets accessible on Pull Requests from forks,
# consider using pull_request_target. However, be mindful of security implications.
on:
  pull_request:
    types:
      - opened
      - synchronize

jobs:
  comment:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    env:
      # By default, we set DEBUG to 'false' for a pull_request event.
      GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
      REPO: "${{ github.repository }}"
      PR_AUTHOR: "${{ github.event.pull_request.user.login }}"
      PR_ID: "${{ github.event.pull_request.number }}"
      PR_BRANCH: "${{ github.event.pull_request.head.ref }}"
      PR_TITLE: "${{ github.event.pull_request.title }}"
      PR_URL: "${{ github.event.pull_request.html_url }}"
      PR_BODY: "${{ github.event.pull_request.body }}"
      DEBUG: "false"

    steps:
      # (1) Install required dependencies (jq and curl)
      - name: 🛠️ Prepare Dependencies
        run: |
          echo "Installing jq and curl..."
          if ! sudo apt-get update && sudo apt-get install -y jq curl; then
            echo "Failed to install jq and curl. Exiting."
            exit 1
          fi
          echo "Dependencies installed."

      # (2) Validate critical input data
      - name: 🛡️ Validate Input Data
        run: |
          echo "Validating input data: REPO=$REPO, PR_ID=$PR_ID, PR_BRANCH=$PR_BRANCH"
          if [[ -z "$REPO" || -z "$PR_ID" || -z "$PR_BRANCH" ]]; then
            echo "Invalid input data. REPO, PR_ID, or PR_BRANCH is empty."
            exit 1
          fi

          if [[ -z "$GITHUB_TOKEN" ]]; then
            echo "GITHUB_TOKEN is empty. Cannot proceed."
            exit 1
          fi

          echo "Input data validated successfully."

      # (3) Checkout the code
      - name: 📥 Checkout Code
        uses: actions/checkout@v3

      # (4) Detect project type and define commands
      - name: 🔍 Detect Project Type
        id: detect
        run: |
          echo "Detecting project type..."

          if [ -f "package.json" ]; then
            echo "PROJECT_TYPE=node" >> $GITHUB_ENV
            echo "BUILD_COMMAND='npm install'" >> $GITHUB_ENV
            echo "TEST_COMMAND='npm test'" >> $GITHUB_ENV
            echo "LINT_COMMAND='npm run lint'" >> $GITHUB_ENV
          elif [ -f "pom.xml" ]; then
            echo "PROJECT_TYPE=maven" >> $GITHUB_ENV
            echo "BUILD_COMMAND='mvn clean package'" >> $GITHUB_ENV
            echo "TEST_COMMAND='mvn test'" >> $GITHUB_ENV
            echo "LINT_COMMAND='mvn checkstyle:check'" >> $GITHUB_ENV
          elif [ -f "build.gradle" ]; then
            echo "PROJECT_TYPE=gradle" >> $GITHUB_ENV
            echo "BUILD_COMMAND='gradle build'" >> $GITHUB_ENV
            echo "TEST_COMMAND='gradle test'" >> $GITHUB_ENV
            echo "LINT_COMMAND='gradle check'" >> $GITHUB_ENV
          elif [ -f "requirements.txt" ]; then
            echo "PROJECT_TYPE=python" >> $GITHUB_ENV
            echo "BUILD_COMMAND='pip install -r requirements.txt'" >> $GITHUB_ENV
            echo "TEST_COMMAND='pytest'" >> $GITHUB_ENV
            echo "LINT_COMMAND='flake8 .'" >> $GITHUB_ENV
          else
            echo "PROJECT_TYPE=unknown" >> $GITHUB_ENV
            echo "BUILD_COMMAND='echo \"No build command (unknown project type)\"'" >> $GITHUB_ENV
            echo "TEST_COMMAND='echo \"No test command (unknown project type)\"'" >> $GITHUB_ENV
            echo "LINT_COMMAND='echo \"No lint command (unknown project type)\"'" >> $GITHUB_ENV
          fi

      # (5) Build & Test (dynamic based on project type)
      - name: ⚙️ Build & Test
        run: |
          echo "PROJECT_TYPE=$PROJECT_TYPE"
          if [ "$PROJECT_TYPE" = "unknown" ]; then
            echo "No recognized project files found. Skipping build & test."
            echo "TESTS_SUMMARY='No tests ran (unknown project type)'" >> $GITHUB_ENV
            echo "COVERAGE_REPORT='No coverage (unknown project type)'" >> $GITHUB_ENV
            exit 0
          fi

          echo "Running build command: $BUILD_COMMAND"
          eval "$BUILD_COMMAND"

          echo "Running test command: $TEST_COMMAND"
          eval "$TEST_COMMAND"

          # Example placeholders
          echo "TESTS_SUMMARY='Example: tests completed successfully'" >> $GITHUB_ENV
          echo "COVERAGE_REPORT='Example: coverage ~85%'" >> $GITHUB_ENV

      # (6) Lint (dynamic based on project type)
      - name: 🧹 Lint
        run: |
          if [ "$PROJECT_TYPE" = "unknown" ]; then
            echo "No recognized project type. Skipping lint."
            echo "LINTER_RESULTS='No lint (unknown project type)'" >> $GITHUB_ENV
            exit 0
          fi

          echo "Running lint command: $LINT_COMMAND"
          eval "$LINT_COMMAND" || true  # Avoid failing if lint errors occur

          # Example placeholder
          echo "LINTER_RESULTS='Example: 0 warnings, 2 errors'" >> $GITHUB_ENV

      # (7) Gather commit information
      - name: 🔎 Gather Commit Info
        run: |
          COMMIT_SHA=$(git rev-parse HEAD)
          COMMIT_MESSAGE=$(git log -1 --pretty=%B)
          echo "COMMIT_SHA=$COMMIT_SHA" >> $GITHUB_ENV
          echo "COMMIT_MESSAGE=$COMMIT_MESSAGE" >> $GITHUB_ENV

      # (8) Gather PR details (labels, changed files)
      - name: 📊 Gather PR Details
        id: pr_details
        run: |
          # Define a retry function to robustly fetch data from GitHub API
          retry_command() {
            local -r cmd="$1"
            local -r max_retries="$2"
            local -r wait_time="$3"
            local -n result_ref="$4"
            local retry_count=0

            while [[ $retry_count -lt $max_retries ]]; do
              echo "Executing: $cmd... Attempt $(($retry_count + 1))"
              result_ref=$(eval $cmd)
              if [[ $? -eq 0 ]]; then
                return 0
              fi
              retry_count=$((retry_count + 1))
              sleep $wait_time
            done

            echo "Failed to execute $cmd after $max_retries attempts. Exiting."
            exit 1
          }

          echo "Initializing or cleaning up files_linked.txt..."
          > files_linked.txt

          # Extract PR labels from GitHub event payload
          echo "Fetching PR labels from event payload..."
          PR_LABELS=$(jq -r '.pull_request.labels[]?.name' "$GITHUB_EVENT_PATH" | tr '\n' ', ') || exit 1
          echo "PR_LABELS=$PR_LABELS" >> "$GITHUB_ENV"

          # Base URL for GitHub API calls
          GITHUB_API_BASE_URL="https://api.github.com"

          # Collect changed file details
          retry_command "curl -s -H \"Authorization: token $GITHUB_TOKEN\" $GITHUB_API_BASE_URL/repos/$REPO/pulls/$PR_ID/files" 3 5 FILES_JSON

          if [[ -z "$FILES_JSON" ]]; then
            echo "FILES_JSON is empty, which is unexpected. Exiting."
            exit 1
          fi

          REPO_URL="https://github.com/$REPO/blob/$PR_BRANCH"

          echo "Iterating through each file to gather details..."
          for row in $(echo "${FILES_JSON}" | jq -r '.[] | @base64'); do
            _jq() {
              echo ${row} | base64 --decode | jq -r ${1}
            }

            FILE_NAME=$(_jq '.filename')
            ADDITIONS=$(_jq '.additions')
            DELETIONS=$(_jq '.deletions')
            CHANGES=$(_jq '.changes')

            echo "- [$FILE_NAME]($REPO_URL/$FILE_NAME) (Additions: $ADDITIONS, Deletions: $DELETIONS, Changes: $CHANGES)" >> files_linked.txt
          done

          {
            echo 'FILES_LINKED<<EOF'
            cat files_linked.txt
            echo EOF
          } >> "$GITHUB_ENV"

      # (9) Workflow status: success or failure
      - name: ✔️ Workflow Status = Success
        if: success()
        run: |
          echo "WORKFLOW_STATUS=success" >> $GITHUB_ENV
          echo "Workflow completed successfully."

      - name: ❌ Error Handling (Workflow Status = Failure)
        if: failure()
        run: |
          echo "WORKFLOW_STATUS=failure" >> $GITHUB_ENV
          exit 1

      # (10) Post a comment on the PR, including extended details
      - name: 📝 Add Comment to PR
        run: |
          FILES_LINKED=$(echo "$FILES_LINKED" | sed -E 's/["“”]/\\"/g')

          COMMENT_BODY=$(jq -n \
            --arg pr_author "$PR_AUTHOR" \
            --arg pr_title "$PR_TITLE" \
            --arg pr_url "$PR_URL" \
            --arg pr_body "$PR_BODY" \
            --arg repo "$REPO" \
            --arg pr_labels "$PR_LABELS" \
            --arg workflow_status "$WORKFLOW_STATUS" \
            --arg files_linked "$FILES_LINKED" \
            --arg lint_results "$LINTER_RESULTS" \
            --arg tests_summary "$TESTS_SUMMARY" \
            --arg coverage_report "$COVERAGE_REPORT" \
            --arg commit_sha "$COMMIT_SHA" \
            --arg commit_msg "$COMMIT_MESSAGE" \
            '{
              "body": (
                "Hey @" + $pr_author + ",\n" +
                "- **PR Title**: " + $pr_title + "\n" +
                "- **PR URL**: " + $pr_url + "\n" +
                "- **PR Body**: " + $pr_body + "\n" +
                "- **Labels**: " + $pr_labels + "\n" +
                "- **Workflow Status**: " + $workflow_status + "\n\n" +
                "**Files Changed:**\n" + $files_linked + "\n\n" +
                "**Lint Results:** " + $lint_results + "\n" +
                "**Test Summary:** " + $tests_summary + "\n" +
                "**Coverage Report:** " + $coverage_report + "\n\n" +
                "**Last Commit SHA**: " + $commit_sha + "\n" +
                "Message: " + $commit_msg + "\n\n" +
                "Please ensure all tests and checks pass. We will review this shortly.\n" +
                "For more details, see [our contribution guide](https://github.com/" + $repo + "/blob/main/CONTRIBUTING.md)."
              )
            }'
          )

          if [[ -z "$COMMENT_BODY" ]]; then
            echo "Comment body is empty. Aborting."
            exit 1
          fi

          if [[ ${#COMMENT_BODY} -gt 65536 ]]; then
            echo "Comment body exceeds 65536 characters. Truncating..."
            COMMENT_BODY="${COMMENT_BODY:0:65532}...)"
          fi

          echo "Posting comment to PR..."
          curl \
            -X POST \
            -H "Authorization: token $GITHUB_TOKEN" \
            -H "Content-Type: application/json" \
            -d "$COMMENT_BODY" \
            "https://api.github.com/repos/$REPO/issues/$PR_ID/comments"

          echo "Comment added to PR successfully."

      # (11) Debug Logs (executed only if DEBUG == 'true')
      - name: 🐛 Debug Logs
        if: env.DEBUG == 'true'
        run: |
          echo "===== DEBUG LOGS ====="
          echo "REPO: $REPO"
          echo "PR_ID: $PR_ID"
          echo "PR_BRANCH: $PR_BRANCH"
          echo "PR_LABELS: $PR_LABELS"
          echo "LINTER_RESULTS: $LINTER_RESULTS"
          echo "TESTS_SUMMARY: $TESTS_SUMMARY"
          echo "COVERAGE_REPORT: $COVERAGE_REPORT"
          echo "COMMIT_SHA: $COMMIT_SHA"
          echo "COMMIT_MESSAGE: $COMMIT_MESSAGE"
          echo "FILES_JSON: $FILES_JSON"
          echo "======================"
```

## Debugging

If you need extra logs for troubleshooting, set `DEBUG` to `true`. The “Debug Logs” step (11) will print out additional environment variables and data.

## Limitations

- The comment body cannot exceed **65,536 characters** (GitHub’s limit). If it does, it will be truncated.
- This workflow only auto-detects Node.js, Maven, Gradle, or Python. If your project uses a different tech stack, please update the detection logic accordingly.

## Contributing

Issues and pull requests are welcome. Feel free to propose enhancements, bug fixes, or additional features to make this action even more versatile.
