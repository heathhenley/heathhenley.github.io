---
title: "Error Checking out Large Repo on GitHub Actions Windows Runner"
date: 2023-12-06T18:07:36-05:00
draft: false
metathumbnail: "/gh_actions/error.png"
description: "Some notes on a weird continuous integration error I ran into cloning a large repo on a GitHub Actions Windows Runner with actions/checkout@v4. Just in case it helps someone else out! (or future me)"
tags: ["CI/CD", "software", "software-engineering", "notes"]
categories: ["software", "software-engineering"]
keywords: ["software", "software-engineering", "CI/CD", "github", "github-actions", "windows", "windows-runner", "error"]
---

**TL;DR:** If you're getting a weird looking
`Error: fatal: fetch-pack: invalid index-pack output` error checking
out a large repo on a GitHub Actions using a Windows Runner, try
switching to use HTTPS instead of SSH for your clones (by providing a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) instead of an SSH key).

![The fetch-pack error I had in actions](/gh_actions/error.png)

# Error Checking out Large Repo on GitHub Actions Windows Runner
## Summary
I was recently working on getting CI setup for a project that has a pretty large
repo with a few submodules and uses LFS for some large binaries. We use
[buildbot](https://www.buildbot.net/) with local runners and I'm working on
switching some (or maybe eventually all) of it run on hosted runners on GitHub
Actions instead. We're a windows shop, so we need to run on the Windows 
runners. I ran into this error when checking out our repo along with the
submodules and LFS files on the Windows 2022, using the
[actions/checkout@v4 action](https://github.com/actions/checkout):

```
 fetch-pack: unexpected disconnect while reading sideband packet
  Error: fatal: early EOF
  Error: fatal: fetch-pack: invalid index-pack output
```

If you are here, you probably have this problem in some context - so in my
case the only thing that worked consistently was switching from using SSH to
HTTPS for the clone. I did this by providing a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) instead of an SSH key, in that
configuration, the checkout action handles switching to HTTPS for you.

That looks like this in the .yml file:

```yaml
# ... other stuff
runs-on: windows-latest-l
steps:
  - uses: actions/checkout@v4
    with:
      submodules: 'recursive'
      lfs: true
      token: ${{ secrets.PAT }} # finely scoped personal access token
# ... other stuff
```

You might have different results, here are some related issues / answers I 
found while I was trying to figure this out:
- https://github.com/actions/checkout/issues/1379
- https://github.com/actions/checkout/issues/748
- https://stackoverflow.com/questions/68882027/git-error-fatal-fetch-pack-invalid-index-pack-output

I tried a bunch of iterations on the suggestions on those issues and more,
mostly different `git config` settings, but nothing worked consistently. In
case you want to give it a try, here are some of the settings I tried, with
different combinations of values for all of them:

```
git config --global http.postBuffer 1048576000
git config --global core.packedGitLimit 4095m
git config --global core.packedGitWindowSize 4095m
git config --global pack.deltaCacheSize 4095m
git config --global pack.packSizeLimit 4095m
git config --global pack.windowMemory 4095m
```

Which I stuck in the .yml file just before the checkout step:

```yaml
- name: Set git config
  run: |
    git config --global http.postBuffer 1048576000
    git config --global core.packedGitLimit 4095m
    git config --global core.packedGitWindowSize 4095m
    git config --global pack.deltaCacheSize 4095m
    git config --global pack.packSizeLimit 4095m
    git config --global pack.windowMemory 4095m
```
I tried different values of  `core.compression` as well from 0 to 9 as some
people suggested that it fixed the same problem for them, but it didn't do the
trick for me.

I also tried cloning the repo manually in the action with depth=1 and then
running the submodule update and LFS pull manually, but that left me with the
same error. 🤷‍♂️

As I mentioned - the only thing that worked consistently was switching to HTTPS,
but I sincerely hope that you have better luck than I did - or maybe this will
save you some time!