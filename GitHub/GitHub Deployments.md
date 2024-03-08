---
title: GitHub Deployments
publish: true
topics:
  - GitHub-API
  - GitHub-Webhook
description: GitHub의 Deployments
---
#### Deployments
- Deployments are requests to deploy a specific ref (branch, SHA, tag)
- GitHub dispatches a `deployment` event that external services can listen for and act on when new deployments are created
- Deployments enable developers and organizations to build loosely coupled tooling around deployments, without having to worry about the implementation details of delivering different types of applications (e.g., web, native).
#### Deployment statuses
- Deployment statuses allow external services to mark deployments with an `error`, `failure`, `pending`, `in_progress`, `queued`, or `success` state that systems listening to `deployment_status` events can consume.
- Deployment statuses can also include an optional `description` and `log_url`
	- `log_url` is the full URL to the deployment output
	- `description` is a high-level summary of what happened with the deployment
#### Interactions
- GitHub dispatches `deployment` and `deployment_status` events when new deployments and deployment statuses are created.
	- Multiple systems can listen for deployment events
	- Each system can decide whether it is responsible for pushing the code to your servers or building native code.
- These events allow third-party integrations to receive and response to deployment requests, and update the status of a deployment as progress is made.
#### Inactive deployments
- When you set the status of one deployment to `success`, the status of other deployments with the same environment in the same repository changes to `inactive`
- Setting the `auto_inactive` to `false` can prevent this from happening
- Setting the `state` to `inactive`, GitHub displays the deployment as `destroyed` and removes access to it