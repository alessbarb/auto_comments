# Auto-comment on PR GitHub Action

This GitHub Action automatically adds a comment to newly opened or synchronized pull requests. It pulls details like the PR title, body, labels, and file changes, then posts all that info as a comment in the PR.

## Features

- Automatically triggered when a pull request is opened or updated.
- Collects PR metadata using the GitHub API (labels, files changed, etc.).
- Writes a consolidated comment back to the PR, including file-level details.
- Retries API calls to handle transient errors.
- Leverages `jq` for JSON parsing and `curl` for HTTP requests.

## Prerequisites

- A GitHub Token (`GITHUB_TOKEN`) with permission to write to pull requests.
- The runner must have `jq` and `curl` installed (the workflow installs them if needed).

## Workflow Configuration

### Environment Variables

- **`GITHUB_TOKEN`**: The token used to authenticate GitHub API calls.
- **`REPO`**: The `owner/repo` string for the repository where the action runs.
- **`PR_AUTHOR`**: The GitHub username of the PR’s author.
- **`PR_ID`**: The pull request number.
- **`PR_BRANCH`**: The branch from which the pull request was created.
- **`PR_TITLE`**: The title of the pull request.
- **`PR_URL`**: The direct URL of the pull request.
- **`PR_BODY`**: The description/body text of the PR.
- **`DEBUG`**: Controls debug logs (default is `false`).

### Steps

1. **🛠️ Prepare Dependencies**: Installs `jq` and `curl` on the runner.
2. **🛡️ Validate Input Data**: Ensures that necessary variables (`REPO`, `PR_ID`, `PR_BRANCH`, and `GITHUB_TOKEN`) are present.
3. **📥 Checkout Code**: Uses `actions/checkout@v3` to fetch the repository contents.
4. **📊 Gather PR Details**: Pulls labels and changed files from the GitHub API, retrying on transient errors.
5. **✔️ Workflow Status = Success**: Sets a success status if everything completes without errors.
6. **❌ Error Handling**: Marks the workflow as failed if any step fails.
7. **📝 Add Comment to PR**: Builds a detailed comment with all collected PR info and posts it to the PR.
8. **🐛 Debug Logs**: Prints additional information if `DEBUG` is set to `true`.

## Usage

Create a new file under `.github/workflows`, for example `auto-comment-on-pr.yml`, and add the following:

```yaml
name: Auto-comment on PR
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
      GITHUB_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
      REPO: "${{ github.repository }}"
      PR_AUTHOR: "${{ github.event.pull_request.user.login }}"
      PR_ID: "${{ github.event.pull_request.number }}"
      PR_BRANCH: "${{ github.event.pull_request.head.ref }}"
      PR_TITLE: "${{ github.event.pull_request.title }}"
      PR_URL: "${{ github.event.pull_request.html_url }}"
      PR_BODY: "${{ github.event.pull_request.body }}"
      DEBUG: "${{ github.event.inputs.debug || 'false' }}"  

    steps:
    - name: 🛠️ Prepare Dependencies
      run: |
        echo "Installing jq and curl..."
        if ! sudo apt-get install -y jq curl; then
          echo "Failed to install jq and curl. Exiting."
          exit 1
        fi

    - name: 🛡️ Validate Input Data
      run: |
        echo "Validating input data: REPO=$REPO, PR_ID=$PR_ID, PR_BRANCH=$PR_BRANCH"
        if [[ -z "$REPO" || -z "$PR_ID" || -z "$PR_BRANCH" ]]; then
          echo "Invalid input data. REPO, PR_ID, or PR_BRANCH is empty."
          exit 1
        fi

        if [[ -z "$GITHUB_TOKEN" ]]; then
          echo "GITHUB_TOKEN is empty. Cannot continue."
          exit 1
        fi

    - name: 📥 Checkout Code
      uses: actions/checkout@v3

    - name: 📊 Gather PR Details
      id: pr_details
      run: |
        # Retry function to handle transient errors
        retry_command() {
          local -r cmd="$1"
          local -r max_retries="$2"
          local -r wait_time="$3"
          local -n result_ref="$4"
          local retry_count=0

          while [[ $retry_count -lt $max_retries ]]; do
            echo "Executing: $cmd... Attempt $((retry_count + 1))"
            result_ref=$(eval $cmd)
            if [[ $? -eq 0 ]]; then
              return 0
            fi
            retry_count=$((retry_count + 1))
            sleep $wait_time
          done

          echo "Failed to execute $cmd after $max_retries attempts."
          exit 1
        }

        # Prepare output file
        > files_linked.txt

        # Extract labels using jq
        PR_LABELS=$(jq -r '.pull_request.labels[]?.name' "$GITHUB_EVENT_PATH" | tr '\n' ', ') || exit 1
        echo "PR_LABELS=$PR_LABELS" >> "$GITHUB_ENV"

        # Fetch changed files
        GITHUB_API_BASE_URL="https://api.github.com"
        retry_command "curl -s -H \"Authorization: token $GITHUB_TOKEN\" $GITHUB_API_BASE_URL/repos/$REPO/pulls/$PR_ID/files" 3 5 FILES_JSON

        if [[ -z "$FILES_JSON" ]]; then
          echo "FILES_JSON is empty, exiting."
          exit 1
        fi

        REPO_URL="https://github.com/$REPO/blob/$PR_BRANCH"
        for row in $(echo "${FILES_JSON}" | jq -r '.[] | @base64'); do
          _jq() { echo ${row} | base64 --decode | jq -r ${1}; }
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

    - name: ✔️ Workflow Status = Success
      if: success()
      run: echo "WORKFLOW_STATUS=success" >> $GITHUB_ENV

    - name: ❌ Error Handling
      if: failure()
      run: |
        echo "WORKFLOW_STATUS=failure" >> $GITHUB_ENV
        exit 1

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
                        '{
                          "body": (
                            "Hey @" + $pr_author + ",\n" +
                            "- **PR Title**: " + $pr_title + "\n" +
                            "- **PR URL**: " + $pr_url + "\n" +
                            "- **PR Body**: " + $pr_body + "\n" +
                            "- **Labels**: " + $pr_labels + "\n" +
                            "- **Workflow Status**: " + $workflow_status + "\n\n" +
                            "**Files Changed:**\n" + $files_linked + "\n\n" +
                            "Please ensure all tests and checks are passed. We will examine it shortly.\n" +
                            "For more details, please check [our contribution guide](https://github.com/" + $repo + "/blob/main/CONTRIBUTING.md)."
                          )
                        }')

        # Validate comment body
        if [[ -z "$COMMENT_BODY" ]]; then
          echo "Comment body is empty. Exiting."
          exit 1
        fi

        # Avoid exceeding GitHub's character limit
        if [[ ${#COMMENT_BODY} -gt 65536 ]]; then
          COMMENT_BODY="${COMMENT_BODY:0:65532}...)"
        fi

        # Post comment
        curl -X POST \
             -H "Authorization: token $GITHUB_TOKEN" \
             -H "Content-Type: application/json" \
             -d "$COMMENT_BODY" \
             "https://api.github.com/repos/$REPO/issues/$PR_ID/comments"

    - name: 🐛 Debug Logs
      if: env.DEBUG == 'true'
      run: |
        echo "===== DEBUG LOGS ====="
        echo "REPO: $REPO"
        echo "PR_ID: $PR_ID"
        echo "PR_BRANCH: $PR_BRANCH"
        echo "PR_LABELS: $PR_LABELS"
        echo "FILES_JSON: $FILES_JSON"

```

## Debugging

Set `DEBUG` to `true` in the environment variables if you need extra logs for troubleshooting.

## Limitations

- The comment body cannot exceed 65,536 characters (GitHub’s limit).

## Contributing

Issues and pull requests are welcome. Feel free to propose enhancements, bug fixes, or additional features to make this action even more useful.
