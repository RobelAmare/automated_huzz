# Automated Single-File Releases with GitHub Actions (Cross-Repository) (LINUX) 💤 💤

This repository uses GitHub Actions to automatically create daily releases of a single Python file from a specified folder in a **source repository**, with releases created in a **destination repository**, ensuring no file is released more than once.


## How It Works

1.  **Daily Trigger:** The workflow is triggered daily at midnight UTC, as defined by the `cron: '0 0 * * *'` setting.
2.  **Checkout Source Repository:** The workflow checks out the code from the **source repository** using the `SOURCE_REPO_TOKEN`.  The code is checked out into a directory named `source_repo`.
3.  **Version Increment:** It retrieves the latest version tag from the **destination repository**, increments it by 0.1, or defaults to version 1.0 if no tags exist.
4.  **File Listing:** It lists all Python files in the specified folder within the checked-out **source repository** code.
5.  **Released File Retrieval:** It retrieves a list of files that have already been released by parsing the release descriptions in the **destination repository**.
6.  **File Selection:** It iterates through the list of files and selects the first file that hasn't been released yet.
7.  **Conditional Release:** If a new file is found, it creates a new release in the **destination repository** with the incremented version number, including the selected file's name in the release description. If no new files are found, the workflow exits without creating a release.



KEY🔐:
source-repo: the repo with the folder that has the code in it
destination-repo: repo to be commited to at the end


---

## Prerequisites

*   A **source repository** (e.g., `source-owner/source-repo`) containing the Python files (`.py`) in a specific folder.
*   A **destination repository** (e.g., `destination-owner/destination-repo`) where the releases will be created. **This is where you'll set up the workflow**.
*   **Current Date and Time (UTC):** 2025-03-28 xx:yy:zz
*   **Your GitHub Username:** RobelAmare (example, use your own)

---

## Setup Instructions


1.  **Create a Workflow File in the Destination Repository:**
    *   In the **destination repository**, go to ACTIONS  >  and click SKIP THIS AND SET UP A WORKFLOW YOURSELF
    *   Create a new file named `release.yml` inside the `.github/workflows` directory.
  

2.  **Copy and Paste the Workflow Code: ⏬ **
    *   Open the `release.yml` file in a text editor.
    *   Copy the following code into the `release.yml` file. **Important:**
    *   Source Repo: Change `"source-owner/source-repo"` to the GitHub `username/repository` name where your Python files are stored `(e.g., "my-org/my-python-repo")`.
    *   Folder Path: Change `"path/to/your/python/folder"` to the exact folder path inside that source repository where your Python `.py` files are located `(e.g., "scripts" or "src/utilities")`. If the files are right at the root of the repository, use `""` (an empty string).


    ```yaml
    name: Create Daily Single-File Release (Cross-Repository)

    on:
      schedule:
        - cron: '0 0 * * *'  # Runs daily at midnight UTC

    jobs:
      release:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout code (source repo)
            uses: actions/checkout@v3
            with:
              repository: "source-owner/source-repo"
              path: source_repo
              token: ${{ secrets.SOURCE_REPO_TOKEN }}

          - name: Get current version
            id: get_version
            run: |
              VERSION=$(git tag --sort=committerdate | tail -n 1 | sed 's/^v//')
              if [[ -z "$VERSION" ]]; then
                VERSION="1.0"
              fi
              echo "VERSION=$VERSION" >> $GITHUB_OUTPUT

          - name: Increment version
            id: increment_version
            run: |
              VERSION=${{ steps.get_version.outputs.VERSION }}
              NEW_VERSION=$(echo "$VERSION + 0.1" | bc)
              echo "NEW_VERSION=$NEW_VERSION" >> $GITHUB_OUTPUT

          - name: Get list of files
            id: get_files
            run: |
              FILES=$(ls source_repo/"path/to/your/python/folder"/*.py)
              echo "FILES=$FILES" >> $GITHUB_OUTPUT

          - name: Get released files
            id: get_released
            run: |
              RELEASED_FILES=$(git tag --sort=committerdate | xargs -I {} git show -s --format="%b" {} | grep "Released file:" | awk '{print $3}')
              echo "RELEASED_FILES=$RELEASED_FILES" >> $GITHUB_OUTPUT

          - name: Select file to release
            id: select_file
            run: |
              FILES=(${ { steps.get_files.outputs.FILES }})
              RELEASED_FILES=(${ { steps.get_released.outputs.RELEASED_FILES }})
              for file in "${FILES[@]}"; do
                released=false
                for released_file in "${RELEASED_FILES[@]}"; do
                  if [[ "$file" == "$released_file" ]]; then
                    released=true
                    break
                  fi
                done
                if [ "$released" == "false" ]; then
                  SELECTED_FILE=$file
                  break
                fi
              done
              if [ -z "$SELECTED_FILE" ]; then
                echo "No new files to release."
                exit 0
              fi
              echo "SELECTED_FILE=$SELECTED_FILE" >> $GITHUB_OUTPUT

          - name: Create release
            if: steps.select_file.outputs.SELECTED_FILE != ''
            uses: softprops/action-gh-release@v1
            with:
              tag_name: v${{ steps.increment_version.outputs.NEW_VERSION }}
              name: Release v${{ steps.increment_version.outputs.NEW_VERSION }}
              body: |
                Automated daily release.
                Released file: ${ { steps.select_file.outputs.SELECTED_FILE }}
              draft: false
              prerelease: false
            env:
              GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    ```

3.  **Configure the Source Repository Token:**
    *   In the **destination repository**, go to "Settings" -> "Secrets and variables" -> "Actions".
    *   Click "New repository secret".
    *   Name the secret `SOURCE_REPO_TOKEN`.
    *   Create a personal access token (PAT) with `repo` scope in the **source repository**.
    *   Paste the PAT into the "Value" field for the `SOURCE_REPO_TOKEN` secret.

4.  **Adjust the Workflow for Your Folder:**
    *   In the `release.yml` file, locate the following line:
        ```yaml
        FILES=$(ls source_repo/"path/to/your/python/folder"/*.py)
        ```
    *   Replace `"path/to/your/python/folder"` with the actual path to the folder containing your Python files in the **source repository**. For example, if your Python files are in a folder named `scripts` in the source repository, the line should be:
        ```yaml
        FILES=$(ls source_repo/"scripts/*.py")
        ```

6.  **Commit the Workflow File:**
    *   Commit the `release.yml` file to the **destination repository**.

7.  **GitHub Actions Token:**
    *   The workflow uses the default `GITHUB_TOKEN` secret for authentication in the **destination repository**. You don't need to create this secret manually. GitHub automatically provides it for each workflow run.

---




# Important Considerations

*   This workflow relies on parsing the release descriptions in the **destination repository** to determine which files have already been released. If you manually modify the release descriptions, the workflow may not function correctly.
*   The workflow assumes that the Python files are directly within the specified folder in the **source repository**. It does not support nested subfolders.
*   The personal access token (PAT) used for the `SOURCE_REPO_TOKEN` must have the `repo` scope to allow the workflow to access the source repository.


---

## Example

After setting up the workflow, you should see new releases created daily in the "Releases" section of the **destination repository**. The releases will be named `Release v1.0`, `Release v1.1`, `Release v1.2`, and so on. Each release will include the name of the Python file that was released from the **source repository**.


---
## Author

[RobelAmare](https://github.com/RobelAmare)
