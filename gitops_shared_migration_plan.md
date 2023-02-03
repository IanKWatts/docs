# Migration to shared instance of Argo CD in GitOps Operator

The existing "shared" instance of Argo CD in the 'argocd-shared' namespace of each cluster will be replaced with an instance of the GitOps Operator, which provides essentially the same function, but requires a different system of management.  The CCM role 'argocd' is being replaced by the 'gitops' role.  The resources in the 'argocd-shared' resources will remain in place for the time being.

Updates have been merged in CCM in PR 602.

https://github.com/bcgov-c/platform-gitops-gen/pull/602

Upgrade steps are below.

## Notice to users (automation specialist)
Post a message to the `#devops-argocd` channel in Rocketchat alerting users to the upcoming change.

## Run installation playbook (cluster admins)
Prior to the CCM push for a given cluster, someone with cluster admin privileges will run the gitops install playbook.

Set ENV vars for:
* ARGOCD_OIDC_CLIENT_SECRET: Base64-encoded password of the OIDC client
* ARGOCD_GITHUB_DEPLOY_KEY:  Base64-encoded private SSH key for GitHub access
* SEALED_SECRETS_CERT:       Path to the cluster's certificate file
`oc login` to cluster 
`ansible-playbook -i inventory/clustername -e do_gitops=false playbooks/gitops/install.yaml`

## CCM push (cluster admins)
* patcher job will run at end of sync of 'gitops-extra-shared'
* includes warden operator updates for image and ARGOCD_NAMESPACE

## Manual setup (automation specialist)
* Add SSH keys for github.com
* Add repository template for bcgov-c

## Run script to export Applications from argocd-shared (automation specialist)
* Run a script to export all Applications from argocd-shared
* Clean manifests and change namespace from argocd-shared to openshift-bcgov-gitops-shared

## Create Applications in openshift-bcgov-gitops-shared (automation specialist)
* Apply updated manifests to openshift-bcgov-gitops-shared
* Ensure that all apps are synced

## Update Route (automation specialist)
* Remove 'argocd-shared' route from argocd-shared namespace
* Add 'argocd-shared' route in openshift-bcgov-gitops-shared

## Notice to users (automation specialist)
* Post a message to users informing them that the migration is complete and to check their access and apps.

