# Jenkins Webhook Proxy

The webhook proxy service allows to trigger Jenkins pipelines. Further, it
automatically creates pipelines that do not exist yet and can delete pipelines
that are no longer needed.

One instance of the webhook proxy runs in every `<project>-cd` namespace next to
the Jenkins instance.

## Endpoints

### `POST /`
Accepts webhooks from BitBucket and forwards them to the corresponding Jenkins
pipeline (which is determined based on the component param and the branch name).
If there is no corresponding pipeline yet, it will be created on the fly (by
creating a `BuildConfig` in OpenShift which is synced to Jenkins via the
OpenShift plugin). Once a branch is deleted or a pull request declined/merged,
the corresponding Jenkins pipeline is deleted.

### `POST /build`
Accepts a payload of the following form:
```
{
    "branch": "foo",
    "repository": "repository",
    "env": [
       {
          "name": "FOO_BAR",
          "value": "baz"
       }
    ]
}
```

**Important**: In order to avoid conflicts between pipelines created/triggered
via BitBucket and pipelines created/triggered via `/build`, most likely you'd
want to pass a component name to `/build`, like so: `/build?component=foo`, see
the next section.


### Parameters
Both `/` and `/build` accept the following query parameters. They are offered
as query parameters only because otherwise they could not be adjusted for
BitBucket webhooks.

| Variable | Description |
| --- | --- |
| jenkinsfile_path | The path to the `Jenkinsfile`. By default, the `Jenkinsfile` is assumed to be in the root of the repository, therefore this value defaults to simply `Jenkinsfile`. |
| component | The component part of the pipeline name. If not given, the pipeline name is created from the repository and the branch. |


## Adding a webhook in BitBucket

The provisioning app sets up one webhook per repository by default. It is
possible to create webhooks manually as well, e.g. to add more than one
webhook (likely differentiated by the `component` param then). To manually
create a webhook, go to "Repository Settings > Webhooks" and click on
"Create webhook". Enter e.g. `Jenkins` as title and the route to the webhook
proxy instance as URL. Under "Repository events", select `Push`. Under
"Pull request events", select `Merged` and `Declined`. Save your changes.



## Customizing the behaviour of the webhook proxy

The following environment variables are read by the proxy:

| Variable | Description | Optional |
| --- | --- | --- |
| PROTECTED_BRANCHES | Comma-separated list of branches which pipelines should not be cleaned up. Use either exact branch names, branch prefixes (e.g. `feature/`) or `*` for all branches. Defaults to: `master,develop,production,staging,release/`. | Yes |
| OPENSHIFT_API_HOST | Defaults to `openshift.default.svc.cluster.local`. Usually does not need to be modified. | Yes |
| REPO_BASE | The base URL of the repository (e.g. your BitBucket host). This variable is set by the template and usually does not need to be modified. | No |
| TRIGGER_SECRET | The secret which protects the pipeline to be executed from outside. This variable is set by the template and usually does not need to be modified. | Yes |
| NAMESPACE_FILE | Location of the file containing the OpenShift namespace. Defaults to: `/var/run/secrets/kubernetes.io/serviceaccount/namespace`. | yes |
| TOKEN_FILE | Location of the file containing the OpenShift access token. Defaults to: `/var/run/secrets/kubernetes.io/serviceaccount/token`. | yes |
| CA_CERT_FILE | Location of the file containing the OpenShift instance CA cert. Defaults to: `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`. | yes |
| SERVER_PORT | Server port. Defaults to: 8080. | yes |


## Development

See the `Makefile` targets.
