# Kyverno Management

## Overview
Kyverno is a policy engine designed for Kubernetes.  It can be used to both **validate** resources, ensuring that they have all necessary parameters or configuration, and to **mutate** resources, changing them before they are actually created to add, remove, or modify parts of their definition.

Policies can be global (`ClusterPolicy`) or scoped to a namespace (`Policy`).  The Platform Services team uses ClusterPolicies to enforce best practices for security, performance, and clarity.

Kyverno receives validating and mutating admission webhook HTTP callbacks from the Kubernetes API server and applies matching policies to return results that enforce admission policies or reject requests.

Each cluster has a Kyverno deployment, which consists of four components:
* The admissions controller receives incoming requests from the Kubernetes API server and modifies them based on installed policies.
* The background controller handles all generate and mutate-existing policies by reconciling UpdateRequests, an intermediary resource.
* The report controller handles creation and reconciliation of Policy Reports.
* The cleanup controller handles CleanupPolicies and ClusterCleanupPolicies.

[Official Kyverno site](https://kyverno.io)

## GitOps Setup
The recommended installation method is the Helm chart, which is not well suited for integration with our CCM system.  Instead, we download the all-in-one YAML manifest and make some small modifications to it.

[Release information](https://github.com/kyverno/kyverno/releases) is available in the Kyverno GitHub repository.

See the most recent [upgrade notes](https://github.com/bcgov-c/platform-gitops-gen/blob/master/roles/kyverno/upgrade_notes_1.11.md) for details on the preparation of the manifest for our platform.

Like other platform services, Kyverno is managed in our [CCM system](https://github.com/bcgov-c/advsol-docs/tree/master/OCP4/CCM).

## CCM role
The main points of interest in the CCM role are the configuration customizations and ClusterPolicy manifests.

### Deployment configuration - default/main.yaml
The role's defaults file contains overrides of Deployment 'resources' and Kyverno's "client rate limits".  The client rate limits were raised after observing a lot of "client-side throttling" messages in the logs of the admissions controller.

The other main component of the defaults file is **resource filters**.  Resource filters are used to make Kyverno ignore certain kinds of resources.  That is, any resources matching a filter are not acted upon by Kyverno, even if they match a ClusterPolicy.  There are two sections of filters: the first is `resource_filters_default`, which is the stock list taken directly from the all-in-one manifest; the other is `resource_filters_local`, which includes all of our local filter additions.  They are kept separate so that we can easily update them for an upgrade without accidentally removing important local filters or retaining obsolete filters.

### Templates
Ansible templates are used for the main deployment manifest, role bindings, and the ClusterPolicies.  ClusterPolicies are Kubernetes resources and managed just like anything else.

## Writing ClusterPolicies
ClusterPolicies are somewhat complex and there is extensive information about them in the [official Kyverno documentation](https://kyverno.io/docs/kyverno-policies/).  So far, we use them to either validate resources or automatically modify them.

ClusterPolicies are composed of rules.  There can be more than one rule in a ClusterPolicy.  Each rule includes a 'match' section that defines the resources that are affected by this policy, and usually either a 'validate' or 'mutate' section.  

'Validate' rules typically block an update if a resource request does not match the requirements.  This is done with a 'deny' block.  Be sure to include a good error message when creating validate rules so that a user will know why their request has failed - the error message will be displayed in the output of an 'oc' command or in the sync status in ArgoCD.

'Mutate' rules are used to add a label to a pod, set a reasonable default value when one is not provided, and so on.

See the [existing ClusterPolicies](https://github.com/bcgov-c/platform-gitops-gen/tree/master/roles/kyverno/templates) as examples.

**Important:** Because our ClusterPolicies are created as Jinja2 templates, and ClusterPolicies also use double curly braces (`{{ }}`) like Jinja2, any section of a policy that has double braces in it must be wrapped with raw/endraw tags.  For example:
```
{%- raw %}
<snip>
        preconditions:
          any:
            - key: "{{ element.interval || '' }}"
              operator: Equals
              value: ""
<snip>
{%- endraw %}
```

## Policies
Policies are just like ClusterPolicies, but are scoped to a single namespace and not the whole cluster.  Although we do not use them in the CCM, users may create Policies themselves.  You may see messages in the log files related to these.


