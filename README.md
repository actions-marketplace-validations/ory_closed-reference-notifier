# Closed Reference Notifier Action

A GitHub action to open an issue when GitHub references in your code are closed.
These could be references to open pull requests or issues.

Quite often you will have references like this one in your code:

```js
// workaround for https://github.com/ory/kratos/issues/364
dsn = `${dsn}&multiStatements=true`
```

You will then either have to remind yourself to check back or (more likely) you
will stumble across the comment some time later to manually check whether the
issue got fixed. This simple GitHub actions searches your whole repository for
references to GitHub pulls and issues. When it finds a closed reference it will
open an issue in your repository. That way you will automatically be reminded to
check back on the reference.

## Configuration

Please note that this action has to run on node.js 14, which is currently only
possible using a workaround. Therefore, you have to make sure this action runs
on a sufficient node version, e.g. by using this minimal job config:

```yaml
jobs:
  find_closed_references:
    runs-on: ubuntu-latest
    name: Find closed references
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2-beta
        with:
          node-version: "14"
      - uses: ory/closed-reference-notifier@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

We recommend having this action run regularly e.g. daily. To enable access to
the GitHub API you have to set the token input (see below).

All other inputs are optional.

| Key         | Description                                                    | Default value    |
| ----------- | -------------------------------------------------------------- | ---------------- |
| ignore      | ignore paths, comma seperated list of .gitignore style entries | .git             |
| issueLabels | the labels to create issues with, comma seperated list         | closed reference |
| issueLimit  | the maximum number of issues to create, supposed to catch bugs | 5                |

Additionally to setting `ignore` as an input, you can also place a
`.reference-ignore` file in the root of your repo (entries should be `\n`
separated). Both, the input and the file will be concatenated.

Individual references can be marked as intentional by placing two exclamation
marks immediately behind the reference:

```js
// This prevents regression of github.com/testUser/testRepo/pull/420!!
t.Skip()
```

Note that the action fails when there are more issues to be created than the
limit allows. This is because GitHub Actions do not have manual approval.

Verbose example with all inputs:

```yaml
on:
  schedule:
    - cron: "0 7 * * *"

jobs:
  find_closed_references:
    runs-on: ubuntu-latest
    name: Find closed references
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2-beta
        with:
          node-version: "14"
      - uses: ory/closed-reference-notifier@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          ignore: .git
          issueLabels: closed reference
          issueLimit: 5
```

## Testing manually before automating

To manually test what references this action finds you can run

<!-- update when published to npm https://github.com/ory/closed-reference-notifier/issues/13 -->

```
$ npx https://github.com/ory/closed-reference-notifier.git local/path/to/repo ignore,list
```

## Manual workflow trigger

To allow for easy temporary issue limit raising, we recommend you add the
following manual trigger:

```yaml
on:
  workflow_dispatch:
    inputs:
      issueLimit:
        description: maximum number of issues to create
        required: true
        default: "5"

jobs:
  find_closed_references:
    runs-on: ubuntu-latest
    name: Find closed references
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2-beta
        with:
          node-version: "14"
      - uses: ory/closed-reference-notifier@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # fall back to 5 when the issueLimit is not available (e.g. with a scheduled event)
          issueLimit: ${{ github.event.inputs.issueLimit || '5' }}
```
