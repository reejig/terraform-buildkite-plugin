# Buildkite Terraform Plugin

![GitHub release (latest by date)](https://img.shields.io/github/v/release/codeninja55/terraform-buildkite-plugin) ![GitHub Release Date](https://img.shields.io/github/release-date/codeninja55/terraform-buildkite-plugin) ![GitHub contributors](https://img.shields.io/github/contributors/codeninja55/terraform-buildkite-plugin) ![GitHub issues](https://img.shields.io/github/issues/codeninja55/terraform-buildkite-plugin)

Runs Terraform as a Buildkite pipeline step.

## Disclaimer

This plugin does not claim to solve every problem related to running Terraform in a Buildkite pipeline. It provides a 
basis for expansion. We have tried to avoid providing so many options that the plugin becomes confusing, but we 
understand that use cases vary and many people do many different things with their Terraform workflows. If you find 
that a critical feature is either configured in a way that hinders your process or missing outright, please open an 
issue or contribute.

## Prerequisites

The plugin makes a few assumptions about your environment:

#### Initialization

It's up to you to define what flags are used for `terraform init` by taking advantage of `init_args`. This is going to 
vary depending on your provider(s) and/or backend(s).

#### Workspaces

If using a Terraform workspace, the workspace is selected before running `plan` and will also be used before running 
`apply`. By default, if the workspace doesn't exist it will be created. You may disable this behavior (and let the 
plugin explicitly fail if the workspace does not exist) by setting `auto_create_workspace: false` in the plugin config.

If using a Terraform workspace, the plugin will also look for a `.tfvars` file in the form 
`WORKSPACE_NAME-terraform.tfvars` by default. However, you can supply `vars_dir` and `vars_file_name` to override 
this behaviour.

If not using a Terraform workspace, the `default` workspace is used.

#### Agent Requirements

Your Buildkite agents will need Docker.

## Behavior

The plugin does a few things automatically:

- At the end of the Terraform `plan` step, the plugin automatically generates `tfplan.json` for a JSON formatted 
  version of the plan output in the `terraform/` directory. This is useful if you'd like to use a plugin like 
  [artifacts](https://github.com/buildkite-plugins/artifacts-buildkite-plugin) to move your Terraform plan between 
  steps, which is even more useful if you've opted to make two separate steps with a dedicated `apply` step with `plan` 
  disabled. We've provided an example of this below. You can also run the JSON output through something like OPA.
- At the end of the Terraform `plan` step, the plugin sets the Buildkite  meta-data field `tf_diff` to either `true` 
  or `false` based on whether or not  there are changes in the plan.

## Versioning

Leaving the `#v1.x.x` tag off of the end of the plugin name will automatically pull down the latest version.

## Examples

Add the following to your `pipeline.yml`:

```yml
steps:
  - label: "terraform"
    plugins:
      - echoboomer/terraform#v1.2.25:
          init_args:
            - "-input=false"
            - "-backend-config=bucket=my_gcp_bucket"
            - "-backend-config=prefix=my-prefix"
            - "-backend-config=credentials=sa.json"
```

While no commands are required, out of the box behavior may be undesirable 
without at least `init_args` and `directory`.

```yml
steps:
  - label: "terraform"
    plugins:
      - echoboomer/terraform#v1.2.25:
          directory: terraform
          init_args:
            - "-input=false"
            - "-backend-config=bucket=my_gcp_bucket"
            - "-backend-config=prefix=my-prefix"
            - "-backend-config=credentials=sa.json"
          image: myrepo/mycustomtfimage
          version: 1.0.0
          use_workspaces: true
          workspace: development
```

To pass in extra environment variables to the Docker container:

```yml
steps:
  - label: "terraform"
    plugins:
      - echoboomer/terraform#v1.2.25:
          directory: terraform
          env:
            - "FOO=foo"
            - "BAR=baz"
          init_args:
            - "-input=false"
            - "-backend-config=bucket=my_gcp_bucket"
            - "-backend-config=prefix=my-prefix"
            - "-backend-config=credentials=sa.json"
          image: myrepo/mycustomtfimage
          version: 0.12.21
          use_workspaces: true
          workspace: development
```

If you want an out of the box solution that simply executes a `plan` on 
non-master branches and `apply` on merge to master, you can use this:

```yml
steps:
  - label: "terraform"
    plugins:
      - echoboomer/terraform#v1.2.25:
          apply_master: true
          directory: terraform
          init_args:
            - "-input=false"
            - "-backend-config=bucket=my_gcp_bucket"
            - "-backend-config=prefix=my-prefix"
            - "-backend-config=credentials=sa.json"
          version: 0.12.21
```

This will simply look at `BUILDKITE_BRANCH` and only run the `apply` step if 
it is set to `master`.

You can also break your Terraform steps into different sections depending on 
your use case. For example:

```yml
steps:
  - label: "terraform plan"
    branches: "!master"
    plugins:
      - echoboomer/terraform#v1.2.25:
          directory: terraform
          init_args:
            - "-input=false"
            - "-backend-config=bucket=my_gcp_bucket"
            - "-backend-config=prefix=my-prefix"
            - "-backend-config=credentials=sa.json"
          version: 0.12.21
      - artifacts#v1.2.0:
          upload: "tfplan"
  - label: "terraform apply"
    branches: "master"
    plugins:
      - artifacts#v1.2.0:
          download: "tfplan"
      - echoboomer/terraform#v1.2.25:
          directory: terraform
          apply_only: true
          init_args:
            - "-input=false"
            - "-backend-config=bucket=my_gcp_bucket"
            - "-backend-config=prefix=my-prefix"
            - "-backend-config=credentials=sa.json"
          version: 1.0.0
```

This is useful if you want more control over the behavior of the plugin or if it is necessary to split apart the 
`apply` step for whatever reason.

## Configuration

### `apply` (Not Required, boolean)

Whether or not to run `terraform apply` on this step. The plugin only assumes `terraform plan` without this option. 
Set to `true` to run `terraform apply`.

### `apply_only` (Not Required, boolean)

If this option is supplied, `plan` is skipped and `apply` is forced. This is useful if creating separate steps for 
`plan` and `apply`. You do not need to use both this and `apply`.

### `apply_master` (Not Required, boolean)

If this option is supplied, `apply` will automatically run if `BUILDKITE_BRANCH` is `master`. This allows you to 
define a single `pipeline.yml` step that will provide a `plan` on pull request and `apply` on master merge.

### `debug` (Not Required, boolean)

If providing this option and setting it to `true`, additional output is provided to help troubleshoot.

### `directory` (Not required, string)

By default, this plugin assumes a directory `terraform` with all Terraform configurations in your root directory. 
You can override this by supplying the `directory` and setting it to the value where your Terraform files are.

### `env` (Not Required, string, array)

Extra environment variables to pass to the Docker container.

### `image` (Not Required, string)

If using a custom Docker image to run `terraform`, set it here. This should only be the `repo/image` string. Set the 
tag in `version`. Defaults to `hashicorp/terraform`.

### `init_args` (Not Required, string, array)

Arguments to pass to `terraform init`. Can be a `string` or `array` depending on needs. Is not required, but is likely 
critical for any Terraform commands to work.

### `no_validate` (Not Required, boolean)

If provided and set to `true`, the `terraform validate` step will be skipped.

### `precommand` (Not Required, string)

Adds a `pre-command` script that will run before the main Terraform plugin runs. This could be useful for downloading 
credentials or other setup steps before running Terraform.

### `skip_apply_no_diff` (Not Required, boolean)

If this is provided and set to `true`, the `apply` step will be skipped if `TF_DIFF` is also `false`. The latter 
variable is automatically exported during every `plan` step.

### `use_workspaces` (Not Required, boolean)

If you need to use Terraform workspaces in your repository, set this to `true`. You also need to pass in the name of 
a workspace.

### `vars_dir` (Not Required, string)

If there is a Terraform variables file, whether set with `vars_file_name` or `use_workspaces` / `workspace`, you can 
override where to find this file in the directory. Default is to assume it is in the root `directory` or `./terraform`.

### `vars_file_name` (Not Required, string)

If specified, a Terraform variables file will be passed to the command with the flag `--var-file 
$BUILDKITE_PLUGIN_TERRAFORM_VARS_FILE_NAME`. Additionally, you can also use the `vars_dir` property to specify where 
to find the vars file. 

If `use_workspaces` is set to `true`, the default behaviour is to use `$WORKSPACE-terraform.tfvars`.

### `version` (Not Required, string)

Which version of Terraform to use. Defaults to `0.13.0`. This is the tag applied to the Docker image, whether using the 
default of `hashicorp/terraform` or your own custom image.

### `volumes` (Not Required, string, array)

Additional volume mount statements to provide to the Docker container in the form `foo:bar`. Uses `-v` on the backend.

### `workspace` (Not Required, string)

If setting `use_workspaces` to `true`, pass in the Terraform workspace name here.

### `workspace_metadata_key` (Not Required, string)

If setting `use_workspaces` to `true`, pass in a 
[Buildkite metadata](https://buildkite.com/docs/pipelines/build-meta-data) key and the plugin will set the Terraform 
workspace based on the metadata value. 

Note: If `workspace` is also set it will be overridden.   

## Developing

To run the linting tool:

```shell
docker-compose up --build lint
```

## Contributing

1. Fork the repo.
2. Make your changes.
3. Make sure linting passes.
4. Commit and push your changes to your branch.
5. Open a pull request.
