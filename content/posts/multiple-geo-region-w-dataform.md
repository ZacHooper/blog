+++
date = '2025-04-21T19:51:23+10:00'
draft = true
title = 'Multiple Geographic Regions with single Dataform repo'
+++

> tldr: Use seperate `git` branches to set your `workflow_settings.yaml` with the regions you want to deploy to and then setup release configurations using each different branches. Finally anytime you want to execute a pipeline in a particular region use the appropriate release configuration.

Do you have data requirements where you need to run the same pipelines in particular regions? Naively, we could copy the same code into region-specific repositories, or maintain separate configuration files for each region in the same repository. Ultimately, when we need to make a change to the pipeline, we run into the issue of needing to update the same code in multiple places, increasing the risk of making mistakes along the way.

At TRENDii, we encountered this exact challenge while ingesting our clients' product feeds. We needed to keep each client's data within their geographical zone for billing and administration purposes, but the processing logic for each client was identical regardless of region. This raised an important question: How do we maintain a DRY (Don't Repeat Yourself) repository while still deploying to multiple regions?

The solution was surprisingly simple - create `git` branches where we modify only the default region in our `workflow_settings.yaml` file. Then, whenever code is pushed to our `main` branch, we automatically sync these changes to our region-specific branches. For example:

```yaml
# workflow_settings.yaml (us-east1 branch)
defaultProject: project-id-us
defaultLocation: us-east1

# workflow_settings.yaml (europe-west1 branch)
defaultProject: project-id-eu
defaultLocation: europe-west1
```

> In our case, we had an added layer of complexity where each region was also deployed into their own projects. The solution for this was the same but we changed the `defaultProject` field in each branch as well as `defaultLocation`

Below a diagram shows how the code and data flows. Importantly, clear seperate lines are formed for each region.

- code changes pushed to the `main` branch.
- CI/CD automatically syncs those changes to region specific branches.
- Within each branch the `workflow_settings.yaml` file is updated to ensure all queries run within the right region.
- Region specific release configurations point to each branch
- Voila region specific pipelines using the same codebase

![[Geographic specific branch overview.drawio.png]]

The core bit code here is the CI/CD that syncs changes on the main branch. We are using GitHub to store our repo so a GitHub action was perfect option to do this for us.

Below is an action that syncs the main branch to `us-west`.

- It runs on every `push` to main and can also be triggered manually on demand.
- Checkouts the target branch
- Merges the latest commits on `main` into the target branch

```yaml
name: Sync Main to us-west

on:
  # Trigger when changes are pushed to main
  push:
    branches:
      - main
  # Optional: Allow manual trigger
  workflow_dispatch:

jobs:
  sync-branch:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          # Fetch all history for all branches
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config --global user.name 'GitHub Action'
          git config --global user.email 'action@github.com'

      - name: Sync main to target branch
        run: |
          # Replace 'target-branch' with your desired branch name
          TARGET_BRANCH="us-west"
          
          # Create target branch if it doesn't exist, or switch to it if it does
          git checkout $TARGET_BRANCH 2>/dev/null || git checkout -b $TARGET_BRANCH
          
          # Merge main into target branch
          git merge main --no-ff --no-edit -m "Merge main into $TARGET_BRANCH"
          
          # Push changes
          git push origin $TARGET_BRANCH
```

Using Git branches to manage region-specific configurations in Dataform provides an elegant solution to a common data engineering challenge. This approach not only maintains code consistency across regions but also significantly reduces the risk of errors that could arise from managing multiple codebases. By automating the sync process through GitHub Actions, we ensure that any updates to our core logic are seamlessly propagated across all regional deployments while maintaining region-specific settings.
