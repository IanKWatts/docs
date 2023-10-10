# GitHub Apps

_"GitHub Apps are tools that extend GitHub's functionality. GitHub Apps can do things on GitHub like open issues, comment on pull requests, and manage projects. They can also do things outside of GitHub based on events that happen on GitHub. For example, a GitHub App can post on Slack when an issue is opened on GitHub."_

https://docs.github.com/en/apps/using-github-apps/about-using-github-apps

This document describes the apps that we use as well as how to install and configure a GitHub App.

## Table of Contents
1. [GitHub Apps in BC Gov GitHub Organizations](#github-apps-list)
   1. [policy-bot](#policy-bot)
   2. [bulldozer](#bulldozer)
4. [Installation of the Application](#installation)
5. [Creation of the GitHub App](#creation)

## GitHub Apps in BC Gov GitHub Organizations<a name="github-apps-list"></a>
The GitHub Apps in use are:
* policy-bot
* bulldozer
* GitOpsCI

policy-bot and bulldozer are both produced by Palantir Technologies and complement each other.

GitOpsCI?  Need more info

### policy-bot<a name="policy-bot"></a>
https://github.com/palantir/policy-bot

policy-bot is a GitHub App for enforcing approval policies on pull requests. It does this by creating a status check, which can be configured as a required status check.

While GitHub natively supports required reviews, policy-bot provides more complex approval features:
* Require reviews from specific users, organizations, or teams
* Apply rules based on the files, authors, or branches involved in a pull request
* Combine multiple approval rules with and/or conditions
* Automatically approve pull requests that meet specific conditions

Behavior is configured by a file in each repository. policy-bot also provides a UI to view the detailed approval status of any pull request.

#### Configuration
Each repository configured for policy-bot contains a **.policy.yml** file in the top-level directory.  This file contains a list of rules and a list of policies that define how to apply those rules.

##### Approval rules
Typical usage in the BC Gov cloud environments is to automatically approve pull requests that have the **auto-merge** label.  The following rule matches when the 'auto-merge' label has been added to the PR.  The 'count' setting is zero, because that refers to how many authorized users must approve the PR.  Because we don't want users to have to manually approve it, 'count' is set to zero.
```
approval_rules:
  - name: auto-merge
    description: automatic merge for create and edit requests
    if:
      has_labels:
        - "auto-merge"
    requires:
      count: 0
```

Another rule that we use is to require the manual approval of the PR by certain team members.  This is an important safety check to ensure that a single user does not create and merge their own PR, which may contain a typo or other error.  Sometimes a second or third set of eyes should review the PR before it is merged.
```
approval_rules:
- name: one reviewer
  options:
    allow_contributor: true
  requires:
    count: 1
    users:
    - "<GithubUser1>"
    - "<GithubUser2>"
    - "<GithubUser3>"
    - "<GithubUser4>"
```
Because count is set to one, any one team member may approve the PR.  With 'allow_contributor' set to true, a user that has committed to the PR or created the PR is allowed to approve it.  This is helpful when all of the approved users have contributed to a PR.

These rules can be used together so that if the PR has the 'auto-merge' label, it will be automatically merged once it has been approved by at least one reviewer.

For more information, see the documentation in the policy-bot repo.

https://github.com/palantir/policy-bot/blob/develop/README.md

##### Approval policies
We use an 'approval' policy to define how the approval rules are applied.  To allow the merging of the PR when any one of our two example rules are satisfied, we use an 'or' condition.
```
policy:
  approval:
    - or:
      - one reviewer
      - auto-merge
```
The values in the 'or' list match the 'name' field of the approval rule.

When there is just a single approval rule, the 'or' condition is removed.
```
policy:
  approval:
    - one reviewer
```

For more information, see the documentation in the policy-bot repo.

https://github.com/palantir/policy-bot/blob/develop/README.md


### bulldozer<a name="bulldozer"></a>
While policy-bot defines the rules for the merging of PRs, bulldozer does the actual merge once the conditions are met.  Remember that policy-bot sets checks on a PR and if those checks have not passed, GitHub will not merge it.  So those conditions must be satisfied, as well as the conditions set in bulldozer.

Consider the following bulldozer configuration file:
```
merge:
  trigger:
    labels: ["auto-merge", "requires-approval"]
  method: squash
  required_statuses:
    - "BC-Ruler-ent: main"
  delete_after_merge: true
```
In order to act on this PR:
* It must have a label of "auto-merge" or "requires-approval".
* It will 'squash' multiple commits into a single commit.
* The PR must have passed the checks associated with "BC-Ruler-ent: main" (the policy-bot rules)
* bulldozer will delete the source branch after the merge.

Note that the value in the 'required_statuses' list corresponds to the name of the GitHub App as installed in this org.

For more information, see the documentation in the bulldozer repo.

https://github.com/palantir/bulldozer/blob/develop/README.md


## Installation of the Application<a name="installation"></a>
Behind the GitHub App in the GitHub org, there is an actual application deployed that can be reached by the GitHub servers.  This application should be deployed before that configuration can be done.  There will be a little bit of configuration done to the deployment after the GitHub App is created.

The deployment configuration will depend on the nature of the app, but in the case of policy-bot and bulldozer we deploy the Docker image of the app, as well as a Service, Route, and Secret.  The Service and Route allow GitHub to reach the app.  The Secret is used to set info related to the GitHub App, such as its "integration ID" and private key.

**To Do: set up a gitops repo for argocd to manage these**


## Creation of the GitHub App<a name="creation"></a>
A "GitHub App" is a configuration within a GitHub organization, not be confused with the actual application, which is running on our cluster.  After the actual application has been deployed, you can create the GitHub App.  You must be an owner or admin of the org.

Start at the GitHub org page.

Click Settings --> Developer settings --> GitHub Apps --> New GitHub App

| Field | Input |
|-------|-------|
| GitHub App name | Enter a unique name for each GitHub App |
| | Enter a description of the app |
| Homepage URL | Copy the URL from the Route |
| Callback URL | Leave blank |
| Setup URL | Leave blank |
| Webhook URL | For policy-bot and bulldozer, it is the URL from the Route + /api/github/hook (may be different for other apps) |
| Webhook secret | see https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries |
| Repository permissions | Select only what is needed |
| Organization permissions | Probably none; what does the app do? |
| Account permissions | Probably none; what does the app do? |
| Subscribe to events | Review the list; see bulldozer as an example |

If you are creating an app in a new org, or otherwise creating an app that already exists in another org, you could copy some of these settings from the other org, especially the three 'permissions' blocks.  The URL is unique for each org.

Click 'Create GitHub App' to complete the process.

After creation, a banner message is shown, which contains a link to generate a private key.  Click the link, then Generate Private Key.  The key will be downloaded to your browser.  Save it and add it to the team's Vault account.  You must also make a record of the Webhook secret.

**You're not done yet!** On the app settings page, click 'Install App' and click Install for the org.
* Select "Installation only for select repositories"
* Select the repositories
* Click 'Install'

**Secrets**

In the case of policy-bot and bulldozer, each Deployment references the webhook secret and github private key from a Secret in the namespace where the app runs. After creating the GitHub App, you'll have to go back to OpenShift and update the relevant Secret with those values and restart the app.










