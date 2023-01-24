## Todo
- Images
- ~~Landing page redesign~~
- README
- ~~CONTRIBUTING.md~~
- GH Actions with build test
- Host with GH pages

# Integrator's Guide

A community repo for users that integrate with Vertica with provisioning, monitoring, management, storage, clients, loaders, and other tools.


## Setup

Before you can contribute, you must configure your git [username](https://docs.github.com/en/get-started/getting-started-with-git/setting-your-username-in-git) and [email address](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address): 
```shell
$ git config --global user.name "Jane Smith"
$ git config --global user.email "email@example.com"
```

### Quick setup

The Integrator's Guide uses [Hugo-supported Markdown](https://www.markdownguide.org/tools/hugo/). You can create or edit documentation in any text editor that supports Markdown.

### Advanced setup

This repository uses [Hugo](https://gohugo.io/) with a [Docsy](https://www.docsy.dev/) theme. To set up the complete authoring environment, install the following dependencies:
- [Go](https://go.dev/doc/install) (version 1.18+)
- [Hugo extended package](https://gohugo.io/installation/) (version 0.99.1)
- [Docsy](https://www.docsy.dev/docs/get-started/#installation-options)

## Contribute 

The Vertica team is always open to suggestions--please feel free to share your ideas about how to improve the Integrator's Guide. To provide suggestions, open an issue with details describing any features that you would like added or changed.

1. Fork the repo and checkout your local copy:
   ```shell
   $ git clone git@github.com:vertica/integrators-guide.git
   $ cd integrators-guide
   ```
   Your GitHub repository is called _origin_ in git. You should also set up **vertica/integrators-guide** as an _upstream_ remote:
   ```shell
   $ git remote add upstream git@github.com:vertica/integrators-guide.git
   $ git fetch upstream
   ```
2. Create a new branch for your contribution. Use a descriptive name:
   ```shell
   $ git checkout -b my-new-feature
   ```
3. Implement your fix or feature. Documentation files are located in the `/content` directory. For edits or additions to existing documentation, locate the file and make the changes.
   For guidelines about adding a new documentation, see [New Features](#new-features).
4. Push your work to the remote repository on GitHub:
   ```shell
   $ git push origin my-new-feature
   ```
5. Merge any updates to the upstream repo (vertica/integrators-guide) with the **rebase** command.
   The rebase command creates a linear history by moving your local commits onto the tip of the upstream commits.
   The following commands rebase your branch locally and force-push to your GitHub repository:
   ```shell
   $ git checkout my-new-feature
   $ git fetch upstream
   $ git rebase upstream/main
   $ git push -f origin my-new-feature
   ```
6. When your work is complete, create a [pull request](https://help.github.com/articles/creating-a-pull-request/) to `vertica:main`.
   A good pull request has:
   - Commits with one logical change in each.
   - Well-formed messages for each commit.


### New features

For new documentation, use the following guidelines:
- Name directories and files with [kebob case](https://en.wiktionary.org/wiki/kebab_case).
- Each new directory must have an `_index.md` file that serves as a top-level table of contents page for the new section.
- You must include [front matter](https://gohugo.io/content-management/front-matter/) at the top of each new markdown file. At minimum, the front matter must provide the title and weight.
  For example, the following front matter is YAML-formatted and located at the top of the `/content/containers/_index.md` file:
  ```yaml
    ---
    title: "Containerized Vertica"
    linkTitle: "Containerized Vertica"
    weight: 20
    ---
  ```
  The `weight` value determines the order of the table of contents (TOC). For example, a topic with `weight: 20` is listed below a topic with `weight: 30`. Use multiples of 10, and consider the following:
  - For `_index.md` files, `weight` determines where the the directory in the left-hand TOC.
  - Any other `*.md` files, `weight` determines the topic placement in the right-hand TOC.

### Bug reports

If you find a bug or typo, submit an issue. If you opened an issue and then later resolved it on your own, comment on the issue and then close the issue.

#### Commits

After you make changes on your branch, stage and commit as often as necessary:

```shell
$ git add .
$ git commit -m 'Add new Dockerfile example'
```

When writing the commit message:
- Describe precisely what the commit does.
- Limit each line of the commit message to 72 characters.
- Include the issue number `#N`, if the commit is related to an issue.


#### Tests

We test that the Hugo can build and serve the documentation. This test should take only a few moments.


#### About CI
Unit tests are run automatically for each commit in the PR. After you create the PR, you can run tests manually for each subsequent commit by selecting the workflow among the list in the **Actions** section of Github.

#### Sign the CLA
Before we accept a pull request, we ask contributors to sign a Contributor License Agreement (CLA) to confirm they have the right to donate the code. To electronically sign the CLA, follow the comment from **CLAassistant** on your pull request page. 

#### Review
Pull requests are usually reviewed within a few days. To address comments:
1. Apply your changes in new commits.
2. Rebase your branch and force-push to the same branch.
3. Re-run the test suite to ensure that tests are still passing. 

To produce a clean commit history, our maintainers do squash merging after your PR is approved. Squash merging combines all of your PR commits into a single commit in the master branch.

After your pull request is merged, you can safely delete your branch and pull the changes from the upstream repository.


## Additional resources
- [Hugo-supported Markdown](https://www.markdownguide.org/tools/hugo/)


## License

Distributed under the Apache 2.0 License. See [LICENSE](LICENSE) for more information.