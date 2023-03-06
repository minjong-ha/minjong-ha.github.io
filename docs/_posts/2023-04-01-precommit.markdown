---
layout: posts
title:  "Pre-commit: For better project management"
author: Minjong Ha
published: false
date:   2023-04-01 12:00:00 +0900
---

## Introduction

Pre-commit hook is a script or command that is executed before commit. 
It is a method that automatically performing some checks or actions on project before it is committed, which can help to catch errors or prevent mistakes.

You can install pre-commit with:
```bash
pip install pre-commit

apt install pre-commit
```

## Pre-commit Hook

To set up a pre-commit hook in git, you need to create a script or command and save it in a file named "pre-commit" in your repository: ".git/hooks". 
The script should exit with a non-zero status code if the commit should be rejected, or with a zero status code if the commit is allowed.

For example, Suppose that I want to apply 'black' and check format with 'pylint' on my python files that I want to commit.
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

However, If I just check its score with pylint and commit whatever the score is, it is meaningless action..
'pylint' is valuable only when it denies commit that not satisfying the minimum score.
I could use following hook that checking pylint score:
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

Now the pre-commit hook check the pylint score and refuse to commit if it does not satisfy the minimum score.
In this case, I set the minimum score as 9.0.

'pylint' is useful tool to maintain the code quality.
However, sometimes it feels too tight for work.
You can add exceptions for pylint with:
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
pylint_output=$(pylint --disable=C0103 $files)
pylint_score=$(echo "$pylint_output" | awk '/Your code has been rated at/ {print $8}')

# Check if the pylint score is below 9.0 and exit with an error message if it is
if (( $(echo "$pylint_score < 9.0" | bc -l) )); then
  echo "Pylint score is below 9.0. Aborting commit."
  exit 1
fi

# Add the changes to the commit
git add $files
```

In above case, I disable 'C0103' warning.

### .yaml file

Since the pre-commit hook script is in .git/ directory, it is not allowed to add the script as a file for git project.
Instead, pre-commit presents sharable configuration with yaml.
You can specificate the modules you want to use during the pre-commit step, and add it to git project.
Then anyone can install and run the script based on yaml.

For example, following codes is the contents of yaml:
```bash
# .pre-commit-hook.yaml

repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        language_version: python3.9
        exclude: ^tests/

  - repo: https://github.com/PyCQA/pylint
    rev: v2.16.4
    hooks:
      - id: pylint
        args:
            - --disable=C0103,E0401
        exclude: ^tests/

  - repo: local
    hooks:
      - id: check-pylint-score
        name: Check Pylint Score
        entry: bash -c 'pylint --disable=C0103,E0401 $GIT_PARAMS | awk "/Your code has been rated at/ {if (\$8 < 8.0) {exit 1}}"'
        language: system
        files: '\.py$'
        verbose: true

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
    - id: trailing-whitespace
    - id: end-of-file-fixer
    - id: check-added-large-files
```

You can install the pre-commit hook with:
```bash
pre-commit install
```

You can run pre-commit hook only with:
```bash
pre-commit run $(file_names)
pre-commit run --all-files
```

If you want to commit regardless the result of pre-commit, run with:
```bash
git commit -a -m "COMMIT_LOG" --no-verify
```


### pyproject.toml

Configurations such as `--disable` in pylint should be applied on entire project.
For this reason, managing common options with pyproject.toml is a great choice.
Following is a `pyproject.toml` file for `--disable`:
```bash
[tool.pylint.messages_control]
confidence = []
disable = [
    'invalid-name',
    'import-error',
    'unspecified-encoding',
    'consider-using-with',
    'bare-except',
    'c-extension-no-member',
    'too-many-arguments',
    'no-name-in-module',
    'too-few-public-methods',
    'unused-import',
    'logging-too-many-args',
    'too-many-public-methods',
    'duplicate-code',
]

# https://pylint.readthedocs.io/en/latest/user_guide/messages/messages_overview.html
```

Since `pyproject.toml` already specifies the `--disable` options, there is no need to mention it on `.pre-commit-config.yaml`:
```bash
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        language_version: python3.9
        exclude: ^tests/

  - repo: https://github.com/PyCQA/pylint
    rev: v2.16.4
    hooks:
      - id: pylint
```



## Appendix

### Radon

Radon is a Python library that provides tools for analyzing the complexity of Python code. 
It uses various metrics, such as cyclomatic complexity, to measure the complexity of code and provide insights into its maintainability.

```bash
radon cc /path/to/python/project
```


