# Concourse Bitbucket Pull Request Resource

Tracks pull requests made to a Bitbucket repository.
A status of pending, success, or failure will be set on the pull request, which must be explicitly defined in your pipeline.

Currently only basic username/password authentication offers full functionality.
Private key allows scanning for pull requests because this is pure git, but verifying merge status and setting status on pull requests is done through the Bitbucket REST Api for which SSL hasn't been tested yet.

> This resource was made for Bitbucket Server and will probably not work for Bitbucket Cloud.

## Installing

Use this resource by adding the following to the `resource_types` section of a pipeline config:

```yaml
---
resource_types:
- name: concourse-bitbucket-pullrequest
  type: docker-image
  source:
    repository: nikolovdragan/concourse-bitbucket-pullrequest-resource
```

See [concourse docs](http://concourse.ci/configuring-resource-types.html) for more details on adding `resource_types` to a pipeline config.

## Source Configuration

* `uri`: *Required.* The location of the repository.

* `private_key`: *Optional.* Private key to use when pulling/pushing.
    Example:
    ```
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
    ```

* `username`: *Optional.* Username for HTTP(S) auth when pulling/pushing.
  This is needed when only HTTP/HTTPS protocol for git is available (which does not support private key auth) and auth is required.

* `password`: *Optional.* Password for HTTP(S) auth when pulling/pushing.

* `token`: *Optional.* Token for HTTP(S) auth when pulling/pushing.
  If you have configured a [personal access token](https://confluence.atlassian.com/bitbucketserver/personal-access-tokens-939515499.html), you can use it instead of your username and password.

* `skip_ssl_verification`: *Optional.* Skips git ssl verification by exporting `GIT_SSL_NO_VERIFY=true`.

* `git_config`: *Optional*. If specified as (list of pairs `name` and `value`) it will configure git global options, setting each name with each value.

  This can be useful to set options like `credential.helper` or similar.

  See the [`git-config(1)` manual page](https://www.kernel.org/pub/software/scm/git/docs/git-config.html)
  for more information and documentation of existing git options.

* `only_for_branch`: *Optional.* If specified only pull requests which target those branches will be considered.
It will accept a regular expression as determined by [egrep](http://linuxcommand.org/man_pages/egrep1.html).

* `only_without_conflicts`: *Optional (default: true).* If enabled only pull requests which are not in a conflicted state will be built.

* `only_when_mergeable`: *Optional (default: false).* If enabled only pull requests which are mergeable (all tasks done, required number of approvers reached, ...) will be built.

* `rebuild_phrase`: *Optional (default: test this please).* Regular expression as determined by [egrep](http://linuxcommand.org/man_pages/egrep1.html) will match all comments in pull request overview.
If a match is found the pull request will be rebuilt.

* `debug`: *Optional (default: false).* If set to `true` the bash tracing (`set -x`) will be enabled for the `in`, `out` and `check` scripts.

## Behavior

### `check`: Search for pull requests to build.

Check will return a version for every pull request that matches the criteria defined in source configuration.

### `in`: Clone the repository, at the given pull request merge ref

Clones the repository to the destination, and locks it down to a given ref.

** IMPORTANT **
It is essential that you set the [version](https://concourse.ci/get-step.html#get-version) to `every` on the get step of your job configuration.
It will allow you to build all versions instead of only the latest.

Submodules are initialized and updated recursively.

Note: the name of the branch from which the pull request has been created is stored in the special git config `pullrequest.branch` so that you can use it as reference in your pipeline.

#### Parameters

* `depth`: *Optional.* If a positive integer is given, *shallow* clone the repository using the `--depth` option. Using this flag voids your warranty.
  Some things will stop working unless we have the entire history.

* `submodules`: *Optional.* If `none`, submodules will not be fetched. If specified as a list of paths, only the given paths will be fetched. If not specified, or if `all` is explicitly specified, all submodules are fetched.

* `disable_git_lfs`: *Optional.* If `true`, will not fetch Git LFS files.

#### GPG signature verification

If `commit_verification_keys` or `commit_verification_key_ids` is specified in the source configuration, it will additionally verify that the resulting commit has been GPG signed by one of the specified keys. It will error if this is not the case.

### `out`: Update the status of a pull request

Set the status message on specified pull request.

#### Parameters

* `path`: *Required.* The path of the repository to reference the pull request.

* `status`: *Required.* The status of success, failure or pending.

  * [`on_success`](https://concourse.ci/on-success-step.html) and [`on_failure`](https://concourse.ci/on-failure-step.html) triggers may be useful for you when you wanted to reflect build result to the pull request (see the example below).

* `comment`: *Optional.* A custom comment that you want added to the status message.
  Any occurence of `[[BRANCH]]` will be replace by the actual branch name form the
  pull request.

* `commentFile`: *Optional.* The path to a file that contains a custom comment to
  add to the message. This allow the comment to be built by a previous task in the job.

## Example pipeline

```yaml
resource_types:
- name: concourse-bitbucket-pullrequest
  type: docker-image
  source:
    repository: emeraldsquad/concourse-bitbucket-pullrequest-resource

resources:
- name: pullrequest
  type: concourse-bitbucket-pullrequest
  source:
    username: {{bitbucket-username}}
    password: {{bitbucket-password}}
    uri: https://bitbucket.exemple.net/scm/exemple-project/exemple-repo.git
    debug: true

jobs:
- name: test pull request
  plan:
  - get: pullrequest
    trigger: true
    version: every
  - put: pullrequest
    params:
      path: pullrequest
      status: pending
  - task: test
    config:
      platform: linux

      inputs:
      - name: pullrequest

      ...

    on_success:
      put: pullrequest
      params:
        path: pullrequest
        status: success
    on_failure:
      put: pullrequest
      params:
        path: pullrequest
        status: failure
```
