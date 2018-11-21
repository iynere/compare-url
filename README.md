# Compare URL Orb

CircleCI's 2.1 config processing preview disables the `$CIRCLE_COMPARE_URL` environment variable, useful when working with monorepo projects. This orb manually recreates (and improves!) it.

## Functionality

Originally, and as recreated here, `$CIRCLE_COMPARE_URL` outputs a URL of the following form, slighly different for GitHub vs. Bitbucket projects:

```
https://github.com/$CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME/compare/$COMMIT_1...$COMMIT_2

https://bitbucket.org/$CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME/branches/compare/$COMMIT_1...$COMMIT_2
```

`$COMMIT_2` always represents the current job's commit (`$CIRCLE_SHA1`). In the most common use case, `$COMMIT_1` will be the most recent previously pushed commit on the same branch as the current job.

If the current branch is new or has only ever had a single commit pushed to it, then `$COMMIT_1` will be the most recent [ancestor commit](https://git-scm.com/docs/git-merge-base) as defined in the `git` specifications (whereas the original `$CIRCLE_COMPARE_URL` environment variable would, in this case, instead output a compare URL containing only `$COMMIT_2`—essentially unusable in the monorepo scenario that this orb addresses).

##  Usage
Declare the orb in your config.yml file:

```yaml
orbs:
  circle-compare-url: iynere/compare-url@volatile
```

Then call the orb's sole command, `reconstruct`, in a CircleCI job:

```yaml
steps:
  - checkout
  - circle-compare-url/reconstruct
```

### Parameters

The orb takes four optional parameters:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `circle-token` | string | `$CIRCLE_TOKEN` | Your CircleCI API token, defaults to `$CIRCLE_TOKEN` |
| `project-path` | string | `~/project` | Absolute path to your project's base directory, necessary for running `git` commands |
| `debug` | boolean | `false` | Additional debugging output for folks developing the orb |
| `debug-steps` | steps | `[]` | Additional debugging steps for folks developing the orb |

Refer to CircleCI's [Reusing Config](https://circleci.com/docs/2.0/reusing-config/#using-the-parameters-declaration) documentation for additional information about parameters.

## Example

The below example is drawn from [CircleCI's `circleci-orbs` monorepo](https://github.com/CircleCI-Public/circleci-orbs), where the Compare URL Orb is used to automate publishing changes to individual orbs. [See that repository's `config.yml` file](https://github.com/CircleCI-Public/circleci-orbs/blob/master/.circleci/config.yml) for a more recent iteration of what follows.

By default, [every new CircleCI step runs in a fresh shell](https://circleci.com/docs/2.0/configuration-reference/#run). Thus, any environment variables stored during the `circle-compare-url/reconstruct` step would not be available to subsequent steps, without additional configuration by the end user (e.g., exporting the environment variable to a `.bash_env` and then file manually sourcing that file in any subsequent steps).

To mitigate this problem, the orb outputs the `$CIRCLE_COMPARE_URL` data to a file called `CIRCLE_COMPARE_URL.txt`, making it available to any subsequent steps (and even subsequent jobs, via [Workspaces](https://circleci.com/docs/2.0/workflows/#using-workspaces-to-share-data-among-jobs)).

Thus, as seen in the below example, it may be necessary to save the contents of the `CIRCLE_COMPARE_URL.txt` file as a (step-localized) environment variable in any steps that will make use of the compare URL.

```
version: 2.1

orbs:
  compare-url: iynere/compare-url@volatile

workflows:
  version: 2
  publish-orbs:
    jobs:
      - publish

jobs:
  publish:
    docker:
      - image: circleci/circleci-cli
    steps:
      - checkout

      - compare-url/reconstruct

      - run:
          name: Publish modified orbs
          shell: /bin/bash -exo pipefail
          command: |
            # save value stored in file to a local env var
            CIRCLE_COMPARE_URL=$(cat CIRCLE_COMPARE_URL.txt)

            COMMIT_RANGE=$(echo $CIRCLE_COMPARE_URL | sed 's:^.*/compare/::g')

            echo "Commit range: $COMMIT_RANGE"

            for ORB in folder-containing-orb-subdirs/*/; do

              orbname=$(basename $ORB)

              if [[ $(git diff $COMMIT_RANGE --name-status | grep "$orbname") ]]; then

                echo "publishing ${orbname}"

                circleci orb publish ${ORB}/orb.yml namespace/${orbname}@version
              else
                echo "${orbname} not modified; no need to publish"
              fi
            done
```

## Contributing

See CircleCI's [Creating Orbs](https://circleci.com/docs/2.0/creating-orbs/) documentation to get started.

This orb has only minimal testing—issues, pull requests, or other suggestions are welcome towards the goal of improving test depth/coverage.

See [Testing orbs](https://github.com/CircleCI-Public/config-preview-sdk/blob/master/docs/orbs-testing.md) from CircleCI's 2.1 config preview SDK, or [this Discuss thread](https://discuss.circleci.com/t/testing-orbs), for further discussion of emerging orb testing best practices.
