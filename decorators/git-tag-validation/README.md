# Git Tag Validation Azure Pipeline Decorator

## Introduction

Git Tag validation decorator to ensure the tag commit Id exists in the commit history of the default branch.

More information ca be found from my blog post: [Git Tag Validation Using Azure DevOps Pipeline Decorator](https://blog.tyang.org/2024/12/23/git-tag-validation-using-ado-pipeline-decorator).

## Pre-requisite

Prior to packaging and publishing the decorator extension, `NodeJS` and `NPM` must be installed on the local computer.

**For Ubuntu Linux (such as WSL):**

```shell
#Install node.js
sudo apt install nodejs -y

#Install NPM
sudo apt install npm -y

#Install Cross-platform CLI for Azure DevOps
sudo npm i -g tfx-cli
```

**For Windows (using WinGet):**

```shell
#Install node.js
winget install -e OpenJS.NodeJS

#restart the terminal if required

#Install Cross-platform CLI for Azure DevOps
npm i -g tfx-cli
```

## Instruction
The following steps are required to create a brand new extension. It only needs to be done once, which is **NOT** required for this pipeline decorator because it's already created.

```shell
# initialize a new npm package manifest
npm init -y

#Install the Microsoft VSS Web Extension SDK package and save it to your npm package manifest
npm install azure-devops-extension-sdk --save
```

To create the packaged extension, make sure the [SemVer](https://semver.org/) `version` number is increased appropriately in the [`vss-extension.json`](./vss-extension.json) file, and then run the following command in the root directory of the extension:

```shell
#create extension
tfx extension create
```

After the extension is packaged, you can either manually publish it from the [Visual Studio Marketplace page](https://marketplace.visualstudio.com/manage/publishers/), or using the following command to publish it:

```shell
#publish or update extension and share with your ADO org

tfx extension publish --publisher [your-publisher] --auth-type 'pat' --token [your-pat-token] --username [your-username] --manifest ./[your-publisher].gittagvalidation-[version].vsix --share-with [your-ado-org]
```

The above command will publish / update the extension, and as well as sharing it with your ADO organization. you can also publish it without sharing by removing `--share-with [your-ado-org]` from the command. However, if it is not shared via the command line, it has to be manually shared from the marketplace page.

## Reference

- [Publish from command line](https://learn.microsoft.com/azure/devops/extend/publish/command-line?view=azure-devops)

## Pipeline Requirements

At the beginning of each pipeline job, a `checkout` task is automatically added as the first task of the job (regardless if it is explicitly defined in the YAML code). This task is responsible for checking out the source code from the repository. By default, the [`checkout`](https://learn.microsoft.com/azure/devops/pipelines/yaml-schema/steps-checkout?view=azure-pipelines) task performs a [`Shallow fetch`](https://learn.microsoft.com/azure/devops/pipelines/repos/pipeline-options-for-git?view=azure-devops&tabs=yaml#shallow-fetch) of the repository. This means that only the required commit is fetched, and not the entire repository history. This is done to optimize the pipeline run time and reduce the amount of data transferred over the network.

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

The `Git Tag Validation` pipeline decorator is configured to run immediately after each `checkout` task in the pipeline, with the following conditions:

`${{ if and(eq(variables['System.CollectionUri'], 'https://dev.azure.com/contoso/'), eq(variables['System.TeamProject'], 'MyProject'), eq(resources.repositories['self'].name, 'my-repo'), or(eq(variables['System.DefinitionId'], '1'), eq(variables['System.DefinitionId'], '2'), eq(variables['System.DefinitionId'], '3')), startsWith(variables['Build.SourceBranch'], 'refs/tags/')) }}:`

This configuration ensures the decorator task only gets executed when the pipeline meets the following conditions:

1. The ADO organization URI must be `https://dev.azure.com/contoso/` (`eq(variables['System.CollectionUri'], 'https://dev.azure.com/contoso/')`) - You will need to update this value to the URI of your ADO organization.
2. The ADO project name must be `MyProject` (`eq(variables['System.TeamProject'], 'MyProject')`) - You will need to update this value to the ADO project within your organization.
3. The repository name must be `my-repo` (`eq(resources.repositories['self'].name, 'my-repo')`) - You will need to update this value to the git repository name within your project.
4. The pipeline definition ID must be `1`, `2`, or `3` (`or(eq(variables['System.DefinitionId'], '1'), eq(variables['System.DefinitionId'], '2'), eq(variables['System.DefinitionId'], '3'))`) - Update this to the IDs of the pipelines that you want the pipeline decorator to target.
5. The pipeline must be triggered by a git tag (`startsWith(variables['Build.SourceBranch'], 'refs/tags/'))`)

### Git Fetch Depth

The decorator can be configured to check the latest `x` number of commits in the default branch. This is done by setting the `maxGitFetchDept` environment variable in the decorator YAML file. At the moment, the `maxGitFetchDept` is set to `3000` by default. This means that the decorator will check the latest 3000 commits in the default branch. If the tag is not created from one of the latest 3000 commits, the pipeline will fail.

This value can be adjusted to ensure a very old tag cannot be deployed.
