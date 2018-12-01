# Pull request validation based on GitHub labels using Azure DevOps

This repository demonstrate how to use Azure DevOps pipelines to validate PRs, customizing job execution based on GitHub Pull Request labels.

![Overview](./media/overview.png)

## Scenario

Managing an OSS project that does static validation on PR requests. However, team wants to run full ci pipeline including deployment and integration tests if a specific Label is applied to PR.

## Solution

Solution has two key parts:

1. For a pipeline started for PR validation, check if it contains "fullci" label

```yaml
  # If reason build is started is "PullRequest"
  # Use GitHub API to check if PR has specific label ("fullci")
  # GitHub API: https://api.github.com/repos/{organization-or-name}/{project}/issues/{pr-number}
  # If found, set output variable prHasCILabel to true
  # echo "##vso[task.setvariable variable=prHasCILabel;isOutput=true]true"
  - bash: |
     echo "Looking for label at https://api.github.com/repos/$BUILD_REPOSITORY_ID/issues/$SYSTEM_PULLREQUEST_PULLREQUESTNUMBER/labels"
     if curl -s "https://api.github.com/repos/$BUILD_REPOSITORY_ID/issues/$SYSTEM_PULLREQUEST_PULLREQUESTNUMBER/labels" | grep '"name": "fullci"'
     then
       echo "##vso[task.setvariable variable=prHasCILabel;isOutput=true]true"
       echo "[INFO] fullci label found!"
     fi
    displayName: Check for CI label build on PR
    condition: eq(variables['Build.Reason'], 'PullRequest') # only run step if it is a PR
    name: checkPrLabel
```

2. Define Full CI job that is only executed against master, dev or a PR with "fullci" label (from previous job/step verification)

```yaml
# This job is executed only if:
# - Commit on master or dev
# - Validation of a PR with 'fullci' label
- job: full_ci
  displayName: Full CI
  dependsOn: build_and_static_validation # depends on previous job
  # conditions to run:
  #  - master/dev commit
  #  - pr with 'fullci' label
  condition: or(in(variables['Build.SourceBranch'], 'refs/heads/master', 'refs/heads/dev'), eq(dependencies.build_and_static_validation.outputs['checkPrLabel.prHasCILabel'], true))
  pool:
    vmImage: 'Ubuntu 16.04'
```

## Important:

Currently by default Azure DevOps is overwritting the PR trigger configuration of the yaml. To follow what is defined in the yaml file, change the build configuration the following way:

![PR validation setting](./media/pr-devops-build-settings.png)

Pull request validation in yaml:

```yaml
# On PRs against master and dev
pr:
  branches:
    include:
    - master
    - dev
```