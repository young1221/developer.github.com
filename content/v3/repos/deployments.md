---
title: Deployments | GitHub API
---

# Deployments

* TOC
{:toc}

<div class="alert">
  <p>
    The Deployments API is currently available for developers to preview.
    During the preview period, the API may change without advance notice.
    Please see the <a href="/changes/2014-01-09-preview-the-new-deployments-api/">blog post</a> for full details.
  </p>

  <p>
    To access the API during the preview period, you must provide a custom <a href="/v3/media">media type</a> in the <code>Accept</code> header:
    <pre>application/vnd.github.cannonball-preview+json</pre>
  </p>
</div>

Deployments are a request for a specific ref(branch,sha,tag) to be deployed.
GitHub then dispatches deployment events that external services can listen for
and act on. This enables developers and organizations to build loosely-coupled
tooling around deployments, without having to worry about implementation
details of delivering different types of applications (e.g., web, native).

Deployment Statuses allow external services to mark deployments with a
'success', 'failure', 'error', or 'pending' state, which can then be consumed
by any system listening for `deployment_status` events.

Deployment Statuses can also include an optional `description` and `target_url`, and
we highly recommend providing them as they make deployment statuses much more
useful. The `target_url` would be the full URL to the deployment output, and
the `description` would be the high level summary of what happened with the
deployment.

Deployments and Deployment Statuses both have associated
[repository events](/v3/activity/events/types/#deploymentevent) when
they're created. This allows webhooks and 3rd party integrations to respond to
deployment requests as well as update the status of a deployment as progress is
made.

Below is a simple sequence diagram for how these interactions would work.

<pre>
+---------+             +--------+            +-----------+        +-------------+
| Tooling |             | GitHub |            | 3rd Party |        | Your Server |
+---------+             +--------+            +-----------+        +-------------+
     |                      |                       |                     |
     |  Create Deployment   |                       |                     |
     |--------------------->|                       |                     |
     |                      |                       |                     |
     |  Deployment Created  |                       |                     |
     |<---------------------|                       |                     |
     |                      |                       |                     |
     |                      |   Deployment Event    |                     |
     |                      |---------------------->|                     |
     |                      |                       |     SSH+Deploys     |
     |                      |                       |-------------------->|
     |                      |                       |                     |
     |                      |   Deployment Status   |                     |
     |                      |<----------------------|                     |
     |                      |                       |                     |
     |                      |                       |   Deploy Completed  |
     |                      |                       |<--------------------|
     |                      |                       |                     |
     |                      |   Deployment Status   |                     |
     |                      |<----------------------|                     |
     |                      |                       |                     |
</pre>

Keep in mind that GitHub is never actually accessing your servers. It's up to
your 3rd party integration to interact with deployment events.
This allows for [github-services](https://github.com/github/github-services)
integrations as well as running your own systems depending on your use case.
Multiple systems can listen for deployment events, and it's up to each of
those systems to decide whether or not they're responsible for pushing the code
out to your servers, building native code, etc.

Note that the `repo_deployment` [OAuth scope](/v3/oauth/#scopes) grants
targeted access to Deployments and Deployment Statuses **without**
granting access to repository code, while the `repo` scope grants permission to code
as well.

## List Deployments

Users with pull access can view deployments for a repository:

    GET /repos/:owner/:repo/deployments

### Response

<%= headers 200, :pagination => default_pagination_rels %>
<%= json(:deployment) { |h| [h] } %>

## Create a Deployment

If your repository is taking advantage of [commit statuses](/v3/repos/statuses),
the API will reject requests that do not have a successful [combined
status.](/v3/repos/statuses/#get-the-combined-status-for-a-specific-ref) (Your
repository is not required to use commit statuses. If no commit statuses are
present, the deployment will always be created.)

The `force` parameter can be used when you really just need a deployment to go
out. In these cases, all checks are bypassed, and the deployment is created for
the ref.

The `auto_merge` parameter is used to ensure that the requested ref is not
behind the repository's default branch. If the ref *is* behind the default
branch for the repository, we will attempt to merge it for you. If the merge
succeeds, the API will return a successful merge commit. If merge conflicts
prevent the merge from succeeding, the API will return a failure response.

The `payload` parameter is available for any extra information that a
deployment system might need. It is a JSON text field that will be passed on
when a deployment event is dispatched.

Users with push access can create a deployment for a given ref:

    POST /repos/:owner/:repo/deployments

### Parameters

Name | Type | Description
-----|------|--------------
`ref`|`string`| **Required**. The ref to deploy. This can be a branch, tag, or sha.
`force`|`boolean`| Optional parameter to bypass any ahead/behind checks or commit status checks. Default: `false`
`payload`|`string` | Optional JSON payload with extra information about the deployment. Default: `""`
`auto_merge`|`boolean`| Optional parameter to merge the default branch into the requested deployment branch if necessary. Default: `false`
`description`|`string` | Optional short description. Default: `""`

#### Example

<%= json \
  :ref           => "topic-branch",
  :payload       => "{\"environment\":\"production\",\"deploy_user\":\"atmos\",\"room_id\":123456}",
  :description   => "Deploying my sweet branch"
%>

<%= headers 201,
      :Location =>
'https://api.github.com/repos/octocat/example/deployments/1' %>
<%= json :deployment %>

## Update a Deployment

Once a deployment is created, it cannot be updated. Information relating to the
success or failure of a deployment is handled through Deployment Statuses.

# Deployment Statuses

## List Deployment Statuses

Users with pull access can view deployment statuses for a deployment:

    GET /repos/:owner/:repo/deployments/:id/statuses

### Parameters

Name | Type | Description
-----|------|--------------
`id` |`integer`| **Required**. The Deployment ID to list the statuses from.


### Response

<%= headers 200, :pagination => default_pagination_rels %>
<%= json(:deployment_status) { |h| [h] } %>

## Create a Deployment Status

Users with push access can create deployment statuses for a given deployment:

    POST /repos/:owner/:repo/deployments/:id/statuses

### Parameters

Name | Type | Description
-----|------|--------------
`state`|`string` | **Required**. The state of the status. Can be one of `pending`, `success`, `error`, or `failure`.
`target_url`|`string` | The target URL to associate with this status.  This URL should contain output to keep the user updated while the task is running or serve as historical information for what happened in the deployment. Default: `""`
`description`|`string` | A short description of the status. Default: `""`

#### Example

<%= json \
  :state       => "success",
  :target_url  => "https://example.com/deployment/42/output",
  :description => "Deployment finished successfully."
%>

### Response

<%= headers 201,
      :Location =>
'https://api.github.com/repos/octocat/example/deployments/42/statuses/1' %>
<%= json :deployment_status %>
