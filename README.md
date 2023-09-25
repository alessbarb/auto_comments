# Auto-comment on PR GitHub Action

## Overview

This GitHub Action automatically comments on new pull requests with essential details such as the PR number, labels, workflow status, and files changed. It is designed to help maintainers and contributors get a quick overview of the changes introduced in the pull request.

## Features

- Automatically triggers when a new pull request is opened.
- Provides PR metadata including the PR number and author.
- Lists all the files that have been changed in the PR, along with additions, deletions, and total changes for each file.
- Indicates the workflow status (e.g., success or failure).

## Prerequisites

- GitHub Token: Make sure you have set up a GitHub Token in your secrets to interact with the GitHub API.

## Usage

Include this workflow in your `.github/workflows/` directory with a filename like `auto-comments.yml`.

Here's the workflow configuration:

```yaml
name: Auto-comment on PR
on:
  pull_request:
    types: [opened]

jobs:
  comment:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Cache Node.js modules
      uses: actions/cache@v2
      with:
        path: ~/.npm
        key: ${{ runner.OS }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.OS }}-node-

    - name: Gather PR details
      id: pr_details
      run: |
        set -e  # Stop execution on any error
        
        PR_ID=$(jq -r ".pull_request.number" "$GITHUB_EVENT_PATH") || exit 1
        PR_AUTHOR=$(jq -r ".pull_request.user.login" "$GITHUB_EVENT_PATH") || exit 1
        PR_BRANCH=$(jq -r ".pull_request.head.ref" "$GITHUB_EVENT_PATH") || exit 1
    
        REPO_URL="https://github.com/${{ github.repository }}/blob/$PR_BRANCH"
        
        # Fetch Files Changed details
        FILES_JSON=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
          https://api.github.com/repos/${{ github.repository }}/pulls/$PR_ID/files) || exit 1
    
        FILES_LINKED=""
        
        # Iterate through each file to get details
        for row in $(echo "${FILES_JSON}" | jq -r '.[] | @base64'); do
          _jq() {
            echo ${row} | base64 --decode | jq -r ${1}
          }
          
          FILE_NAME=$(_jq '.filename')
          ADDITIONS=$(_jq '.additions')
          DELETIONS=$(_jq '.deletions')
          CHANGES=$(_jq '.changes')
          
          FILES_LINKED+="- [$FILE_NAME]($REPO_URL/$FILE_NAME) (Additions: $ADDITIONS, Deletions: $DELETIONS, Changes: $CHANGES)"$'\n'
        done
        
        echo "PR_ID=$PR_ID" >> $GITHUB_ENV
        echo "PR_AUTHOR=$PR_AUTHOR" >> $GITHUB_ENV
        echo "FILES_LINKED=$FILES_LINKED" >> $GITHUB_ENV

    - name: Set Workflow Status
      run: echo "WORKFLOW_STATUS=success" >> $GITHUB_ENV
      if: success()

    - name: Error Handling
      if: failure()
      run: |
        echo "WORKFLOW_STATUS=failure" >> $GITHUB_ENV
        echo "An error occurred. Check the logs for details."

    - name: Add comment to PR
      uses: thollander/actions-comment-pull-request@v2
      with:
        message: |
          Hey @${{ env.PR_AUTHOR }},
          - **PR Number**: [#${{ env.PR_ID }}](https://github.com/${{ github.repository }}/pull/${{ env.PR_ID }})
          - **Labels**: ${{ env.LABELS }}
          - **Workflow Status**: ${{ env.WORKFLOW_STATUS }}
          
          **Files Changed:**
          ${{ env.FILES_LINKED }}
          
          Please ensure all tests and checks are passed. We'll be reviewing it shortly.
          For more details, please check [our contribution guide](https://github.com/${{ github.repository }}/blob/main/CONTRIBUTING.md).
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

None.

## Outputs

None.

## Environment Variables

- `PR_ID`: The ID of the pull request.
- `PR_AUTHOR`: The author of the pull request.
- `FILES_LINKED`: A markdown-formatted list of files changed with additional details.
- `WORKFLOW_STATUS`: The status of the workflow, either `success` or `failure`.

## Contributing

Feel free to open issues or PRs if you find any problems or have suggestions for improvements.
