# Security
There are numerous features that make the OpenShift platform more secure than most traditional hosting environments, but several areas require extra attention to ensure a secure application.  These include:
* [Network Policies](#network-policies)
* [RoleBindings](#rolebindings)
* [Vault](#Vault)


## Network Policies
By default, your namespaces are created with a "deny all" NetworkPolicy called 'platform-services-controlled-default', which will block all traffic from reaching your services, even within your own namespace.  This is a secure default configuration and you will have to create other network policies in order for your application to be usable.
* When creating network policies, **do not create an "allow all" policy.**
* Allow only specific traffic that is needed.
* For more information, see [OpenShift network policies](https://docs.developer.gov.bc.ca/openshift-network-policies/)

Review your network policies.  Are they overly permissive?  Do you allow inbound traffic from the Internet to services other than those that are meant for public access?

Note that there is a production cluster, Emerald, that uses the NSX software-defined network (SDN), as opposed to the default OpenShift SDN, as the other clusters use.  This allows for a much more fine-grained configuration, including full egress and ingress controls.  If your application has greater network security requirements, consider using the Emerald cluster.  Contact the Platform Services team if you have questions about this.

## RoleBindings
As with network policies, RoleBindings in your namespaces should be as minimal as possible.  By default, all contacts that are associated with the product in the platform's Registry will be given administrative access to all four namespaces.  These namespace admins can create other RoleBindings as needed to support your application.

* Only grant namespace access to those who need it.
* Grant minimal permissions.  How much access do your users need?
    * There is a 'view' ClusterRole for read-only access.
    * For special cases, like ServiceAccounts, create a Role and RoleBinding to grant it only the permissions that it needs to fulfill its function.
* Create separate RoleBindings for users with differing access needs.  That is, if User A requires 'edit' access and User B requires only 'view' access, create separate RoleBindings rather than give 'edit' access to both.
* Periodically review the RoleBindings in your namespaces.  As your team changes, remove access for those that leave.

For service accounts, be sure to grant access only to accounts in your own namespace.  It is possible to mistakenly grant access to all accounts in the cluster!

A good RoleBinding for service accounts, if granting permission to all of your namespace's service accounts, would have a 'subjects' section that includes your namespace name in the 'name' line:
```
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: 'system:serviceaccounts:abc123-prod'
```
If the 'name' line included only 'system:serviceaccounts', then all service accounts on cluster would be given access!

For more information, see [Grant user access in OpenShift](https://docs.developer.gov.bc.ca/grant-user-access-openshift/).

## Vault
Vault uses on-disk encryption, so even if a malicious actor somehow got a copy of the contents of the filesystem that Vault uses, they would not be able to access any of the secrets without also possessing the master key.  This is more secure than OpenShift secrets, which are merely **encoded, not encrypted**, on disk.  Secrets are also encrypted in transit between Vault and your pods as they are loaded in.

To learn of the other advantages of Vault and to get started with it, see [Vault secrets management](https://docs.developer.gov.bc.ca/vault-secrets-management-service/)



