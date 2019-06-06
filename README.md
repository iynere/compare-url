# Compare URL Orb [![CircleCI status](https://circleci.com/gh/iynere/compare-url.svg "CircleCI status")](https://circleci.com/gh/iynere/compare-url) [![CircleCI Orb Version](https://img.shields.io/badge/endpoint.svg?url=https://badges.circleci.io/orb/iynere/compare-url)](https://circleci.com/orbs/registry/orb/iynere/compare-url) [![CircleCI Community](https://img.shields.io/badge/community-CircleCI%20Discuss-343434.svg)](https://discuss.circleci.com/c/ecosystem/orbs)
CircleCI's 2.1 config processing preview disables the `$CIRCLE_COMPARE_URL` environment variable, useful when working with monorepo projects. This orb manually recreates (and slightly improves!) it.

## Functionality
Originally, and as recreated here, `$CIRCLE_COMPARE_URL` outputs a URL of the following form, slighly different for GitHub vs. Bitbucket projects:

```
# GitHub
https://github.com/$CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME/compare/$COMMIT1...$COMMIT2

# Bitbucket
https://bitbucket.org/$CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME/branches/compare/$COMMIT1...$COMMIT2
```

`$COMMIT2` always represents the current job's commit (`$CIRCLE_SHA1`). In the most common use case, `$COMMIT1` will be the most recent previously pushed commit on the same branch as the current job.

If the current branch is new or has only ever had a single commit pushed to it, then `$COMMIT1` will be the most recent [ancestor commit](https://git-scm.com/docs/git-merge-base) as defined in the `git` specifications (whereas the original `$CIRCLE_COMPARE_URL` environment variable would, in this case, instead output a compare URL containing only `$COMMIT2`—essentially unusable in the monorepo scenario that this orb addresses).

##  Usage
Declare the orb in your config.yml file:

```yaml
orbs:
  circle-compare-url: iynere/compare-url@x.y.z
```

Then call the orb's `reconstruct` command or job:

### Command usage
```yaml
steps:
  - checkout
  - circle-compare-url/reconstruct
```

### Job usage
```yaml
workflows:
  deploy:
    jobs:
      - circle-compare-url/reconstruct
      - deploy:
          requires:
            - circle-compare-url/reconstruct
```

### `use` command
Finally, the `circle-compare-url/use` command saves the `CIRCLE_COMPARE_URL` value, previously stored in file, to a local environment variable and transforms it into a true commit range value (stored as a `COMMIT_RANGE` environment variable), ready to be utilized as desired:

```yaml
- circle-compare-url/use:
    step-name: Desired step name to display on CircleCI
    attach-workspace: # set this to `true` if `reconstruct` was called as a job; default is `false`
    custom-logic: |
      # what would you like to do with the $CIRCLE_COMPARE_URL/$COMMIT_RANGE values?
      # this typically involves some kind of dynamic decision-making about release types,
      # based on what level of changes were made to your source code between the base commit and the current commit
      # for examples, see below

```

## Parameters

### `reconstruct`
The orb's `reconstruct` command and job both take three optional parameters:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `circle-token` | `env_var_name` | `CIRCLE_TOKEN` | Name of environment variable storing your CircleCI API token |
| `project-path` | `string` | `~/project` | Absolute path to your project's base directory, for running `git` commands |
| `debug` | boolean | `false` | Additional debugging output for folks developing the orb |

Its job also takes an optional fourth paramater, which allows users to run the job in a [smaller container](https://circleci.com/docs/2.0/configuration-reference/#resource_class) to conserve resources:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `resource-class` | `enum` | `medium` | Container size for `reconstruct` job (`["small", "medium"]`)

### `use`
The `use` command also takes three optional parameters and one that is technically optional but typically required:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `step-name` | `string` | Evaluate/use CIRCLE_COMPARE_URL | Specify a custom step name for this command, if desired |
| `attach-workspace` | `boolean` | `false` | Attach a workspace for this command to use? Useful when this orb's `reconstruct` job is called upstream in a given workflow |
| `workspace-root` | `string` | "." | Workspace root path (either an absolute path or a path relative to the working directory), defaults to "." (the working directory) |
| `custom-logic` | `string` | echo "What should COMMIT_RANGE ($COMMIT_RANGE) be used for?" | What should be done with the commit information created by the `reconstruct` command/job? [See examples in the orb registry](https://circleci.com/orbs/registry/orb/iynere/compare-url) (or [below](#examples)) |

Refer to CircleCI's [Reusing Config](https://circleci.com/docs/2.0/reusing-config/#using-the-parameters-declaration) documentation for additional information about parameters.

## Examples
The below examples are drawn from [CircleCI's `circleci-orbs` monorepo](https://github.com/CircleCI-Public/circleci-orbs), where the Compare URL Orb is used to automate publishing changes to individual orbs. [See that repository's `config.yml` file](https://github.com/CircleCI-Public/circleci-orbs/blob/master/.circleci/config.yml) for a more recent iteration of what follows.

By default, [every new CircleCI step runs in a fresh shell](https://circleci.com/docs/2.0/configuration-reference/#run). Thus, any environment variables stored during the `circle-compare-url/reconstruct` step would not be available to subsequent steps, without additional configuration by the end user (e.g., exporting the environment variable to a `.bash_env` and then file manually sourcing that file in any subsequent steps).

To mitigate this problem, the orb outputs the `$CIRCLE_COMPARE_URL` data to a file called `CIRCLE_COMPARE_URL.txt`, making it available to any subsequent steps (and even subsequent jobs, via [Workspaces](https://circleci.com/docs/2.0/workflows/#using-workspaces-to-share-data-among-jobs)). It also persists this file, along with a `BASE_COMPARE_COMMIT.txt` file, to a workspace, for possible usage in downstream jobs.

Thus, as seen in the below examples, it may be necessary to save the contents of the `CIRCLE_COMPARE_URL.txt` file as a (step-localized) environment variable in any steps that will make use of the compare URL.

### Command example
```yaml
version: 2.1

orbs:
  compare-url: iynere/compare-url@x.y.z

jobs:
  publish:
    docker:
      - image: circleci/circleci-cli
    steps:
      - checkout

      - compare-url/reconstruct

      - compare-url/use:
          step-name: Publish modified orbs
          custom-logic: |
            for ORB in folder-containing-orb-subdirs/*/; do

              orbname=$(basename $ORB)

              if [[ $(git diff $COMMIT_RANGE --name-status | grep "$orbname") ]]; then

                echo "publishing ${orbname}"

                circleci orb publish ${ORB}/orb.yml namespace/${orbname}@version
              else
                echo "${orbname} not modified; no need to publish"
              fi
            done

workflows:
  publish-orbs:
    jobs:
      - publish
```

### Job example
```yaml
version: 2.1

orbs:
  compare-url: iynere/compare-url@x.y.z

jobs:
  publish:
    docker:
      - image: circleci/circleci-cli
    steps:
      - checkout

      - compare-url/use:
          step-name: Publish modified orbs
          attach-workspace: true
          command: |
            for ORB in folder-containing-orb-subdirs/*/; do

              orbname=$(basename $ORB)

              if [[ $(git diff $COMMIT_RANGE --name-status | grep "$orbname") ]]; then

                echo "publishing ${orbname}"

                circleci orb publish ${ORB}/orb.yml namespace/${orbname}@version
              else
                echo "${orbname} not modified; no need to publish"
              fi
            done

workflows:
  publish-orbs:
    jobs:
      - compare-url/reconstruct
      - publish:
          requires:
            - compare-url/reconstruct
```

## Contributing
See CircleCI's [Creating Orbs](https://circleci.com/docs/2.0/creating-orbs/) documentation to get started.

This orb has only minimal testing—issues, pull requests, or other suggestions are welcome towards the goal of improving test depth/coverage.

See [Creating automated build, test, and deploy workflows for orbs, part 1](https://circleci.com/blog/creating-automated-build-test-and-deploy-workflows-for-orbs/) and [
Creating automated build, test, and deploy workflows for orbs, part 2](https://circleci.com/blog/creating-automated-build-test-and-deploy-workflows-for-orbs-part-2/) for more information on automated orb testing/deployment.
