# github-actions
Examples for using GitHub actions

## Getting Started

```yml
name: CI Deploy
on: push
env:
  MONGODB_DB_NAME: oldname
jobs:
  lint:
    env:
      MONGODB_DB_NAME: newname
    runs-on: ubuntu-latest
    steps:
    - name: Get code
      uses: actions/checkout@v3
    - name: Install NodeJS
      uses: actions/setup-node@v3
      with:
        node-version: 18
    - name: Install dependencies
      run: npm ci
    - name: Run lint
      run: npm run lint
  test:
    runs-on: ubuntu-latest
    steps:
    - name: Get code
      uses: actions/checkout@v3
    - name: Install NodeJS
      uses: actions/setup-node@v3
      with:
        node-version: 18
    - name: Install dependencies
      run: npm ci
    - name: Run test
      run: npm run test
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Get code
      uses: actions/checkout@v3
    - name: Install NodeJS
      uses: actions/setup-node@v3
      with:
        node-version: 18
    - name: Install dependencies
      run: npm ci
    - name: Run build
      run: npm run build


name: Output Issues Info
on: issues
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
    - name: Output event details
      run: echo "${{ toJson(github.event) }}"
```

## Matrix

All remaining steps in a matrix job will be skipped as soon as one of them fails.

## Artifacts

``` yml
steps:
  - name: Upload build artifacts
    uses: actions/upload-artifact@v3
    with:
      name: dist-files
      path: |
        dist
	package.json

steps:
  - name: Get build artifacts
    uses: actions/download-artifact@v3
    with:
      name: dist-files
```

The artifacts are downloaded to the runner's working directory.
If the artifact is a single folder, its contents are directly placed in the working directory.

## Outputs

``` yml
JOB_NEEDED:
  outputs:
    CUSTOM-NAME: ${{ steps.STEP_ID.outputs.CUSTOM_FILENAME }}
  steps:
    - name: Publish JS filename
      id: STEP_ID
      run: find dist/assets/*.js -type f -execdir echo 'CUSTOM_FILENAME={}' >> $GITHUB_OUTPUT ';'

needs: JOB_NEEDED
runs-on: ubuntu-latest
steps:
  - name: Output filename
    run: echo "${{ needs.JOB_NEEDED.outputs.CUSTOM-NAME }}"
```

The ``needs`` context contains the outputs of all jobs that are defined as a dependency of the current job.

## Caching

The GitHub Actions Cache is, by default, global, and therefore accessible from any workflow.

``` yml
steps:
  - name: Cache dependencies
    uses: actions/cache@v3
    with:
      path: ~/.npm
      key: deps-node-modules-${{ hashFiles('**/package-lock.json') }}
  - name: Install dependencies
    run: npm ci
```

``~/.npm`` is where the dependencies are downloaded.
Caching it ensures that other jobs that use the cache don't need to download the dependencies again.
The key needs to be variable, so that the cache is rewritten when necessary changes occur.
If it was constant, it would never change.
The GitHub function ``hashFiles()`` converts the input file into a hash string,
thus changing if the file contents change.
In this case, the ``package-lock.json`` file is used, ensuring that dependency changes result in new caches.
To reuse the cache in another job, simply replicate the step:

``` yml
steps:
  - name: Cache dependencies
    uses: actions/cache@v3
    with:
      path: ~/.npm
      key: deps-node-modules-${{ hashFiles('**/package-lock.json') }}
```

## Environment Variables

Environment variables are accessible from the workflow in two ways:

``` yml
# (1)
run: ${{ env.ENVIRONMENT_VARIABLE_NAME }}

# (2)
run: $ENVIRONMENT_VARIABLE_NAME
```

GitHub provides a set of default environment variables for general use:
https://docs.github.com/en/actions/learn-github-actions/variables#default-environment-variables

## Conditionals

By default, Steps and Jobs have the condition ``if: success()``.
Other conditional functions are necessary to execute Steps/Jobs despite failure/cancellation.

- ``failure()`` -> returns true when any previous Step or Job failed
- ``success()`` -> returns true when none of the previous steps have failed
- ``always()`` -> returns true even when the workflow has been cancelled
- ``cancelled()`` -> returns true if the workflow has been

For failure/success/cancellation conditions to be met, the condition must be evaluated after the execution of other Steps/Jobs.
Hence, Job/Step dependencies must be specified.

``success`` / ``failure`` / ``cancelled`` / ``always()`` are assessed based on the step's "outcome" property.
The ``continue-on-error`` setting, however, impacts the step's ``conclusion``,
which is evaluated after the ``outcome``.
Hence, when ``continue-on-error`` is set to true, the workflow does not fail because of this step.

## Skip Workflow

To skip a workflow (e.g. if the commit consists of changes that don't need CI to be run),
add ``[skip ci]``, etc.

Refer to https://docs.github.com/en/actions/managing-workflow-runs/skipping-workflow-runs
for more skipping expressions.

## Security

To prevent against script injection attacks, using actions instead of inline shell scripts is recommended.

## Permissions

By default, GitHub actions have full permissions.
In order to limit permissions, specify at least one, using:

``` yml
job:
  build:
    permissions:
      issues: write
```

That is because every job generates a token accessible through ``secrets.GITHUB_TOKEN``. If permissions aren't specified, the token has full permissions. This default behaviour can be changed to be more restrictive.

## Custom Actions

⚠ When using custom actions, **the repository that owns the action must be checked out**.

./.github/actions/cached-deps/action.yml
``` yml
name: 'Get & Cache Deps'
description: 'Get the deps (via npm) and cache them.'
runs:
  using: 'composite'
  steps:
    - name: Cache deps
      id: cache
      uses: actions/cache@v3
      with:
        path: node_modules
        key: deps-node-modules-${{ hashFiles('**/package-lock.json') }}
    - name: Install deps
      if: steps.cache.outputs.cache-hit != 'true'
      run: npm ci
      shell: bash  # In custom composite actions it is mandatory to specify the shell
```

./.github/workflows/deploy.yml
``` yml
[...]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Load & cache deps
        uses: ./.github/actions/cached/deps
```