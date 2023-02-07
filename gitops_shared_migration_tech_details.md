# Migration of shared Argo CD to GitOps Operator
These are the technical details for the migration to the GitOps Operator for the shared instance of Argo CD.
This doc is a companion to the more high-level <a href="gitops_shared_migration_plan.md">migration plan</a>.

## CCM pre-migration preparation
The CCM release should include an update enabling the setup of the shared instance of the GitOps Operator.
In the CCM repo, update inventory/group_vars/all/all.yaml settings gitops_shared to true.
`gitops_shared: true`

## Verify installation
After the 'gitops' install playbook has been run, check the cluster for the new namespace and resources, which should all be up and running.
* Namespace: openshift-bcgov-gitops-shared
* Deployments: 
  - gitops-shared-notifications-controller
  - gitops-shared-redis
  - gitops-shared-repo-server
  - gitops-shared-server
* StatefulSets:
  - gitops-shared-application-controller
* The Secret 'argocd-secret' should contain an entry for 'oidc.sso.clientSecret'
* The ArgoCD UI should be available at the "gitops-shared" URL, e.g.
  - https://gitops-shared.apps.silver.devops.gov.bc.ca/applications

## Add SSH keys for github.com
The default keystore includes one SSH key for github.com, but we need to add others in order to be able to connect.
* Log in to Argo CD at the new gitops-shared URL
* Go to Settings --> Certificates
* Click the button 'Add SSH Known Hosts'
* In a separate terminal window, run `ssh-keyscan github.com`
* Copy the output and paste it into the form field, then click 'Create'

Example output of `ssh-keyscan`
```
# github.com:22 SSH-2.0-babeld-662c2099
github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl
# github.com:22 SSH-2.0-babeld-662c2099
github.com ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAq2A7hRGmdnm9tUDbO9IDSwBK6TbQa+PXYPCPy6rbTrTtw7PHkccKrpp0yVhp5HdEIcKr6pLlVDBfOLX9QUsyCOV0wzfjIJNlGEYsdlLJizHhbn2mUjvSAHQqZETYP81eFzLQNnPHt4EVVUh7VfDESU84KezmD5QlWpXLmvU31/yMf+Se8xhHTvKSCZIFImWwoG6mbUoWf9nzpIoaSjB+weqqUUmpaaasXVal72J+UX2B+2RPW3RcT0eOzQgqlJL3RKrTJvdsjE3JEAvGq3lGHSZXy28G3skua2SmVi/w4yCE6gbODqnTWlg7+wC604ydGXA8VJiS5ap43JXiUFFAaQ==
# github.com:22 SSH-2.0-babeld-662c2099
github.com ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEmKSENjQEezOmxkZMy7opKgwFB9nkt5YRrYMjNuG5N87uRgg6CLrbo5wAdT/y6v0mKV0U2w0WZ2YB/++Tpockg=
```

## Add repository connection template
* In the Argo CD UI, go to Settings --> Repositories
* Click the button 'Connect Repo Using SSH'
  - Name: GitOps
  - Project: default
  - Repository URL: git@github.com:bcgov-c/
  - SSH private key data: copy from team vault account, argocd --> gitops-shared-github-key
  - Click 'Save as Credentials Template'

## Verify Warden Operator update
The CCM push includes a configuration change for the Warden Operator, setting ARGOCD_NAMESPACE to 'openshift-bcgov-gitops-shared'.  Verify that this parameter is set:
* Namespace: openshift-bcgov-warden-operator
* Deployment: warden-operator-controller-manager
The latest build of the Warden Operator includes a change to apply a label to any namespaces used by Argo CD projects, based on the presence of a `GitOpsTeam` or `GitOpsAlliance` resource.

## Export Applications
* Run a shell script to create manifest files for each Application (see below for script)
* Update the namespace in each manifest
  - `perl -pi -e 's|namespace: argocd-shared|namespace: openshift-bcgov-gitops-shared|' *.yaml`
* Apply the modified manifests to the new openshift-bcgov-gitops-shared namespace
  - `oc -n openshift-bcgov-gitops-shared apply -f *.yaml`

## Notify users
Post a message in the `#devops-argocd` channel informing users that the migration is taking place in this cluster.

## Change route
* Verify new Applications via new route, gitops-shared.apps...
* Remove argocd-shared route from argocd-shared
* Manually add argocd-shared route in openshift-bcgov-gitops-shared

## Shut down argocd-shared
* Scale down all deployments
* Update cluster gitops repo directly to disable auto-sync for argocd-shared

## Follow-up
Add argocd-shared Route to gitops-shared in CCM

## To Do
* Add to automation:
  - add SSH known hosts
  - add repo template

## Appendix A - initial testing
* Create Project in CLAB argocd-shared
* Create 2 Applications in project in CLAB argocd-shared
  - synced and healthy
* Delete existing ArgoCD instance gitops-shared
* Manually generate and apply manifests for gitops-shared
* Run script to export Applications
  - then update namespace from argocd-shared --> openshift-bcgov-gitops-shared
	perl -pi -e 's|namespace: argocd-shared|namespace: openshift-bcgov-gitops-shared|' cluster/*.yaml
* Update Warden Operator with new namespace and version
  - In Deployment warden-operator-controller-manager as an ENV var
* Check that Project is created
* Manually apply Application manifests
* Verify that Applications are synced and healthy

## Appendix B - shell script to export Applications
```
#!/bin/sh
#
# migrate_applications.sh
#
# Migrate Applications from 'argocd-shared' to 'openshift-bcgov-gitops-shared'
# ----------------------------------------------------------------------------

# Check OCP login and project
# ---------------------------
unset GET_RESOURCES_PROJECT
GET_RESOURCES_PROJECT=`oc project`
if [ -z "$GET_RESOURCES_PROJECT" ]; then
  echo ""
  echo "Please log in to OpenShift and select your namespace."
  echo ""
  exit 1
else
  echo ""
  echo $GET_RESOURCES_PROJECT
  echo ""
fi

CLUSTER=`echo $GET_RESOURCES_PROJECT | perl -p -e 's|^.*api\.([a-z]+)\.devops.*|$1|'`
echo "Cluster: ${CLUSTER}"
echo "Press ENTER to continue."
read INPUT

mkdir -p $CLUSTER
cd $CLUSTER

# Get a list of Applications
# --------------------------
oc -n argocd-shared get applications --no-headers=true | awk '{ print $1 }' > list.txt

# Generate YAML manifests, removing auto-generated fields, status, etc.
# ---------------------------------------------------------------------
for app in `cat list.txt`; do
  MANIFEST_FILE="${app}.yaml"
  echo $MANIFEST_FILE
  oc -n argocd-shared get application $app -o json | jq '
if .metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"] != null then del(.metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]) else . end |
if .metadata.creationTimestamp != null then del(.metadata.creationTimestamp) else . end |
if .metadata.generation != null then del(.metadata.generation) else . end |
if .metadata.managedFields != null then del(.metadata.managedFields) else . end |
if .metadata.resourceVersion != null then del(.metadata.resourceVersion) else . end |
if .metadata.selfLink != null then del(.metadata.selfLink) else . end |
if .metadata.uid != null then del(.metadata.uid) else . end |
if .spec.clusterIP != null then del(.spec.clusterIP) else . end |
if .status != null then del(.status) else . end' | yq -P eval > $MANIFEST_FILE
done
```

