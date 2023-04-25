---
title: "Flake Changed Python Files Only On PR"
date: 2023-04-24T15:31:44+01:00
tags: ["Azure DevOps", "Python", "Flake8"]
draft: false
author: "Me"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
---

Have you ever needed to run linting or Flake8 only on specific Python files during build validation checks on Pull Requests (PRs) in Azure DevOps?

In certain situations, such as implementing Flake8 on a pre-existing large-scale project, you may only want to enforce Flake8 checks on the modified files in a PR. By utilizing Azure DevOps and Git Difference, Flake8 (or any other linting tool!) can be restricted to designated files by following the steps outlined below:

### 1. Setting Shallow Fetch 

When a PR is created in Azure DevOps, a merge branch containing all the modifications in a single commit is generated. However, by default, only the merge commit is downloaded. To enable the generation of the difference by downloading the previous state of the target branch, set the ShallowFetchDepth to 2.

```yaml
variables:
  - name: Agent.Source.Git.ShallowFetchDepth
    value: 2
```

### 2. Setting Up Python Requirements

Set up a Python environment with your testing requirements from *requirements-test.txt*. In this instance, *requirements-test.txt* only contains Flake8 and Flake8_junit.

```yaml
- task: UsePythonVersion@0
    displayName: "Use Python 3.9"
    inputs:
        versionSpec: 3.9
        architecture: "x64"

    - task: Powershell@2
    displayName: "Upgrade Pip"
    inputs:
        targetType: "inline"
        script: "python -m pip install --upgrade pip"

    - task: Powershell@2
    displayName: "Install test requirements"
    inputs:
        targetType: "inline"
        script: "python -m pip install -r ./requirements-test.txt"
```

### 3. Generating Differences & Flaking

To flake only changed files, use *git diff* with *xargs* to pipe the result to the flaking command:

- *git diff HEAD~ HEAD* generates the difference between the source branch and the target branch.

- *--diff-filter=ACMRT* filters out any deleted files. Without this, if files are deleted during a PR, linting will fail as they do not exist.

- *xargs -r* is used to send the contents of *$workingDirectoryChangedFiles* to the linting command; in this case, we are using Flake8 with a redirection to an output file*--no-run-if-empty (-r)* is used to prevent Flake8 from failing.

- Variable *PythonFilesChanged* is set to ensure that if no Python files are included in the PR, further tasks that use the generated report do not fail.

```yaml
- task: Powershell@2
    displayName: "Validate Flake8 Compliance"
    condition: and(succeeded(), eq(variables['Build.Reason'], 'PullRequest'))
    inputs:
        targetType: "inline"
        script: |
        mkdir $(Pipeline.Workspace)/lint-tests

        $workingDirectoryChangedFiles = $(git diff --name-only --diff-filter=ACMRT HEAD~ HEAD | grep -i "*\.py$")
        
        echo $workingDirectoryChangedFiles | xargs -r -t flake8 --tee --output-file $(Pipeline.Workspace)/lint-tests/lint-results.txt

        if($workingDirectoryChangedFiles.Length -gt 0) {
            Write-Output "##vso[task.setvariable variable=PythonFilesChanged]True"
        } else {
            Write-Output "##vso[task.setvariable variable=PythonFilesChanged]False"
        }
        workingDirectory: "$(Build.Repository.LocalPath)"
```

### 4. Generating Reports

Azure DevOps supports including test results in PR summaries, which allows for clear visualization of the report results without needing to dive into the pipeline runs. A report can be generated and published via:

* *flake8_junit* converts the Flake8 output file into a JUnit format that is readable by Azure DevOps.

* *PublishTestResults@2* uploads the generated XML file to Azure DevOps and publishes it once the pipeline is completed.

```yaml
- task: Powershell@2
    displayName: "Generate Flake8 Report"
    condition: and(succeeded(), or(ne(variables['Build.Reason'], 'PullRequest'), eq(variables['PythonFilesChanged'], True)))
    inputs:
        targetType: "inline"
        script: "flake8_junit $(Pipeline.Workspace)/lint-tests/lint-results.txt $(Pipeline.Workspace)/lint-tests/lint-results.xml"
        workingDirectory: "$(Build.Repository.LocalPath)"

- task: PublishTestResults@2
displayName: "Publish Flake8 Report"
    condition: and(succeeded(), or(ne(variables['Build.Reason'], 'PullRequest'), eq(variables['PythonFilesChanged'], True)))
    inputs:
        testResultsFormat: "JUnit"
        testResultsFiles: "$(Pipeline.Workspace)/lint-tests/lint-results.xml"
        mergeTestResults: false
        failTaskOnFailedTests: false
        testRunTitle: "Python Flake8"
```

### 5. Adding Branch Policies

Add a build validation step referencing the YAML pipeline to automatically run this on future PRs. 

Now, PRs containing Python files are automatically run through Flake8 without linting the entire repository.