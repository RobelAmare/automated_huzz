## Customization

*   **Release Frequency:** To change the release frequency, modify the `cron` expression in the `release.yml` file.
*   **Release Naming:** To change the release naming scheme, modify the `tag_name` and `name` properties in the `softprops/action-gh-release@v1` step.
*   **File Selection Order:** The current workflow releases files in the order they are listed by the `ls` command. You can modify the `Select file to release` step to implement a different file selection order (e.g., by creation date)
