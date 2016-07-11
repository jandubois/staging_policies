## Proof of concept: Staging policies and decorators ##

The main project description/discussion is in this [Google doc](https://docs.google.com/document/d/1J9Iw309UwzGC4zoxSp1rcS47XxlfrrMkYxBpiDeKfYc).

A quick&dirty PoC has been implemented in these 4 commits:

* https://github.com/jandubois/cloud_controller_ng/commit/6fb0145
* https://github.com/jandubois/buildpack_app_lifecycle/commit/22e6e1c
* https://github.com/jandubois/runtime-schema/commit/e453b10
* https://github.com/jandubois/stager/commit/dfe8881

They are based on `cf-release` v237 and `diego-release` v0.1472. They can probably be cherry-picked to the latest; I went back to 237 because of some problems that turned out to be my own mistakes.

One caveat is that you have to checkout buildpack_app_lifecycle in *both* locations to the same commit:

```
~/workspace/cf-release/src/capi-release/src/github.com/cloudfoundry-incubator/buildpack_app_lifecycle
~/workspace/diego-release/src/github.com/cloudfoundry-incubator/buildpack_app_lifecycle
```

The submodule in `cf-release` is used by the stager to construct the builder commandline for the staging task request. The submodule in `diego-release` is used to build the tarball of the lifecycle components. So they are tightly coupled.

This PoC assumes that any buildpack whose name ends with `_policy` is a policy/decorator.

There are both a policy and a decorator sample in the [staging_repositories](https://github.com/jandubois/staging_policies) repo.

```
$ git clone git@github.com:jandubois/staging_policies.git
$ cf create-buildpack timestamp_policy staging_policies/timestamp_policy 9999
$ cf create-buildpack php_policy staging_policies/php_policy 9999
```

The [timestamp_policy](https://github.com/jandubois/staging_policies/blob/master/timestamp_policy/bin/compile) prints the current date/time to `stdout` and also sets a `TIMESTAMP` environment variable in the appication itself (via `~/.profile.d`).

The [php_policy](https://github.com/jandubois/staging_policies/blob/master/php_policy/bin/compile) rejects any droplet that includes a `*.php` file.

Here is a sample push of a node app (edited for brevity). You can see the output of the timestamp policy, and how the droplet gets rejected by the presence of the `*.php` files in the punycode NPM module installed by the nodejs_buildpack:

```bash
$ cf push
[...]
-----> Build succeeded!

Timestamp: Mon Jul 11 17:42:16 UTC 2016
We have a zero tolerance policy about PHP here:
/tmp/app/.heroku/node/lib/node_modules/npm/node_modules/request/node_modules/tough-cookie/node_modules/punycode/vendor/docdown/doc/parse.php
/tmp/app/.heroku/node/lib/node_modules/npm/node_modules/request/node_modules/tough-cookie/node_modules/punycode/vendor/docdown/docdown.php
Droplet has been rejected by policy: Blocked by policy
Exit status 225
Staging failed: Exited with status 225

FAILED
Error restarting application: StagingError
```

Disabling the php_policy lets us deploy the app, and checking the app output shows that the `TIMESTAMP` variable is visible to the app:

```bash
$ cf update-buildpack php_policy --disable
Updating buildpack php_policy...
OK

$ cf push
[...]
-----> Build succeeded!

Timestamp: Mon Jul 11 18:01:07 UTC 2016
Exit status 0
Staging complete
Uploading droplet, build artifacts cache...
[...]
$ curl -s node-env.bosh-lite.com | html2text -nobs | grep -A1 TIME
TIMESTAMP
Mon Jul 11 18:01:07 UTC 2016
```
