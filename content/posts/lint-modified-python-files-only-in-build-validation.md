---
title: "Lint Modified Python Files Only In Build Validation"
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

Have you ever needed to run any linting program (such as Flake8) only on specific Python files during build validation checks in Azure DevOps? Perhaps you are adopting a new linting / code validation tool on your extensive project but only want to enforce it for newly modified files? 

By utilizing Azure DevOps and Git Diff, Flake8 (or any other linting tool!) can be restricted to designated files by following the steps outlined below:

### 1. Setting Shallow Fetch 

When a PR is created in Azure DevOps, a merge branch containing all the modifications in a single commit is generated. By default, only the merge commit is downloaded to the agent used for build validation. To enable the generation of the difference, the previous state of the target branch is needed. To enable this, set the ShallowFetchDepth to 2.

```yaml
variables:
  - name: Agent.Source.Git.ShallowFetchDepth
    value: 2
```

### 2. Setting Up Python Environment

Set up a Python environment with your linting requirements from *requirements-test.txt*. In this instance, *requirements-test.txt* only contains Flake8 and Flake8_junit.

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

To lint only the modified files, use *git diff* with *xargs* to pipe the result to the flaking command:

- *git diff HEAD~ HEAD* generates the difference between the source branch and the target branch.

- *--name-only* filters to only return the name of the files changed.

- *--diff-filter=ACMRT* filters out any deleted files. Without this, if files are deleted during a PR, linting will fail as they do not exist.

- *xargs -r* is used to send the contents of *$workingDirectoryChangedFiles* to the linting command; in this case, we are using Flake8 with a redirection to an output file. The arguement *--no-run-if-empty (-r)* is used to prevent Flake8 from failing.

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

Azure DevOps supports including test results in PRs, allowing for a clear visualization of the linting results, without needing to dive into the pipeline runs. A report can be generated and published via:

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

Now, PRs containing Python files are automatically run through Flake8 without linting the entire repository. You can now enforce new linting requirements gradually across a project, without failing PRs for errors in the pre-existing code base.