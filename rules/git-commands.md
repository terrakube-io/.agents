---
trigger: model_decision
description: Git Command Execution Context
---

* **Workspace Structure:** The workspace root contains multiple independent Git repositories located in its immediate subfolders (`root/project-a`, `root/project-b`, etc.). The root folder itself is not a Git repository.
* **Execution Target Rule:** Whenever you run, suggest, or generate Git-related commands (e.g., `git status`, `git commit`, `git push`, `git checkout`), you must target the specific sub-project folder where the changes are taking place.
* **Prohibition:** Never execute bare `git` commands from the workspace root directory unless explicitly instructed to perform an operation across all sub-modules via a multi-repo script. Always scope terminal commands to the respective project directory (e.g., using `cd ./project-name && git ...` or by setting the working directory to the sub-folder).
* **Release Branch & Backport Conventions:**
  - Terrakube maintenance branches use `.X` notation: `release/<major>.<minor>.X` (e.g., `release/2.34.X`). Full semver strings (e.g. `2.34.0`) are git tags, not branches.
  - Backport branches must always be created from `origin/release/<major>.<minor>.X` and target `release/<major>.<minor>.X`.
  - Always use `git cherry-pick -x <sha>` so that the backport commit records its origin commit on `main`.
