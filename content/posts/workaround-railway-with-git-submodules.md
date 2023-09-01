---
title: "Notes: Workaround to use Railway with Git Submodules"
date: 2023-08-31T20:04:45-04:00
draft: fales
description: "A quick workaround to use railway.app to deploy a project that
uses git submodules. Railway doesn't support git submodules yet, so this is a
quick workaround to get it working / deploying."
tags: ["CI/CD", "railway", "git", "submodules"]
categories: ["software", "CI/CD", "railway"]
keywords: ["CI/CD", "railway", "git", "submodules", "workflows"]
---
**TL;DR** - Railway doesn't support git submodules yet. They probably will at
some point - here's the [feature request](https://feedback.railway.app/feature-requests/p/git-submodules-support). In the meantime, here's a quick workaround to get it working / deploying. 

## Use Github Workflow
You can deploy projects to [railway.app](https://railway.app) directly from
github, but they do not support [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules) yet. So your repo gets cloned as usual and railway
attempts to build/deploy. However, the submodules are not included, and the build will
fail. Seems like they will eventually add this [feature](https://feedback.railway.app/feature-requests/p/git-submodules-support),
but until then here's the workaround I used. The idea is to use a Github 
Workflow to:
- Clone your repo (including submodules)
- Remove the submodule from the index / remove the .gitmodules file
- Still keeping the actual code from the submodule
- Switch to a new branch, add the submodule code back as a regular directory
- Commit and push the new branch to github
- Deploy the new branch to railway

This can be set up using to trigger on a push to the main branch, and then just
set up railway to deploy from the new branch. In my case I called the new branch
`deploy`.

Here's the [workflow](https://github.com/heathhenley/ChEBot/blob/main/.github/workflows/git_submodule_workaround.yaml) that I used:

```yaml
name: "Pull submodule / Push to deploy branch"
on:
  push:
    branches:
      - main
jobs:
  pull_submodule_and_deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: true
      - name: Update commit author
        run: |
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
      - name: Push to deploy branch
        run: |
          git checkout -b deploy
          git rm --cached api/ChatGPTBot # here api/ChatGPTBot is a submodule
          git rm .gitmodules
          rm -rf api/ChatGPTBot/.git
          git add api/ChatGPTBot
          git commit -m "Checkout repo + submodule, add all and commit"
          git push origin deploy -f
```

And that's it! That's one simple somewhat hacky workaround to get railway to
run your project with submodules.