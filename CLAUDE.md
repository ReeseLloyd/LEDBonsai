# LEDBonsai Project

## Project Location
`~/Library/CloudStorage/Dropbox/05-Projects/Programming/LEDBonsai/`

## File Naming Convention
Files follow the pattern `ledbonsai-vMAJOR_MINOR.html` and `ledbonsai-vMAJOR_MINOR.md`.
- The `.html` file is the working code
- The `.md` file contains notes/documentation for that version

The current latest version is determined by the highest version number present (e.g., `v1_20`).

## Working with This Project
- Always read the latest `.html` and `.md` files before making changes
- When creating a new version, increment the minor version number (e.g., `v1_20` → `v1_21`)
- Save the new version as a new file — do not overwrite previous versions
- Update the `.md` file alongside the `.html` file to document what changed

## Git & GitHub
- Remote: https://github.com/ReeseLloyd/LEDBonsai.git
- Branch: `main`
- After making changes, commit and push automatically:
  ```
  git add <new files>
  git commit -m "descriptive message"
  git push
  ```
- Use commit messages that describe what changed functionally (e.g., "v1.21: Add color fade transition to branch tips")
