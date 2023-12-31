# Auto-comment on PR GitHub Action

This GitHub Action automatically adds a comment to newly opened or synchronized pull requests. The comment will contain details about the PR, including the title, body, labels, and a list of files changed.

## Features

- Automatically triggered when a PR is opened or synchronized.
- Collects data about the PR using GitHub's API.
- Adds a comment to the PR with all the collected information.
- Implements a retry mechanism for API calls.
- Uses `jq` for JSON parsing and `curl` for API requests.

## Prerequisites

- GitHub Token with permissions to write to pull requests.
- `jq` and `curl` installed on the runner machine.

## Workflow Configuration

### Environment Variables

- `GITHUB_TOKEN`: GitHub token for API authentication.
- `REPO`: GitHub repository where the action is running.
- `PR_AUTHOR`: Author of the pull request.
- `PR_ID`: Pull request ID.
- `PR_BRANCH`: Branch from which the pull request was made.
- `PR_TITLE`: Title of the pull request.
- `PR_BODY`: Body content of the pull request.
- `DEBUG`: Debug mode toggle (default is `false`).

### Steps

1. **🛠️ Prepare Dependencies**: Installs `jq` and `curl`.
2. **🛡️ Validate Input Data**: Checks that all required environment variables are set.
3. **📥 Checkout Code**: Checks out the repository code.
4. **📊 Gather PR Details**: Collects details about the PR, such as labels and files changed.
5. **✔️ Workflow Status = Success**: Sets the workflow status to `success` if all steps are successful.
6. **❌ Error Handling**: Sets the workflow status to `failure` and logs errors if any step fails.
7. **📝 Add Comment to PR**: Adds the generated comment to the PR.
8. **🐛 Debug Logs**: Optional debugging logs.

## Usage

Add this YAML configuration to your `.github/workflows` directory as a new YAML file (e.g., `auto-comment-on-pr.yml`).

Here's the workflow configuration:

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
      PR_TITLE: "${{ github.event.pull_request.html_url }}"
      PR_BODY: "${{ github.event.pull_request.body }}"
      DEBUG: "${{ github.event.inputs.debug || 'false' }}"  

    steps:
    # Preparation Phase: Installing dependencies needed for the workflow
    - name: 🛠️ Prepare Dependencies
      run: |
        echo "Installing jq and curl..."
        if ! sudo apt-get install -y jq curl; then
          echo "Failed to install jq and curl. Exiting."
          exit 1
        fi
        echo "Dependencies installed."

    # Input Validation: Check if all required variables are present
    - name: 🛡️ Validate Input Data
      run: |
        echo "Validating input data: REPO=$REPO, PR_ID=$PR_ID, PR_BRANCH=$PR_BRANCH"
        if [[ -z "$REPO" || -z "$PR_ID" || -z "$PR_BRANCH" ]]; then
          echo "Invalid input data."
          exit 1
        fi
        echo "Input data validated successfully."

    # Checkout Code
    - name: 📥 Checkout Code
      uses: actions/checkout@v2

    # Data Gathering Phase: Collect PR details such as labels and files changed
    - name: 📊 Gather PR Details
      id: pr_details
      run: |
        # Define Retry Function
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

        # Initialize or clean up files_linked.txt
        echo "Initializing or cleaning up files_linked.txt..."
        > files_linked.txt

        # Fetch PR labels
        echo "Fetching PR labels..."
        PR_LABELS=$(jq -r '.pull_request.labels[]?.name' "$GITHUB_EVENT_PATH" | tr '\n' ', ') || exit 1
        if [[ $? -ne 0 ]]; then  
          echo "Failed to fetch PR labels using jq. Exiting."
          exit 1
        fi
        echo "PR_LABELS=$PR_LABELS" >> "$GITHUB_ENV"

        # Define GitHub API Base URL
        GITHUB_API_BASE_URL="https://api.github.com"

        # Fetch Files Changed details
        retry_command "curl -s -H \"Authorization: token $GITHUB_TOKEN\" $GITHUB_API_BASE_URL/repos/$REPO/pulls/$PR_ID/files" 3 5 FILES_JSON

        if [[ -z "$FILES_JSON" ]]; then
          echo "FILES_JSON is empty, which is unexpected. Exiting."
          exit 1
        fi

        FILES_LINKED=""
        REPO_URL="https://github.com/$REPO/blob/$PR_BRANCH"
        
        # Iterate through each file to get details
        echo "Iterating through each file to get details..."
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

        echo "PR details gathered successfully."

    # Workflow Status Phase
    - name: ✔️ Workflow Status = Success
      if: success()
      run: |
        echo "WORKFLOW_STATUS=success" >> $GITHUB_ENV
        echo "Workflow completed successfully."

    - name: ❌ Error Handling (Workflow Status = Failure)
      if: failure()
      run: |
        echo "WORKFLOW_STATUS=failure" >> $GITHUB_ENV
        echo "An error occurred. Check the logs for specific failure points."
        exit 1

    # Finalization Phase
    - name: 📝 Add Comment to PR
      run: |
        FILES_LINKED=$(echo "$FILES_LINKED" | sed -E 's/["“”]/\\"/g')

        # Preparing comment body for posting
        echo "Preparing comment body..."
        COMMENT_BODY=$(jq -n \
                        --arg pr_author "$PR_AUTHOR" \
                        --arg pr_title "$PR_TITLE" \
                        --arg pr_body "$PR_BODY" \
                        --arg repo "$REPO" \
                        --arg pr_labels "$PR_LABELS" \
                        --arg workflow_status "$WORKFLOW_STATUS" \
                        --arg files_linked "$FILES_LINKED" \
                        '{
                          "body": ("Hey @" + $pr_author + ",\n" +
                                  "- **PR Title**: " + $pr_title + "\n" +
                                  "- **PR Body**: " + $pr_body + "\n" +
                                  "- **Labels**: " + $pr_labels + "\n" +
                                  "- **Workflow Status**: " + $workflow_status + "\n\n" +
                                  "**Files Changed:**\n" + $files_linked + "\n\n" +
                                  "Please ensure all tests and checks are passed. We will examine it shortly.\n" +
                                  "For more details, please check [our contribution guide](https://github.com/" + $repo + "/blob/main/CONTRIBUTING.md).")
                        }')

        # Validate comment body before posting
        echo "Validating comment body..."
        if [[ -z "$COMMENT_BODY" ]]; then 
          echo "Comment body is empty. Aborting."
          exit 1
        fi

        if [[ ${#COMMENT_BODY} -gt 65536 ]]; then
          echo "Comment body exceeds 65536 characters. Truncating..."
          COMMENT_BODY="${COMMENT_BODY:0:65532}...)"
        fi

        # Posting comment to PR
        echo "Posting comment to PR..."
        curl \
          -X POST \
          -H "Authorization: token $GITHUB_TOKEN" \
          -H "Content-Type: application/json" \
          -d "$COMMENT_BODY" \
          "https://api.github.com/repos/$REPO/issues/$PR_ID/comments"

        echo "Comment added to PR successfully."

    - name: 🐛 Debug Logs
      if: env.DEBUG == 'true'
      run: |
        echo "===== DEBUG LOGS ====="
        echo "REPO: $REPO"
        echo "PR_ID: $PR_ID"
        echo "PR_BRANCH: $PR_BRANCH"
        echo "PR_LABELS: $PR_LABELS"
        echo "FILES_JSON: $FILES_JSON"
        echo "======================"
```

## Debugging

Set the `DEBUG` environment variable to `true` for additional logs.

## Limitations

- Comment body size must not exceed 65536 characters.

## Contributing

Feel free to contribute to this GitHub Action by opening issues or submitting pull requests.

