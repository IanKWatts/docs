# Reliability and Resiliency

The OpenShift platform, and Kubernetes in general, are designed as dynamic systems in which traditional "set it and forget it" approaches to server and application maintenance are not appropriate.  Your applications' ecosystems must be tolerant of the unplanned loss of running pods and include strategies for the continuous verification of both deployment processes and fault tolerance.

## HPAs - Horizontal Pod Autoscalers
Your applications may experience a wide range of traffic load.  In order to allow for spikes in traffic that might overwhelm your default deployment, while not unnecessarily consuming extra resources at all times, configure a horizontal pod autoscaler (HPA).  You'll set the minimum number of pods (replicas) to run, and define conditions under which it should add more pods automatically.  The extra pods will still have to fit within the resource quotas of your namespace, so you'll need to review the normal resource consumption of your Deployment or StatefulSet and ensure that an increase of X number of pods will not cause pod startup failures due to the quotas.

Requirements:
* `resources` block
* Readiness probe

HPAs have triggers for CPU consumption and/or memory consumption.  When the average resource consumption of your entire deployment is at or above the defined threshold, the deployment is scaled up, then when demand drops, it is scaled back down again.

In order for this to work, your deployment configuration must include a resource request.  That data is processed by the metrics controller and the HPA reads it from the metrics API.  The HPA uses this to know how close to the limit the deployment is for CPU and RAM.

The first thing to do is determine what a normal resource usage range is for your deployment.  Start by logging in to the OCP console, going to your Deployment or StatefulSet, and clicking the Metrics tab.  Monitor the graphs on this page periodically during normal application load, as well as during startup.

The resources block specifies a baseline, guaranteed amount of resources and a maximum, burstable resource limit, which will be provided on a best-effort basis, but will generally be available.  The resources block is a top-level parameter of a container.  The following example specifies a baseline CPU allotment of 200 millicores and RAM allotment of 1280 mibibytes; the maximum allotments are 400 millicores and 2 gibibytes.

```
        - resources:
            requests:
              cpu: 200m
              memory: 1280Mi
            limits:
              cpu: 400m
              memory: 2Gi
```
Adjust the values to meet the needs of your application without asking for more than you need.

Once you have a handle on your app's resource usage under normal load and peak load, you can configure the HPA.  The elements of the HPA are:
* The target deployment
* Minimum replicas
* Maximum replicas
* Resource thresholds 

The following example is based on the 'resources' example above.  It applies to the Deployment called "sample-app" and includes thresholds for both RAM and CPU, either of which could trigger automatic scaling of the deployment.
```
spec:
  scaleTargetRef:
    kind: Deployment
    name: sample-app
    apiVersion: apps/v1
  minReplicas: 3
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 1500Mi
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```
Once the average RAM utilization exceeds 1500Mi, or average CPU utilization exceeds 70% of maximum, the HPA will scale up the deployment from three pods to four.  If the avarage utilization again rises past the threshold, a fifth pod will be created.  If the average utilization then drops below the threshold for several minutes, it will incrementally scaled down the deployment.  There is a brief delay in the scaling down to prevent rapid scaling up and down of pods.

**Check HPA status**
Periodically check the status of your HPA with 'oc describe'.
```
$ oc describe hpa sample-app
Name:                                                  sample-app
Namespace:                                             abc123-prod
Labels:                                                app=sample-app
Annotations:                                           <none>
CreationTimestamp:                                     Fri, 26 May 2023 07:02:57 -0700
Reference:                                             Deployment/sample-app
Metrics:                                               ( current / target )
  resource memory on pods:                             1113748821333m / 1500Mi
  resource cpu on pods  (as a percentage of request):  6% (28m) / 70%
Min replicas:                                          3
Max replicas:                                          5
Deployment pods:                                       3 current / 3 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from memory resource
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:           <none>
```

For more information, see:

https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-autoscaling.html

## PDBs - Pod Disruption Budgets
Cluster maintenance happens during business hours and the expectation is that applications on the platform can tolerate the sequential cycling of worker nodes.  Sometimes this happens in rapid succession - if your application takes time to start up and pass all readiness and liveness probes, then a relocated pod may still be starting up and not ready for service when another of your deployment's pods is terminated, which could cause an outage of your application.  A pod disruption budget (PDB) allows you to specify the minimum number of pods that can be offline at a given time to prevent this scenario.

**Warning:** A badly configured PDB can interfere with cluster maintenance.  For the sake of those who maintain the platform, and those who use it, please configure your PDB carefully.

A minimal PDB contains:
* a selector for pods to protect
* the minimum number of available pods

For example, if we have a StatefulSet called "sample-app", which has a minimum of three pods, we may want to ensure that only one of them is unavailable at a time, such as with a Mongo replica set, which requires that more than half of members are available at any given time.
```
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: sample-app
```
This assumes that pods in the Deployment or StatefulSet have a label `app` with a value of `sample-app`.

Never set `minAvailable` to the minimum number of pods for the deployment.  If you do, it will interfere with cluster maintenance by preventing worker nodes from being taken offline.

**Check PDB status**
Once established, periodically check on the status of your PDB.  Ensure that the total number of pods is correct and that the selector is accurate.
```
$ oc describe pdb sample-app
Name:           sample-app
Namespace:      abc123-prod
Min available:  2
Selector:       app=sample-app
Status:
    Allowed disruptions:  1
    Current:              3
    Desired:              2
    Total:                3
Events:                   <none>
```

## Databases
Databases in particular require careful setup in OpenShift.  They must be highly available and configured for replication.  See the following documentation for details on the care and maintenance of HA databases.
* [High availability database clusters](https://docs.developer.gov.bc.ca/high-availability-database-clusters/)
* [Open-source database technologies](https://docs.developer.gov.bc.ca/opensource-database-technologies/)
* [Database backup best practices](https://docs.developer.gov.bc.ca/database-backup-best-practices/)

## Recoverability
If something unexpected were to happen in your namespace, such as accidental deletion of resources, would you be able to restore them?  Given a well-configured CI/CD pipeline, many resources can be easily recreated, but some resources require extra care.  In order to ensure that your apps can be fully restored after an unfortunate incident of some kind, implement the following preventative measures.

#### Secrets
Secrets are often omitted from CI/CD pipelines, because a secure pipeline setup requires the use of Sealed Secrets (link?).  If the Secret containing your database password were accidentally altered or deleted, would you be able to restore it?  Would your team be able to restore it in your absence?

* Use Vault
    * Vault is an on-cluster password management system that is available to all users.  It uses encryption at rest, is frequently backed up, maintains change history, and is a highly available service.  See the Security section (link) for more information.
* Use Sealed Secrets?
* Use a local password management system
    * In a pinch, you could maintain a local password management file, though this is a less useful approach, because the file must be available to any team members involved in a recovery effort and it would not be part of your pipelines.
**Never store passwords in plain text, even in a private repository.**

#### Storage
Your storage on the platform is not automatically backed up, with the exception of the `netapp-file-backup` storage class, which is not used for production workloads and is typically used only for database backup containers.  

**S3**
Amazon S3 buckets reside outside of the OpenShift clusters and are a good option for storing backup copies of non-sensitive data.  Use an image with the 'minio' client installed to copy data to your S3 bucket, preferably by cron job.  S3 buckets can be procured through your ministry's IMB; they are not offered directly from the Platform Services team.

**Database backups**
The community supported [backup-container](https://github.com/bcgov/backup-container) is an image designed to support the automated backup of various types of database.  It is easy to set up and is well understood by our community of users, so you will be able to get support if needed.

If you need to recover your backup volume from the platform-provided backup, **you will need the name of the volume, not just the PVC name.**  You can get the volume name from the PVC details.  For example:
```
$ oc get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
backup-test    Bound    pvc-02e9d855-cd63-480d-a1d7-9b638b04f6ff   20Gi       RWX            netapp-file-backup     3d19h
```
In this case, you would store the volume name 'pvc-02e9d855-cd63-480d-a1d7-9b638b04f6ff' in a safe place together with the PVC name and a description of its purpose.  The Platform Team will not be able to restore from backup without that volume name.

For more details, see the [Restoring Backup Volumes on OpenShift](https://docs.developer.gov.bc.ca/netapp-backup-restore/).
* For extra protection, copy your DB backups to an S3 bucket.

**Images**
Do your applications pull images directly from Docker Hub, ghcr.io, or another off-cluster source?  Can you be sure that the same version of the image will be available as long as you need it?  Sometimes older images are removed from container repositories, perhaps because they are no longer supported or due to security vulnerabilities.  To be sure that your images will be always available, and to protect against possible network issues preventing their retrieval, store your images in Artifactory.  See the Images section of this document for more information.

## Periodic HA Testing
After configuring HPAs and PDBs for your applications, test them periodically to make sure that they work the way you expect and that your application remains available.  If you're testing in your Test or Dev environment, make sure they're configured the same as Prod so that you can test that same Prod configuration.
* Delete a pod in a Deployment or StatefulSet
* In a DB replica set, delete a secondary member
* Delete the primary member of a DB replica set
Does the application or DB remain available?
* Test your PDB by trying to delete more pods than allowed.  Does it work as expected?

**Load testing**
If you need to run a load test against your application, first check with the Platform Services to ensure that the timing and scope of the test will not impact other users of the platform.

## CI/CD Pipeline
Users of the platform have CI/CD pipelines using [Tekton](https://docs.developer.gov.bc.ca/deploy-an-application/#continuous-deployment-and-maintenance) (OpenShift Pipelines), GitHub Actions, and [ArgoCD](https://github.com/BCDevOps/openshift-wiki/tree/master/docs/ArgoCD).

Review your pipelines from time to time.
* Are there any resources that were manually created that should be added to your pipeline?
* Are there Secrets that could be pulled from Vault instead of your OpenShift namespace?
* Aside from storage recovery processes, could your entire application be restored using the pipeline?















