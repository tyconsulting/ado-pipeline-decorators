# Git Tag Validation Azure Pipeline Decorator

## Pre-requisite

### Ubuntu Linux

```shell
#Install node.js
sudo apt install nodejs -y

#Install NPM
sudo apt install npm -y

#Install Cross-platform CLI for Azure DevOps
sudo npm i -g tfx-cli
```

### Windows

```shell
#Install node.js
winget install -e OpenJS.NodeJS

#restart the terminal if required

#Install Cross-platform CLI for Azure DevOps
npm i -g tfx-cli

```

## Instruction

Run the following commands in PowerShell under the directory of the extension project:

```shell
# initialize a new npm package manifest
npm init -y

#Install the Microsoft VSS Web Extension SDK package and save it to your npm package manifest
npm install azure-devops-extension-sdk --save

#create extension
tfx extension create

#publish or update extension

tfx extension publish --publisher <your-publisher> --auth-type 'pat' --token <your-pat-token> --username <your-username> --manifest ./<your-publisher>.gittagvalidation-<version>.vsix --share-with <your-ado-org>
```

## Reference

- [Publish from command line](https://learn.microsoft.com/en-us/azure/devops/extend/publish/command-line?view=azure-devops)

## Pipeline Requirements

At the beginning of each pipeline job, a `checkout` task is automatically added as the first task of the job (regardless if it is explicitly defined in the YAML code). This task is responsible for checking out the source code from the repository. By default, the [`checkout`](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/steps-checkout?view=azure-pipelines) task performs a [`Shallow fetch`](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/pipeline-options-for-git?view=azure-devops&tabs=yaml#shallow-fetch) of the repository. This means that only the required commit is fetched, and not the entire repository history. This is done to optimize the pipeline run time and reduce the amount of data transferred over the network.

In order for the git tag validation decorator to work, the `checkout` task must be configured not use the `shallow fetch` method. This can be done by setting the `fetchDepth` parameter of the `checkout` task to `0`. This will fetch the entire repository history, including all tags, branches, and commits.

To configure the `checkout` task to perform a `Deep fetch`, add the following snippet to the YAML file at the beginning of each job (under the `steps` section):

```yaml
steps:
  - checkout: self
    fetchDepth: 0
```

If this configuration is not set, the pipeline will fail if it's executed under a git tag and the tag is not created from the latest commit of the default Git branch.

## Configuration

The Git Tag Validation Azure Pipeline Decorator is configured as the following.

### Conditions

The condition is configured as:

`${{ if and(eq(variables['System.CollectionUri'], 'https://dev.azure.com/eHealthCloud/'), eq(variables['System.TeamProject'], 'LZ-Azure'), eq(resources.repositories['self'].name, 'policies'), or(eq(variables['System.DefinitionId'], '84'), eq(variables['System.DefinitionId'], '85'), eq(variables['System.DefinitionId'], '86')), startsWith(variables['Build.SourceBranch'], 'refs/tags/')) }}:`

This configuration is used to only run the decorator task when the pipeline meets the following conditions:

1. The ADO organization URI must be `https://dev.azure.com/eHealthCloud/`
2. The ADO project name must be `LZ-Azure`
3. The repository name must be `policies`
4. The pipeline definition ID must be `84` (Policy Definition), `85` (Policy Initiative), or `86` (Policy Assignment)
5. The pipeline must be triggered by a git tag

### Git Fetch Depth

The decorator can be configured to check the latest `x` number of commits in the default branch. This is done by setting the `maxGitFetchDept` environment variable in the decorator YAML file. At the moment, the `maxGitFetchDept` is set to `3000` by default. This means that the decorator will check the latest 3000 commits in the default branch. If the tag is not created from one of the latest 3000 commits, the pipeline will fail.

This value can be adjusted to ensure a very old tag cannot be deployed.
