# action-dynamic-uses

Uses another github action dynamically with ref to action or dir in repo.

## Description

This action creates `./.tmp-dynamic-action/action.yml` with next [template](./_dyn-action-template.yml):
```yaml
outputs:
  outputs:
    value: ${{ '${{ toJSON(steps.run_sub.outputs) }}' }}
runs:
  using: composite
  steps:
  - name: Run dynamic sub-action
    id: run_sub
    uses: __USES_FOR_SUBSTITUTE__
    with: {}
```

`.runs.steps[0].uses` will replaced with `yq` on passed `uses` or `action_dir`.
`.runs.steps[0].with` will replaced with `yq` on passed `with`.

Outputs of action will stored in `outputs.outputs` as `JSON-string`.

After run action `./.tmp-dynamic-action` directory will be removed for run multiple dynamic actions in one job.

**WARNING!** If you use submodules for desired action, you should checkout to ref with `actions/checkout` action uses 
`submodules: "recursive"` option for init submodules or pass `checkout_ref` input.

Action provides some debug logs. You can rerun-job with [debug logging](https://github.blog/changelog/2022-05-24-github-actions-re-run-jobs-with-debug-logging/).

## Dependencies

Action uses `bash` and [yq](https://github.com/mikefarah/yq) utility for prepare template.

All pre-defined Github runners contains `yq` by default.

**If you are self-hosted runner you SHOULD install `yq` manually!**

## Usage

```yaml
- uses: name212/action-dynamic-uses@v1
  with:
    ##! if uses or action_dir were not passed. Action will fail 
    
    # Action reference or path, e.g. `actions/setup-node@v3`.
    #  If has prefix `dir:` uses directory in repository.
    #  If dir in submodule path you should check action with `submodules: "recursive"`
    #  or pass `checkout_ref`.
    # Required
    uses: ''

    # Git reference to checkout. 
    # If passed, will checkout repo with `actions/checkout` with options
    #   fetch-depth: 0
    #   submodules: "recursive"
    checkout_ref: ''

    # YAML/JSON string of `inputs` for the running action action, e.g. `node-version: 18` or `{"node-version": "18"}`
    # by default uses empty object string
    with: '{}'
```

### Examples

### Uses another action
```yaml
name: Pull request changes
on:
  pull_request:
    types:
      - opened
      - synchronize
      - reopened

jobs:
  uses_tests:
    name: "Run uses test"
    runs-on: ubuntu-latest
    timeout-minutes: 5

    steps:
      - &checkout_step
        name: Checkout
        uses: actions/checkout@v6.0.2
        with:
          fetch-depth: 0
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Get setup go version
        id: go_action
        run: |
          echo "version=6.4.0" >> $GITHUB_OUTPUT
      - name: Uses first (go)
        uses: ./
        with:
          # You can use outputs for dynamically choice version or action  
          uses: actions/setup-go@v${{ steps.go_action.outputs.version }}
          with: |
            go-version: '1.26.x'
      - name: Get go version
        run: go version

      - name: Get setup yq version and key
        id: yq_action
        run: |
          echo "sha=1b9b4ac5187171d2e5e3129be0cfa827c7f9d53d" >> $GITHUB_OUTPUT
          echo "key=a" >> $GITHUB_OUTPUT
      - name: Uses second (yq docker)
        uses: ./
        id: yq_run
        env:
          REPLACE: "replaced"
        with:
          uses: mikefarah/yq@${{ steps.yq_action.outputs.sha }}
          with: |
            cmd: |
              echo '{"${{ steps.yq_action.outputs.key }}": "b"}' | yq -o yaml -P '.${{ steps.yq_action.outputs.key }} = env(REPLACE)'
      - name: Print yq result
        env:
          # Extract output from yq action (yq action set output in .result key)
          RESULT: ${{ fromJSON(steps.yq_run.outputs.outputs).result }}
        run: echo "$RESULT"
```

### Uses sub-dir
```yaml
name: Pull request changes
on:
  pull_request:
    types:
      - opened
      - synchronize
      - reopened

jobs:
  dir_tests:
    name: "Run action dir (no sub-module) test"
    runs-on: ubuntu-latest
    timeout-minutes: 5

    steps:
      - name: Run sub-dir action
        uses: name212/action-dynamic-uses@v1
        with:
          # you can checkout to ref with this action
          # it helpful if you action dir is submodule
          # or not use checkout step in pipeline by hand
          checkout_ref: ${{ github.event.pull_request.head.sha }}
          uses: "dir:.sub-dir"

```