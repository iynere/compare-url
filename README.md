# compare-url-orb
turning on CircleCI's 2.1 config processing preview disables the CIRCLE_COMPARE_URL environment variable, used when working with monorepos on CircleCI. this orb recreates it manually.

## note to self

need to add logic to account for multiple jobs sharing the same commit—use `$CIRCLE_WORKFLOW_ID` (`grep` the API response to check it)
