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

Action uses `bash` and [jq](https://github.com/mikefarah/yq) utility for prepare template.

All pre-defined Github runners contains `yq` by default.

**If you are self-hosted runner you SHOULD install `yq` manually!**

## Usage

```yaml
- uses: name212/action-dynamic-uses@v1
  with:
    ##! if uses or action_dir were not passed. Action will fail 
    
    # Action reference, e.g. `actions/setup-node@v3`. If not passed should pass `action_dir`
    uses: ''

    # Dir path to action in repo. If not passed should pass `uses`.
    # If `action_dir` in submodule path you should check action with `submodules: "recursive"`
    # or pass `checkout_ref`.
    action_dir: ''
    
    # Git reference to checkout. 
    # If passed, will checkout repo with `actions/checkout` with options
    #   fetch-depth: 0
    #   submodules: "recursive"
    checkout_ref: ''

    # YAML/JSON string of `inputs` for the running action action, e.g. `node-version: 18` or `{"node-version": "18"}`
    # by default uses empty object
    with: {}
```