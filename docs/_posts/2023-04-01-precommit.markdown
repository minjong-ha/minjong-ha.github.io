---
layout: posts
title:  "Pre-commit: For better project management"
author: Minjong Ha
published: false
date:   2023-04-01 12:00:00 +0900
---

Pre-commit hook is a script or command that is executed before a commit is made. 
It's a way to automatically perform some checks or actions on project before it is committed, which can help you catch errors or prevent mistakes from being committed to your repository.

To set up a pre-commit hook in Git, you need to create a script or command and save it in a file named "pre-commit" in your repository's ".git/hooks" directory. 
This script should exit with a non-zero status code if the commit should be rejected, or with a zero status code if the commit is allowed.

For example, I want to apply 'black' and check format with 'pylint' on my python files that I want to commit.
In this case, I can write pre-commit hook:
```bash
#!/bin/bash

# Get the list of files that are being committed
files=$(git diff --cached --name-only --diff-filter=ACM "*.py")

# Exit if there are no Python files being committed
if [[ ! "$files" ]]; then
  exit 0
fi

# Run 'black' on the Python files
black $files

# Run 'pylint' on the Python files
pylint $files

# Add the changes to the commit
git add $files
```

If I add above file as executable on the '.git/hooks/pre-commit', hook will be called every time I try to commit files.
Above hook will do:
1. It gets the list of files that are being committed using the 'git diff' command.
2. It checks if there are any Python files being committed, and exits if there are none.
3. It runs the 'black' formatter and 'pylint' linter on the Python files.
4. It adds the changes to the commit.

However, If I just check its score with pylint, it has no meaning.
'pylint' is valuable only when it denies commit that not satisfying the minimum score.
I use following hook that checking pylint score:
```bash
#!/bin/bash

# Get the list of files that are being committed
files=$(git diff --cached --name-only --diff-filter=ACM "*.py")

# Exit if there are no Python files being committed
if [[ ! "$files" ]]; then
  exit 0
fi

# Run 'black' on the Python files
black $files

# Run 'pylint' on the Python files and get the score
pylint_output=$(pylint $files)
pylint_score=$(echo "$pylint_output" | awk '/Your code has been rated at/ {print $8}')

# Check if the pylint score is below 9.0 and exit with an error message if it is
if (( $(echo "$pylint_score < 9.0" | bc -l) )); then
  echo "Pylint score is below 9.0. Aborting commit."
  exit 1
fi

# Add the changes to the commit
git add $files
```
