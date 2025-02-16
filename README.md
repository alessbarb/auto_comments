# Auto-comment on PR GitHub Action (v1.2.0)

This GitHub Action automatically posts a comment on newly opened or updated pull requests. It gathers essential PR information such as the title, description, labels, and modified files, while also **automatically detecting the project type** (supporting Node—with Yarn or npm—Maven, Gradle, or Python) to dynamically execute build, test, and lint steps in a robust and efficient manner.

> **Future Vision:** With this version, we refine continuous integration by providing clearer environment detection, more efficient dependency management, and detailed developer feedback—preparing us to seamlessly adapt to emerging languages and tools as the ecosystem evolves.

---

## Features

- **Automatic Trigger:** Executes when a pull request is opened or updated.
- **Intelligent Project Detection:** Uses a modular function to identify the project type based on key files (such as `yarn.lock`, `package.json`, `pom.xml`, `build.gradle`, or `requirements.txt`).
- **Conditional Node.js Setup:** For Node-based projects, the environment is automatically configured with Node.js version 18 (upgraded from version 16).
- **Efficient Dependency Management:**
  - Implements caching with `actions/cache@v4` based on the detected dependency file and project type.
  - Streamlines cache key preparation using environment variables.
- **Enhanced Retry Logic:** Incorporates an optimized retry function for build and test commands, allowing up to 3 attempts with a 5-second delay between retries.
- **Dynamic Build & Test Execution:** 
  - Executes build and test commands tailored to the project type.
  - For Node projects, it analyzes a coverage file to provide precise metrics; for other project types, it notifies that coverage data is not available.
- **Code Quality Analysis:** Runs the configured lint command and halts the pipeline if errors are detected.
- **Metadata and PR Details Collection:** 
  - Retrieves the latest commit SHA and message.
  - Extracts PR labels and lists changed files with clickable links to the repository.
- **Refined Comment Format:** 
  - Posts a consolidated comment in the PR with the header “## 🤖 Automated PR Analysis”.
  - Summarizes build, test, lint results, coverage information, and file changes.
- **Debug and Diagnostics:** An optional debug mode (enabled by setting `DEBUG` to `true`) prints detailed environment and filesystem information to facilitate troubleshooting.

---

## Prerequisites

- **GitHub Token:** A valid `GITHUB_TOKEN` with permission to post comments on pull requests.
- **Operating System:** The action is designed for Ubuntu-based runners (e.g., `ubuntu-latest`), as it utilizes `sudo apt-get` to install dependencies like `jq` and `curl`.
- **Project Files for Detection:**
  - **Yarn:** Presence of `yarn.lock`.
  - **Node (npm):** Presence of `package.json`.
  - **Maven:** Presence of `pom.xml`.
  - **Gradle:** Presence of `build.gradle`.
  - **Python:** Presence of `requirements.txt`.

  If none of these files are detected, the action will classify the project as `unknown` and skip the build, test, and lint steps.

---

## Workflow Configuration

The workflow is composed of the following phases:

1. **🛠️ Environment Setup:**
   - Updates package repositories and installs `jq` and `curl`.
   - Validates critical environment variables (`REPO`, `PR_ID`, `PR_BRANCH`, and `GITHUB_TOKEN`).

2. **📥 Code Checkout:**  
   - Uses `actions/checkout@v4` to fetch the repository’s code.

3. **🛡️ Define Retry Function:**  
   - Creates a script that implements a function to retry commands (up to 3 attempts, with a 5-second pause between tries).

4. **🔍 Project Type Detection:**
   - Determines the project type by searching for specific files.
   - Configures the corresponding build, test, and lint commands, and sets the dependency file accordingly.

5. **⚙️ Node.js Environment Setup (Conditional):**
   - For Node-based projects, configures the environment with Node.js version 18.

6. **🔑 Dependency Management and Caching:**
   - Assigns the appropriate cache path based on the project type.
   - Caches the dependency directory using a key generated from the dependency file and project type.

7. **🛠️ Build & Test Execution:**
   - Executes the build and test commands with the retry logic.
   - Collects coverage data for Node projects; for others, a default message is assigned.

8. **🧹 Code Linting:**
   - Runs the configured lint command and updates the lint status based on the outcome.

9. **📊 Metadata Collection:**
   - Retrieves the latest commit SHA and commit message.

10. **📌 PR Details Collection:**
    - Extracts PR labels and the list of modified files, generating direct links for each file.

11. **💬 Posting the PR Comment:**
    - Constructs a detailed comment (with the header “## 🤖 Automated PR Analysis”) that summarizes all the gathered information.
    - Ensures the comment does not exceed GitHub’s 65,536-character limit by truncating if necessary.

12. **🐛 Debug Information (Optional):**
    - If the `DEBUG` variable is set to `true`, prints additional environment and filesystem details to assist with troubleshooting.

---

## Usage

1. **Workflow Location:**  
   - Create (or use) the `.github/workflows` folder at the root of your repository.
   
2. **Implementing the Workflow File:**  
   - Copy the contents of the workflow file (e.g., `auto_comments.yml`) into the `.github/workflows` directory.

Once implemented, the workflow will automatically execute on every pull request (when opened or updated). The action detects your project type, manages dependencies, runs build and test steps, and posts a comprehensive comment on the PR.

---

## Debugging

- To obtain detailed logs for troubleshooting, set the `DEBUG` environment variable to `true`.
- The **🐛 Debug Info** step will display detailed environment variables and filesystem information.

---

## Limitations and Considerations

- The comment body is limited to **65,536 characters**; if exceeded, it will be truncated.
- The action natively detects projects using only Yarn, Node.js, Maven, Gradle, or Python. For other environments, extend the detection logic as needed.
- The current configuration stops execution if lint errors are detected. If you prefer warnings instead of halting the workflow, adjust the lint configuration accordingly.

---

## Contributing

Contributions are welcome. Please feel free to report issues, suggest enhancements, or propose new features. Your input is invaluable as we continue to evolve this action to meet future needs.

---
