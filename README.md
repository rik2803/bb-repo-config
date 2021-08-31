# Create and update BB repositories

## Pre-requisites

* Use of:
  * `aws-account-config` and Service Accounts to generate the required credentials in
    the AWS Bastion account and roles to assume in the _functional_ AWS accounts
  * `bb-aws-utils` in the pipelines to set AWS credentials based and AWS roles
    inside the BB pipeline container
* Credentials on the Bastion AWS account and a role with permissions to read from
  the AWS SSM Parameter store
* BB API Token with at least these permissions:
  * Account (required for the v1.0 REST calls)
    * Read
    * Write
    * e-mail
  * Repositories:
    * Read
    * Write
    * Admin
  * Pipelines:
    * Read
    * Write
    * Edit variables
* BB OAuth consumer with at least these permissions (See
  [here](https://support.atlassian.com/bitbucket-cloud/docs/use-oauth-on-bitbucket-cloud/)
  for instructions to create the OAuth consumer):
  * Repositories:
    * Read
    * Write
    * Admin
  * Pipelines:
    * Read
    * Write
    * Edit variables
* Run playbook with envvar `OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES` on Mac to avoid
  following error:

```
TASK [Get client_id for BB authentication from AWS SSM] ***********************************************************************************************************************************************************************************
objc[91589]: +[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called.
objc[91589]: +[__NSCFConstantString initialize] may have been in progress in another thread when fork() was called. We cannot safely call it or ignore it in the fork() child process. Crashing instead. Set a breakpoint on objc_initializeAfterForkError to debug.
ERROR! A worker was found in a dead state
```

## How it works?

* Retrieve the SSM Secrets with the names `bb_client_id` and `bb_client_secret` for OAuth2
  authentication with BB from the organization's bastion account.
* Retrieve the SSM Secrets with the names `bb_user` and `bb_apitoken` for basic authentication
  authentication with BB from the organization's bastion account.
* The name of the SSM parameters should be:
  * `bb_client_id`
  * `bb_client_secret`
  * `bb_user`
  * `bb_apitoken`
* The config repository for the managed AWS organization should have a file named
  `sa_bb_config.yml` in the root directory.
* Repo config files can also live in `./include/**` in the BB
  config repository for the managed AWS organization. That allows you to have a single
  configuration file per managed BB repository. This requires a loop in
  `sa_bb_config.yml` to include files under `./include`
* Retrieve the `<ACCOUNT>_ACCESS_KEY_ID`, `<ACCOUNT>_SECRET_ACCESS_KEY` and
  `<ACCOUNT>_ACCOUNT_ID` from SSM parameter store, and use the values to
  populate the BB pipeline variable.
* When cloning `bb-aws-utils` and sourcing `lib/load.bash`, AWS credentials and config
  files will be created. The first account in the `service_account_list` list will
  also be the default AWS profile. To use any of the other profiles defined in
  `SA-ACCOUNT_LIST`, set and export `AWS_PROFILE` in your pipeline step.
* When `project_key` is not defined, the repo create/update task will be skipped
  to support old repo configs that do not have that property. This behaviour will be
  deprecated in a future version.

## Tags

The only supported tag is `rotate_credentials`.

When the Service Account credentials in the Bastion account are updated (this should be done regularly),
the BB repository Service Account variables should be updated as well. Otherwise, the pipelines will not
be able to assume the required permissions.

This tag limits the number or calls made to BB, because there is a limit of 1.000 calls per hour to the
repository API endpoints.

## The configuration file

| Property                      | Description                                                                                     |
|-------------------------------|-------------------------------------------------------------------------------------------------|  
| `aws_default_role`            | The role that will be assumed on the target account when using the Service Account credentials. |
| `aws_default_region`          | The default region for the AWS profiles                                                         |
| `bitbucket_username`          | The name of the BitBucket workspace to use                                                      |
| `bastion_account_id`          | The account ID of the Bastion account                                                           |
| `sts_role`                    | The `ServiceAccount` role to assume on the Bastion account                                      |
| `project_key`                 | The key of the project the repository should be assigned to                                     |
| `repos[]`                     | The list of repos to manage                                                                     |
|  * `<n>.group_permissions`    | Group permissions to add to the repo                                                            |
|  ** `<m>.group_slug`          | The Group Slug to grant permissions for                                                         |
|  ** `<m>.privilege`           | The privilege to grant, should be one of `read`, `write` or `admin`                             |
|  * `<n>.service_account_list` | List of dicts for the accounts for which to create BB pipeline variables                        |
|  ** `<m>.name`                | The name of the account, this will be used to retrieve the secrets from the SSM Parameter store |
|  ** `<m>.role_to_assume`      | The role to assume on the account, defaults to `cicd`                                           |
|  ** `<m>.state`               | `present` (default) or `absent`, create or remove the BB pipeline variables for this account    |
|  * `<n>.custom_vars`          | List of `name`/`value` dicts to create other BB pipeline variables                              |
|  ** `<m>.name`                | The name of the variable                                                                        |
|  ** `<m>.value`               | The value of the variable                                                                       |
|  ** `<m>.state`               | `present` (default) or `absent`, create or remove the BB pipeline variables for this account    |
|  ** `<m>.secure`              | Is this a secure variable (`yes`) (will not show in BB) or not (`no` - default)                 |

### An example `sa_bb_config.yml`

```yaml
aws_default_region: "eu-central-1"
# aws_default_role is the last part in arn:aws:iam::123456789012:role/ServiceAccount/cicd
aws_default_role: "cicd"
bitbucket_username: "workspace"
bastion_account_id: "123456789012"
repos:
{% for include_file in bb_repo_config_include_files | default([]) %}
{%   include 'include/' + include_file %}

{% endfor %}
```
Or:

```yaml
aws_default_region: "eu-central-1"
# aws_default_role is the last part in arn:aws:iam::123456789012:role/ServiceAccount/cicd
aws_default_role: "cicd"
bitbucket_username: "workspace"
repos:
  - name: "my-repo"
    project_key: "PROJ_KEY"
    group_permissions:
      - group_slug: "good-project-developers"
        privilege: "write"
      - group_slug: "bad-project-developers"
        privilege: "read"
      - group_slug: "project-admins"
        privilege: "admin"
    service_account_list:
      - name: "TOOLING"
      - name: "DEV"
      - name: "STG"
      - name: "PRD"
    custom_vars:
      - name: "A_VARIABLE"
        description: "A variable"
        value: "a value"
        secret: "false"
      - name: "A_SECRET_VARIABLE"
        description: "A secret variable"
        value: "a secret value"
        secret: "true"
  - name: "my-other-repo"
    project_key: "PROJ_KEY"
    ...
```
### An example `include/.../my-repo.yml`

```yml
  - name: "my-repo"
    project_key: "PROJ_KEY"
    group_permissions:
      - group_slug: "good-project-developers"
        privilege: "write"
      - group_slug: "bad-project-developers"
        privilege: "read"
      - group_slug: "project-admins"
        privilege: "admin"
    service_account_list:
      - name: "TOOLING"
      - name: "DEV"
      - name: "STG"
      - name: "PRD"
    custom_vars:
      - name: "A_VARIABLE"
        description: "A variable"
        value: "a value"
        secret: "false"
      - name: "A_SECRET_VARIABLE"
        description: "A secret variable"
        value: "a secret value"
        secret: "true"
```

## IMPORTANT

To remove variables or service accounts for a repo, do not remove the variable
configuration from the BB repo configuration files, as this will not make the
variables disappear in BB.

Instead, add the `state` property and set it to `absent`. If the state property
already exists and is `present`, change it to `absent`.

## Related repo's

* https://github.com/rik2803/aws-account-config
* https://github.com/rik2803/bb-aws-utils