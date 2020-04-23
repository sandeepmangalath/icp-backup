# Backup and Restore of Applications running on RedHat OpenShift using Velero

In this document, we will describe how to back up and restore your containerized applications running on RedHat OpenShift (RHOS) environment using Velero.


### Introduction to Veloro

[Veloro](https://github.com/vmware-tanzu/velero) is an open source, kubernetes backup recovery utility laregely contributed by Heptio team. As of writing this article, Heptio has been acquired by VMware and the community contributors have aggressively worked on the assimilation of Velero as a first class citizen in VMware's Tanzu mission critical portfolio.

Velero provides backup and restore capabilities for all or part of your kubernetes cluster. It backs up all tags, deployments, persistent volumes, and more. Since v0.6.0, Velero has adopted a plugin model which enables anyone to easily implement additional object and block storage backends, outside of the main Velero repository. This allows 

Velero natively supports backing up and restoring data from any Kubernetes volume or persistent volume and.can be used to restore individual crds, or deamon set; practically any kubernetes resources via Custom Resource Definitions.  Each Velero operation -- on-demand backup, scheduled backup, restore -- is a custom resource, defined with a Kubernetes Custom Resource Definition (CRD) and stored in etcd. Velero also includes controllers that process the custom resources to perform backups, restores, and all related operations.

This facilitates back up or restore of all objects in the cluster, or more granular filtering of objects by type, namespace, and/or label.

Velero is ideal for the disaster recovery use case, as well as for snapshotting your application state, prior to performing system operations on your cluster (e.g. upgrades).


* Supports multiple storage backends, including IBM Cloud Object Storage, Amazon S3, Google Cloud Storage, Azure Blob Storage, and Minio amongst others
* Fully encrypts backup data at rest and in transit with AES-256 in counter mode
* Only backs up data that has changed since the prior backup, using content-defined chunking
* De-duplicates data within a single backup for efficient use of storage

![](./images/ark/ct_quote.png)


### Scope

This guide provides an overview and guidance on when it is applicable to use Velero for backup and restore operations. It also provides an IBM opinated view on the Velero product, its key capabilities and future outlook. Furthure, in demonstrating how it works; this guide the learnings from using Velero to backup and restore a realistic Stateful application that is more commonly encountered in an enterprise setting.

*****************
Replace

We will set up and configure the Ark client on a local machine, and deploy the Ark server into our Kubernetes cluster. We'll then deploy a sample Nginx app that uses a Persistent Volume for logging, backup the application to IBM Cloud Object Storage, simulate a disaster recovery scenario and restore the application with its persistent volume.

We are following the [companion guide for IBM Cloud Kubernetes Service (IKS)](https://medium.com/@mlrborowski/using-ark-and-restic-to-provide-dr-for-ibm-kubernetes-service-cae53cfe532) with the following differences:
1. Using ICP vs. IKS
2. Using NFS storage as the storage options for the ICP cluster vs. GlusterFS

This proves:
1. The underlying framework including kubernetes is exactly the same in ICP and IKS as the open sourced kubernetes project; supporting portability across public, private and multi-cloud environments.
2. Ark and Restic is storage agnostic.
********************

## Usage Guidance

This article does not attempt to compare Velero with other traditional backup and recovery tools; but rather guides the reader on its applicability in next gen architectures based on containers and kubernetes.

While GitOps as a methodology triggering operational tasts including backup and restore are gaining popularity with Site Reliability Engineering (SRE) team; Velero caters to the Traditional IT Operations team in providing snapshot based full and incremental backups in a Cloud native container architecture built using native Kubernetes APIs.

Furthur one important thing to note is that GitOps can only restore Kubernetes objects so that means any persistent data required for an application to correctly function must be restored for stateful applications, such as databases, to be back in service.

###Multi-architecture Container Image Support
Velero also provides [multi-arch container images support](https://velero.io/blog/velero-1.3-voyage-continues/) by using Docker manifest lists which adds Linux on Power systems support.

###Physical vs. Logical Backups

Apart from Physical backups; logical backups are an important topic to address here as it save time on operational backup tasks, especially in the case of databases to provide a point-in-time application consistent backup. These are triggered on a per workload basis and only extracts the data from the data files into dump files.

Velero supports application consistent backups via the construct of [hooks](https://velero.io/docs/v1.3.2/hooks/). When performing a backup, a user can specify one or more commands to execute in a container in a pod when that pod is being backed up. The commands can be configured to run before any custom action processing ("pre" hooks), or after all custom actions have been completed and any additional items specified by custom action have been backed up ("post" hooks).

Thus the guidance to use Velero when traditional IT teams prefer to back up their containerized applications orchestrated by kubernetes using cloud native  backup and recovery tools based on next gen architectures.


### Solution Overview

Our current setup includes two master nodes, two proxy nodes, two management nodes, three worker nodes, and one storage (NFS) node.

In order to follow all of the recommendations in this guide, it is assumed that you have already provisioned an ICP cluster and set up NFS storage for the same, and are able to have access to your cluster immediately post-install.

![ Ark Backup Solution Overview ](./images/ark/ark_flow.png)
A simple overview of the process is as follows:

* Login (or first create) to your IBM Cloud Account.
* Create and configure IBM object storage service.
* Install Ark Client.
* Configure Ark and Restic.
* Login to your ICP cluster
* Install Ark and Restic into your ICP cluster.
* Deploy an application and make a change to the PV content.
* Run Ark backup.
* Delete the application and PV, simulating disaster.
* Restore application from Ark/Restic Backup and all is well again.

## Task 1: Setup your Backup target

We will use the IBM Cloud Object Storage (COS) service as the backup target.

### Step 1. Login to the IBM Cloud (or create you free account if this is your first time)

https://console.cloud.ibm.com

### Step 2. Create an IBM Cloud Object Storage Service Instance

To store Kubernetes backups, you need a destination bucket in an instance of Cloud Object Storage (COS) and you have to configure service credentials to access this instance.

If you don’t have a COS instance, you can create a new one, according to the detailed instructions in Creating a new resource instance. The next step is to create a bucket for your backups. Ark and Restic will use the same bucket to store K8S configuration data as well as Volume backups. See instructions in Create a bucket to store your data. We are naming the bucket arkbucket and will use this name later to configure Ark backup location. You will need to choose another name for your bucket as IBM COS bucket names are globally unique. Choose “Cross Region” Resiliency so it is easy to restore anywhere.

![COS Bucket Creation (arkbucket shown but create restic bucket also)](./images/ark/icos_create_bucket.png)


The last step in the COS configuration is to define a service that can store data in the bucket. The process of creating service credentials is described in Service credentials. Several comments:

```
Your Ark service will write its backup into the bucket, so it requires the “Writer” access role.
Ark uses an AWS S3 compatible API. Which means it authenticates using a signature created from a pair of access and secret keys — a set of HMAC credentials. You can create these HMAC credentials by specifying {“HMAC”:true} as an optional inline parameter. See step 3 in the Service credentials guide.
```

![COS Service Credentials](./images/ark/icos_service_credentials.png)

After successfully creating a Service credential, you can view the JSON definition of the credential. Under the ```cos_hmac_keys``` entry there are ```access_key_id``` and ```secret_access_key```. We will use them later.



## Task 2: Setup Ark

### Step 3. Download and Install Ark

Download Ark as described here: https://heptio.github.io/ark/v0.10.0/. A single tar ball download (https://github.com/heptio/ark/releases) should install the Ark client program along with the required configuration files for your cluster.

Note that you will need Ark v0.10.0 or above for the Restic integration as shown in these instructions.
Add the Ark client program (ark) somewhere in your $PATH.

### Step 4. Configure Ark Setup

Configure your kubectl client to access your IKS deployment. From the Ark root directory, edit the file config/ibm/05-ark-backupstoragelocation.yaml file. Add your COS keys as a Kubernetes Secret named cloud-credentials as shown below. Be sure to update <access_key_id> and <secret_access_key> with the value from your IBM COS service credentials. The remaining changes in the file are in the section showing the BackupStorageLocation resource named default. Configure access to the bucket arkbucket (or whatever you called yours) by editing the spec.objectstore.bucket section of the file. Edit the COS region and s3URL to match your choices.

```
apiVersion: v1
kind: Secret
metadata:
  namespace: heptio-ark
  name: cloud-credentials
stringData:
  cloud: |
    [default]
    # UPDATE ME: the value of “access_key_id” of your COS service credential
    aws_access_key_id = <access_key_id>
    # UPDATE ME: the value of “secret_access_key” of your COS service credential
    aws_secret_access_key = <secret_access_key>
---
apiVersion: ark.heptio.com/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: heptio-ark
spec:
  provider: aws
  objectStorage:
    bucket: arkbucket
  config:
    s3ForcePathStyle: "true"
    s3Url:  http://s3-api.us-geo.objectstorage.softlayer.net
    region: us-geo
```
The file should look something like this when done.

### Step 5. Login to your ICP cluster

![ICP Client Credentials](./images/ark/icp_client_config.png)

After you login to the cluster's Admin Console, copy the client configurations from your profile. You will use these credentials to login to your cluster from a terminal.

### Step 6. Deploy Ark into your IBM ICP Instance

Run the following commands from the Ark root directory:
```kubectl apply -f config/common/00-prereqs.yaml```

```kubectl apply -f config/ibm/05-ark-backupstoragelocation.yaml```

```kubectl apply -f config/ibm/10-deployment.yaml```

Should you encounter an error as below, please note that this is caused by Vulnerability Advisor blocking grc content. You can resolve this by adding  gcr.io/heptio-images/* to namespace heptio-ark

Error:
Deny "gcr.io/heptio-images/ark:v0.10.1", no matching repositories in ClusterImagePolicy and no ImagePolicies in the "heptio-ark" namespace

```kubectl apply -f config/aws/20-restic-daemonset.yaml```

Verify that Ark and Restic are running correctly on your ICP cluster with the following command:

```kubectl -n heptio-ark get pods```

which should show pods running similar to this:

```
NAME READY STATUS RESTARTS AGE
ark-7b9c58c57d-rnqff 1/1 Running 0 5m
restic-b8x5m 1/1 Running 0 5m
restic-hzgt7 1/1 Running 0 5m
restic-qwz5x 1/1 Running 0 5m
```

Note above that the count may vary as there is one Ark pod and a Restic Daemon set (in this case 3 pods, one per worker node).


## Step 7. Create Namespace, Persistent Volume and Persistent Volume Claim

You can use the ICP admin console to create the Namespace, Persistent Volume and Persistent Volume Claim as below.


![ICP create Namespace](./images/ark/icp_create_namespace.png)

Under the Platform, Storage settings create the Persistent Volume and Persistent Volume Claim.

![ICP create Persistent Volume](./images/ark/icp_create_pv.png)
![ICP create Persistent Volume Claim](./images/ark/icp_create_pvc.png)

## Task 3: Setup your Application for Backup

## Step 8. Deploy a sample Application with a Volume to be Backed Up

From the Ark root directory cut the yaml code below and save it asconfig/ibm/with-pv.yaml. We are creating a simple nginx deployment in its own namespace along with a service and a dynamically provisioned PV where we store nginx logs. Note the annotation: backup.ark.heptio.com/backup-volumes: nginx-logs below which tells Restic the volume name that we are interested in backing up.

```
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
 name: nginx-deployment
 namespace: nginx-example
spec:
 replicas: 1
 template:
   metadata:
     annotations:
       backup.ark.heptio.com/backup-volumes: nginx-logs
     labels:
       app: nginx
   spec:
     volumes:
       - name: nginx-logs
         persistentVolumeClaim:
           claimName: claim-nginx-logs
     containers:
     - image: nginx:1.7.9
       name: nginx
       ports:
       - containerPort: 80
       volumeMounts:
         - mountPath: "/storage"
           name: nginx-logs
           readOnly: false
---
apiVersion: v1
kind: Service
metadata:
 labels:
   app: nginx
 name: my-nginx
 namespace: nginx-example
spec:
 ports:
 - port: 80
   targetPort: 80
 selector:
   app: nginx
 type: LoadBalancer
```
Now we can deploy this sample app by running the following from the Ark root directory:

```kubectl create -f config/ibm/with-pv.yaml```

We can check if the storage has been provision with:

```kubectl -n nginx-example get pvc```

You may have to run this over the course of a few minutes as the PVC gets bound. It will show pending but eventually show as bound similar to:

```
NAME               STATUS    VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim-nginx-logs   Bound     nginx-logs   24Gi       RWX            nfsstorage 3m
```

Now that we have a volume mounted we can find out the nginx pod name and put something in the volume (or just access the nginx web frontend and see access logs grow). Get your pod name with the following command (sample output shown):
```
kubectl -n nginx-example get pods


NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-54c66df98b-6ppt5   1/1       Running   0          25s
```
Using the above pod name (yours will differ) we can log into the instance and add a file with the following commands:

```
 kubectl -n nginx-example exec -it nginx-deployment-54c66df98b-6ppt5 -- /bin/bash
root@nginx-deployment-54c66df98b-6ppt5:/# cd /storage
root@nginx-deployment-54c66df98b-6ppt5:/storage# echo "hw test it is late" > hw-test.txt
```

We now have some content we would expect to be saved and restored with the addition of our hw.txt file. You can, of course just access the nginx front end service via your browser and see the access.log grow also.

## Step 9. Use Ark and Restic to backup K8S config and volume.

We can backup up our sample application by scoping the backup to the application’s namespace as follows:

```
ark backup create my-nginx-bu-test-late --include-namespaces nginx-example

Backup request “my-nginx-bu-test-late” submitted successfully.
Run `ark backup describe my-nginx-bu-test-late` for more details.
```

We can check the result with:

```ark backup describe my-nginx-bu-test-late --details```

which after repeating a few times the result should show a complete status.

![Ark completion](./images/ark/ark_completion.png)

If you examine your IBM Cloud COS bucket associated with the backup you will see that a set of files has appeared.

## Task 4: Simulate Disaster and Restore Application

## Step 10. Simulating Disaster

With the following commands we will delete our application configuration and the PV associated and confirm they are removed:

```
kubectl delete namespace nginx-example
namespace “nginx-example” deleted
```
```
kubectl get pvc -n nginx-example
No resources found.
```
```
kubectl get pods -n nginx-example
No resources found.
```

If your PV is not deleted, manually remove the PV.
```
kubectl delete pv nginx-logs

Verify that the PV is removed.
```
kubectl get pv
```

```
## Step 11. Recovering from Disaster

We can restore the application and volume with the following command:

```
ark restore create --from-backup my-nginx-bu-test-late

Restore request "my-nginx-bu-test-late-20190207183813" submitted successfully.
Run `ark restore describe my-nginx-bu-test-late-20190207183813` or `ark restore logs my-nginx-bu-test-late-20190207183813` for more details.
```

Restoring will take longer because we are dynamically provisioning another network drive behind the scenes.  Run the ark restore describe <restore_request_name> --details command to observe the progress.

```
ark restore describe my-nginx-bu-test-late-20190207183813 --details
Name:         my-nginx-bu-test-late-20190207183813
Namespace:    heptio-ark
Labels:       <none>
Annotations:  <none>

Backup:  my-nginx-bu-test-late

Namespaces:
  Included:  *
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        nodes, events, events.events.k8s.io, backups.ark.heptio.com, restores.ark.heptio.com
  Cluster-scoped:  auto

Namespace mappings:  <none>

Label selector:  <none>

Restore PVs:  auto

Phase:  Completed

Validation errors:  <none>

Warnings:  <none>
Errors:    <none>

Restic Restores:
  Completed:
    nginx-example/nginx-deployment-54c66df98b-6ppt5: nginx-logs

```

Within a minute or two we see our application is up and the volume recovered using the commands below (your pod name will differ). We dump our “hello world” file (hw-test.txt) and its contents are what we had per-disaster, mission accomplished!

```
kubectl get pods -n nginx-example

NAME READY STATUS RESTARTS AGE
nginx-deployment-54c66df98b-6ppt5 1/1 Running 0 6m

kubectl -n nginx-example exec -it nginx-deployment-54c66df98b-6ppt5 -- cat /storage/hw-test.txt

hw test it is late
```

## Limitations
Presently known limitations include:

1. Velero only supports a single set of credentials per provider. It's not yet possible to use different credentials for different locations, if they're for the same provider.

2. Volume snapshots are still limited by where your provider allows you to create snapshots. For example, AWS and Azure do not allow you to create a volume snapshot in a different region than where the volume is. If you try to take a Velero backup using a volume snapshot location with a different region than where your cluster's volumes are, the backup will fail.

3. Each Velero backup has one BackupStorageLocation, and one VolumeSnapshotLocation per volume provider. It is not possible (yet) to send a single Velero backup to multiple backup storage locations simultaneously, or a single volume snapshot to multiple locations simultaneously. However, you can always set up multiple scheduled backups that differ only in the storage locations used if redundancy of backups across locations is important.

4. Cross-provider snapshots are not supported. If you have a cluster with more than one type of volume (e.g. EBS and Portworx), but you only have a VolumeSnapshotLocationconfigured for EBS, then Velero will only snapshot the EBS volumes.

5. Restic data is stored under a prefix/subdirectory of the main Velero bucket, and will go into the bucket corresponding to the BackupStorageLocation selected by the user at backup creation time.

## Outlook

Velero's agnostic posture is on close watch as an Open source project.

When assessing the outlook of an open source project it is important to evaluate on 3 key factors:
1. Ability to nimbly incorporate features benefiting the community
2. Diversity in contributors and number of active contributions 
2. Adoption that drives the need for new features functions as additional usecases are identified. 


1. Incorporating Community Features

The single most important advancement in Velero is the adoption of Container Storage Interface (CSI). [CSI was developed and made GA in December of 2018 as a standard as of Kubernetes v1.13](https://kubernetes.io/blog/2018/12/03/kubernetes-1-13-release-announcement/) for exposing arbitrary block and file storage storage systems to containerized workloads on Container Orchestration Systems (COs) like Kubernetes. Prior to this, third-party storage code caused reliability and security issues in core Kubernetes binaries and the code was often difficult (and in some cases impossible) for Kubernetes maintainers to test and maintain. With the adoption of the Container Storage Interface, the Kubernetes volume layer becomes truly extensible. Using CSI, third-party storage providers can write and deploy plugins exposing new storage systems in Kubernetes without ever having to touch the core Kubernetes code. This gives Kubernetes users more options for storage and makes the system more secure and reliable.

Shortly after the acquisition of Heptio by VMware, Velero was born as a project. Largely driven by core-contributors from Heptio the community saw early investment in driving tighter integration with VMware.

Features such as support for backup and restore of Stateful [applications running natively on vSphere](https://velero.io/blog/velero-v1-1-stateful-backup-vsphere/) were prioritized over CSI in v1.1.

Furthur, integration into [VMware Tanzu portfolio](https://velero.io/blog/announcing-gh-move/)  in v1.2 was prioritized over much awaited CSI capabilities which is still in beta.

VMware continues to drive proliferation of Velero in its EMC storage family of products. For example Velero is gaining usability features starting with its [deep integration in PowerProtect](https://itzikr.wordpress.com/2019/12/31/dell-emc-powerprotect-19-3-is-available-kubernetes-integration-you-bet/)

Velero's implementation of CSI requires Kubernetes version 1.17 or greater and greatly relies on adoption and testing on the [three main hyperscalar Cloud Service Providers that Velero formally supports](https://velero.io/docs/master/supported-providers/). As a result, the much awaited CSI capability continues to remain in beta. Velero's limited support stance coupled with slow adoption by Cloud Providers on upstream Kubernetes releases is hampering innovation.

As of writing this article, the commercial Cloud Providers list that can realistically test CSI features of Velero are:
Kubernetes Service                  Latest K8s version supported      
IBM Kubernetes Service (IKS)    [1.17](https://cloud.ibm.com/docs/containers?topic=containers-cs_versions)               
and following commercial Cloud Providers list that cannot realistically test CSI features of Velero are:
Amazon (EKS)                            [1.15.11](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
Azure (AKS)                             [1.15.x](https://azure.microsoft.com/en-us/updates/azure-kubernetes-service-will-be-retiring-support-for-kubernetes-versions-1-11-and-1-12/)
Alibaba (CSK)                            [1.16.6](https://www.alibabacloud.com/help/doc-detail/115453.htm)
Google (GKE)                            [1.15.11](https://www.alibabacloud.com/help/doc-detail/115453.htm)
Oracle (CEK)                             [1.15.7](https://docs.cloud.oracle.com/en-us/iaas/releasenotes/changes/37013251-39b2-4c08-8536-906d76bba789/)

2. Contributors:
As of writing this document [4 of Velero contributors may be considered active and contributing](https://github.com/vmware-tanzu/velero/graphs/contributors)

3. Adoption:
As of writing this document [Velero has 10 adopters](https://github.com/vmware-tanzu/velero/blob/master/ADOPTERS.md)



## Summary
With this simple usecase, we have proven backup and restore of an application using Velero. Velero offers a developer friendly option to rapid recovery of container hosted applications and their supporting persistent volumes. The extensible plugin based model makes it possible for developers and administrators to support additional PersistentVolume types. While, its strength lies in the Disaster Recovery space supporting Backup and Restore operations; it can support cluster portability by migrating resources between clusters e.g. between Dev/Test environments or across multiple cloud providers.
