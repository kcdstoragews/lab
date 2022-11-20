# :trident: Everything you always wanted to know about storage in Kubernetes (But were too afraid to ask) 

# Introduction


You will work in the NetApp Lab on Demand environment, but we`ve prepared some more resources for you than the typical lab guide offers. Before we can start, there are some preparations to do.

You will connect to a Windows jumphost from which you can access the training environment.  
It will provide you a K8s cluster version 1.22.3, a preconfigures storage system and the CSI driver we will use to demonstrate you how easy it is to use persistent storage in K8s.

## Preparation

1. Plesase choose a username from the following document:  
https://tinyurl.com/kcdlondon

2. Access the lab environment:  
https://lod-bootcamp.netapp.com

3. Request the lab *Using Trident with Kubernetes and ONTAP v5.0* and connect to the jumphost

4. Open *Putty* and connect to the host *rhel3* with the following credentials:  
username: *root*   
password: *Netapp1!*

5. We've prepared some exercises for you that are hosted in this github repo. To have them available on your training environment, please create a directory and clone the repo with the following commands:

```console
cd /root
mkdir kcdlondon
cd kcdlondon
git clone https://github.com/kcdstoragews/lab
```

You should now have several directories available. The lab is structured with different scenarios. Everything you need is placed in a folder with the same name. 

6. As this lab is used for different things and has a lot of stuff in it that might be confusing, please run this little cleanup script which removes all things we don't need in our workshop, updates the environment to a recent version and creates some necessary stuff.   
Please run the following commands:

```console
cd /root/kcdlondon/lab/prework
sh prework.sh
```

# :trident: Scenario 01 - storage classes, persistent volumes & persistent volume claims 
____
**Remember: All required files are in the folder */root/kcdlondon/lab/scenario01* please ensure that you are in this folder now. You can do this with the command** 
```console
cd /root/kcdlondon/lab/scenario01
```
____
In this scenario, you will create two StorageClasses, discovery their capabilities, create PVCs and do some basic troubleshooting. 
## 1. Backends and StorageClasses
You are using NetApp Astra Trident as the CSI driver in this lab. It is running in the namespace *trident*.
The backends in this environment are allready created. Take a brief moment to review them:

```console
kubectl get tbc -n trident
```

First let's create two StorageClasses. We've already prepared the necessary files. There is one storage class prepared for the nas backend and one for san.

The file you will use for nas is called *sc-csi-ontap-nas.yaml*  
The command...

```console
cat sc-csi-ontap-nas.yaml
```

...will provide you the following output of the file:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storage-class-nas
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: csi.trident.netapp.io
parameters:
  backendType: "ontap-nas"
allowVolumeExpansion: true 
```

You can see the following:
1. This StorageClass will be the default in this cluster (see "annotations")
2. NetApp Astra Trident is responsible for the provisioning of PVCs with this storage class (see "provisioner")
3. There are some parameters needed for this provisioner. In our case we have to tell them the backend type (e.g. nas, san).
4. This volume can be expanded after it's creation.

Now let's compare with the one for san:

```console
cat sc-csi-ontap-san.yaml
```

You get a similar output here:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storage-class-san
provisioner: csi.trident.netapp.io
parameters:
  backendType: "ontap-san"
  fsType: "ext4"
mountOptions:
   - discard
reclaimPolicy: Retain
allowVolumeExpansion: true
```

The biggest differences are: 
1. As you are now asking for a block device, you will need a file system on it to make it usable. *ext*4 is specified here (look at fsType in parameters Section)
2. A reclaim policy is specified. We will come back to this later.

If you want to dive into the whole concept of storage classes, this is well documented here: https://kubernetes.io/docs/concepts/storage/storage-classes/

After all this theory, let's just add the StoraceClasses to your cluster:

```console
kubectl apply -f sc-csi-ontap-nas.yaml
kubectl apply -f sc-csi-ontap-san.yaml 
```

You can discover all existing StorageClasses with a simple command:

```console
kubectl get sc
```

If you want to see more details a *describe* will provide them. Let's do this

```console
kubectl describe sc storage-class-nas
```

The output shows you all details. Remember, we haven't specified a *reclaimPolicy* for this class. Therefore the default vaule of *Delete* can be observed in the output. Again, we will look at this later.

## 2. PVCs & PVs

As your cluster has now a CSI driver installed and also StorageClasses, you are all set to ask for storage. But don't be afraid. You will not have to open a ticket at your storage admin team or do some weird storage magic. We want a persistent volume, so let's claim one.  
The workflow isn't complex but important to understand. 

1. A user creates a PersistentVolumeClaim requesting a new PersistentVolume of a particular size from a Kubernetes StorageClass that was previously configured by someone.
2. The Kubernetes StorageClass identifies the CSI driver - in our case Trident - and includes parameters that tell Trident how to provision a volume for the requested class.
3. Trident provisions storage on a matching backend and creates a PersistentVolume in Kubernetes that tells Kubernetes how to find, mount, and treat the volume.
4. Kubernetes binds the PersistentVolumeClaim to the new PersistentVolume. Pods that include the PersistentVolumeClaim can now mount the PersistentVolume on any host that they runs on.

There are two files in your scenario01 folder, *firstpvc.yaml* and *secondpvc.yaml*, both a requesting a 5GiB Volume. Let's create a namespace first, called *funwithpvcs*. We then get the storage into this namespace...

```console
kubectl create namespace funwithpvcs
kubectl apply -f firstpvc.yaml -n funwithpvcs
kubectl apply -f secondpvc.yaml -n funwithpvcs
```

Kubernetes confirms, that both persistent volume claimes have been created. Great... or not? Let's have a look

```console
kubectl get pvc -n funwithpvcs
```

You can see that the PVC named *firstpvc* has a volume, and is in status *Bound*. The PVC with the name *secondpvc*  does not look that healthy, it is still in Status *Pending*. This means that the request is ongoing, K8s tries to get what you want, but for whatever reason it doesn't work.   
Luckily we can describe objects and see what the problem is!

```console
kubectl describe pvc secondpvc -n funwithpvcs
```

Ok we can see, that there is an issue with the StorageClass. But why?  
Everything that is requested in the PVC will be handed over to the provisioner that is defined in the StorageClass. In this case Trident gets a request for a RWX volume with 5GiB and for the backend "ontap-san".   
In contrast to K8s, the CSI Driver is aware what is possible and what not. It recognizes that a RWX volume isn't possible at this backend type (SAN/block storage) as this backend can only serve RWO and ROX. 

If you want to have your second pvc also running and still need RWX access mode, we have to modify the yaml file. Just switch the storage class to *storage-class-nas*. This StorageClass has a backend type that is able to provide RWX mode. Unfortunately a lot of things in a PVC are immutable after creation so before we can see whether this change is working or not, you have to delete the pvc again.
___
<details><summary>Click for the solution</summary>
Edit the *secondpvc.yaml* file like this:

```console
kubectl delete -f secondpvc.yaml -n funwithpvcs
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: secondpvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: storage-class-nas
```
Apply the pvc again

```console
kubectl apply -f secondpvc.yaml -n funwithpvcs
```
</details>

___

After you have deleted the PVC, changed the StorageClass in the pvc file and applied it again, you should see that both pvcs are now bound.


```console
kubectl get pvc -n funwithpvcs
```



```sh
     NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
    firstpvc    Bound    pvc-eb00c989-fddb-4224-aa56-a8918064b9fb   5Gi        RWO            storage-class-san           16m
    secondpvc   Bound    pvc-50f6c56b-3575-43b3-ae16-5b99b35d9a59   5Gi        RWX            storage-class-nas           8s
```

Earlier we mentioned that a *PersistentVolume* is also created. Maybe you ask yourself where to see them. It is pretty easy, let's have a look at our recently created ones:

```console
kubectl get pv
```

```sh
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS        REASON   AGE
    pvc-0e860943-3fae-4dae-80e9-82e2259089c8   5Gi        RWX            Delete           Bound    funwithpvcs/secondpvc   storage-class-nas            99s
    pvc-4adaf625-fc86-42e8-a481-92b02addfdbc   5Gi        RWO            Retain           Bound    funwithpvcs/firstpvc    storage-class-san            2m43s
```

You remember the ReclaimPolicy we definied in our StorageClass? We can see here that pur PVs have different policies. Let's delete both PVCs and see what happens.

```console
kubectl delete -f firstpvc.yaml -n funwithpvcs
kubectl delete -f secondpvc.yaml -n funwithpvcs
```

Let's have a look at PVCs and PVs now 

```console
kubectl get pvc,pv -n funwithpvcs
```

Magic, both PVCs are gone (well... we advised k8s to remove them...) but one PV is still there? No not real magic, just the normal behaviour of the specified ReclaimPolicy. As described before, the default ReclaimPolicy is *Delete*. This means as soon as the corresponding PVC is deleted, the PV will be deleted too. In some use cases this would delete valuable data. To avoid this, you can set the ReclaimPolicy to *Retain*. If the PVC is deleted now, the PV will change its Status from *Bound* to *Released*. The PV could be used again.  

Awesome, you are now able to request storage...but as long as no appliaction is using that, there is no real sense of having persistent storage. Let's create an application that is able to do something with the storage. For this purpose we will use *Ghost* a light weight web blog. There are some files in our scenario01 directory:

- ghost-pvc.yaml to manage the persistent storage of this app
- ghost-deployment.yaml that will define how to manage the app
- ghost-service.yaml to expose the app

You are going to create this app in its own namespace which will be *ghost*. You will use the StorageClass for nas.

```console
kubectl create namespace ghost
```

Have a look at the file for the pvc and create it afterwards:

```console
kubectl apply -f ghost-pvc.yaml -n ghost
```

Now as the PVC is there, Have a look at the file for the deployment and create it afterwards:

```console
kubectl apply -f ghost-deploy.yaml -n ghost
```

We have an app, we have storage for the app, to access it we finally need a service:

```console
kubectl apply -f ghost-service.yaml -n ghost
```

You can see a summary of what you've done with the following command:

```console
kubectl get -n ghost all,pvc,pv
```

It takes about 40 seconds for the POD to be in a running state The Ghost service is configured with a NodePort type, which means you can access it from every node of the cluster on port 30080.   
Give it a try ! => Open the Browser in your Lab environment and go to http://192.168.0.63:30080

Let's see if the */var/lib/ghost/content* folder is indeed mounted to the NFS PVC that was created.

```console
kubectl exec -n ghost $(kubectl -n ghost get pod -o name) -- df /var/lib/ghost/content
```

You should be able to see that it is mounted. Let's have a look into the folder

```console
kubectl exec -n ghost $(kubectl -n ghost get pod -o name) -- ls /var/lib/ghost/content
```

The data is there, perfect. If you want to, you can easily clean up a little bit before you start with the next scenario:

```console
kubectl delete ns ghost
```

# :trident: Scenario 02 - running out of space? Let's expand the volume 
____
**Remember All required files are in the folder */root/kcdlondon/lab/scenario02*. Please ensure that you are in this folder. You can do this with the command ```*cd /root/kcdlondon/lab/scenario02*```**
____
Sometimes you need more space than you thought before. For sure you could create a new volume, copy the data and work with the new bigger PVC but it is way easier to just expand the existing.

First let's check the StorageClasses

```console
kubectl get sc 
```

Look at the column *ALLOWVOLUMEEXPANSION*. As we specified earlier, both StorageClasses are set to *true*, which means PVCs that are created with this StorageClass can be expanded.  
NFS Resizing was introduced in K8S 1.11, while iSCSI resizing was introduced in K8S 1.16 (CSI)

Now let's create a PVC & a Centos POD using this PVC, in their own namespace.

```console
kubectl create namespace resize
kubectl apply -n resize -f pvc.yaml
kubectl apply -n resize -f pod-busybox-nas.yaml
```

Wait until the pod is in running state - you can check this with the command

```console
kubectl get pod -n resize
```

Finaly you should be able to see that the 5G volume is indeed mounted into the POD

```console
kubectl -n resize exec busyboxfile -- df -h /data
```

Resizing a PVC can be done in different ways. We will edit the definition of the PVC & manually modify it.  
Look for the *storage* parameter in the spec part of the definition & change the value (in this example, we will use 15GB)
The provided command will open the pvc definition.

```console
kubectl -n resize edit pvc pvc-to-resize-file
```

change the size to 15Gi like in this example:

```yaml
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 15Gi
  storageClassName: storage-class-nas
  volumeMode: Filesystem
```

you can insert something by pressing "i", exit the editor by pressing "ESC", type in :wq! to save&exit. 

Everything happens dynamically without any interruption. The results can be observed with the following commands:

```console
kubectl -n resize get pvc
kubectl -n resize exec busyboxfile -- df -h /data
```

This could also have been achieved by using the _kubectl patch_ command. Try the following:

```console
kubectl patch -n resize pvc pvc-to-resize-file -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

So increasing is easy, what about decreasing? Try to set your volume to a lower space, use the edit or the patch mechanism from above.
___

<details><summary>Click for the solution</summary>

```console
kubectl patch -n resize pvc pvc-to-resize-file -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
```
</details>

___

Even if it would be technically possible to decrease the size of a NFS volume, K8s just doesn't allow it. So keep in mind: Bigger ever, smaller never. 

If you want to, clean up a little bit

```console
kubectl delete namespace resize
```



# :trident: Scenario 03 -  snapshots, clones etc 
___
**Remember: All required files are in the folder */root/kcdlondon/lab/scenario03*. Please ensure that you are in this folder. You can do this with the command** 
```console
cd /root/kcdlondon/lab/scenario03
```
___
CSI Snapshots have been promoted GA with Kubernetes 1.20.  
While snapshots can be used for many use cases, we will explore 2 different ones, which share the same initial process:

- Restore the snapshot in the current application
- Create a new POD which uses a PVC created from the snapshot (cloning)

There is also a chapter that will show you the impact of deletion between PVC, Snapshots & Clones (spoiler alert: no impact).  

We would recommended checking that the CSI Snapshot feature is actually enabled on this platform.  

This [link](https://github.com/kubernetes-csi/external-snapshotter) is a good read if you want to know more details about installing the CSI Snapshotter.
It is the responsibility of the Kubernetes distribution to provide the snapshot CRDs and Controller. Unfortunately some distributions do not include this. Therefore verify (and deploy it yourself if needed).

In our lab the **CRD** & **Snapshot-Controller** to enable this feature have already been installed. Let's see what we find:

```console
kubectl get crd | grep volumesnapshot
```

will show us the crds

```bash
volumesnapshotclasses.snapshot.storage.k8s.io         2020-08-29T21:08:34Z
volumesnapshotcontents.snapshot.storage.k8s.io        2020-08-29T21:08:55Z
volumesnapshots.snapshot.storage.k8s.io               2020-08-29T21:09:13Z
```

```console
kubectl get pods --all-namespaces -o=jsonpath='{range .items[*]}{"\n"}{range .spec.containers[*]}{.image}{", "}{end}{end}' | grep snapshot-controller
```

will show us the snapshot controller is running in our cluster:

```bash
k8s.gcr.io/sig-storage/snapshot-controller:v4.2.0,
k8s.gcr.io/sig-storage/snapshot-controller:v4.2.0,
```

Aside from the 3 CRDs & the Controller StatefulSet, the following objects have also been created during the installation of the CSI Snapshot feature:

- serviceaccount/snapshot-controller
- clusterrole.rbac.authorization.k8s.io/snapshot-controller-runner
- clusterrolebinding.rbac.authorization.k8s.io/snapshot-controller-role
- role.rbac.authorization.k8s.io/snapshot-controller-leaderelection
- rolebinding.rbac.authorization.k8s.io/snapshot-controller-leaderelection

Finally, you need to create a _VolumeSnapshotClass_ object that connects the snapshot capability with the Trident CSI driver.

```console
kubectl apply -f sc-volumesnapshot.yaml
```

You can see this *VolumeSnapshotClass* with the following command:

```console
kubectl get volumesnapshotclass
```

Note that the _deletionpolicy_ parameter could also be set to _Retain_.

The _volume snapshot_ feature is now ready to be tested.

The following will walk you through the management of snapshots with a simple lightweight BusyBox container.

We've prepared all the necessary files for you to save a little time. Please prepare the environment with the following commands:

```console
kubectl create namespace busybox
kubectl apply -n busybox -f busybox.yaml
kubectl get -n busybox all,pvc
```

The last line will provide you an output of our example environment. There should be one running pod and a pvc with 10Gi.

Before we create a snapshot, let's write some data into our volume.  

```console
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- sh -c 'echo "KCDUK 2022 is fun" > /data/test.txt'
```

This creates the file test.txt and writes *KCDUK 2022 is fun" into it. You can verify the file contents:

```console
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- more /data/test.txt
```

Creating a snapshot of this volume is very simple:

```console
kubectl apply -n busybox -f pvc-snapshot.yaml
```

After it is created you can observe its details:
```console
kubectl get volumesnapshot -n busybox
```
Your snapshot has been created !  

To experiment with the snapshot, let's delete our test file...
```console
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- rm -f /data/test.txt
```

If you want to verify that the data is really gone, feel free to try out the command from above that has shown you the contents of the file:

```console
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- more /data/test.txt
```

One of the useful things K8s provides for snapshots is the ability to create a clone from it. 
If you take a look a the PVC manifest (_pvc_from_snap.yaml_), you can notice the reference to the snapshot:

```yaml
dataSource:
  name: mydata-snapshot
  kind: VolumeSnapshot
  apiGroup: snapshot.storage.k8s.io
```

Let's see how that turns out:

```console
kubectl apply -n busybox -f pvc_from_snap.yaml
```

This will create a new pvc which could be used instantly in an application. You can see it if you take a look at the pvcs in your namespace:

```console
kubectl get pvc -n busybox
```

Recover the data of your application

When it comes to data recovery, there are many ways to do so. If you want to recover only a single file, you can temporarily attach a PVC clone based on the snapshot to your pod and copy individual files back. Some storage systems also provide a convenient access to snapshots by presenting them as part of the filesystem (feel free to exec into the pod and look for the .snapshot folders on your PVC). However, if you want to recover everything, you can just update your application manifest to point to the clone, which is what we are going to try now:

```console
kubectl patch -n busybox deploy busybox -p '{"spec":{"template":{"spec":{"volumes":[{"name":"volume","persistentVolumeClaim":{"claimName":"mydata-from-snap"}}]}}}}'
```

That will trigger a new POD creation with the updated configuration

Now, if you look at the files this POD has access to (the PVC), you will see that the *lost data* (file: test.txt) is back!

```console
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- ls -l /data/
```
or even better, lets have a look at the contents:

```console
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- more /data/test.txt
```

Tadaaa, you have restored your data!  
Keep in mind that some applications may need some extra care once the data is restored (databases for instance). In a production setup you'll likely need a more full-blown backup/restore solution.  


As in every scenario, a little clean up at the end:

```console
kubectl delete ns busybox
```

# :trident: Scenario 04 - Consumption control 
___
**Remember: All required files are in the folder */root/kcdlondon/lab/scenario04*. Please ensure that you are in this folder. You can do this with the command**
```console
cd /root/kcdlondon/lab/scenario04
```
___
There are many different ways to control the storage consumption. We will focus on the possibilities of K8s itself. However please remember: Sometimes the same thing can also be achieved at storage or csi driver level and it might be preferred to do it there.

You can create different objects to control the storage consumption directly in Kubernetes:

- LimitRange: controls the maximum (& minimum) size for each claim in a namespace
- ResourceQuotas: limits the number of PVC or the amount of cumulative storage in a namespace

For this scenario we will create and work in the namespace *control*.

You will create two types of quotas:

1. Limit the number of PVC a user can create
2. Limit the total capacity a user can consume

Take a look at _rq-pvc-count-limit.yaml_ and _rq-sc-resource-limit.yaml_ and then apply them:

```console
kubectl create namespace control
kubectl apply -n control -f rq-pvc-count-limit.yaml
kubectl apply -n control -f rq-sc-resource-limit.yaml
```

You can see the specified ressource quotas with the following command:

```console
kubectl get resourcequota -n control
```

Nice, they are there - but what do they do? Let's take a closer look:

```console
kubectl describe quota pvc-count-limit -n control
```

Ok we see some limitations... but how do they work? Let's create some PVCs to find out

```console
kubectl apply -n control -f pvc-quotasc-1.yaml
kubectl apply -n control -f pvc-quotasc-2.yaml
```

Again, have a look at the ressource limits:

```console
kubectl describe quota pvc-count-limit -n control
```

Two in use, great, let's add a third one

```console
kubectl apply -n control -f pvc-quotasc-3.yaml
```

So far so good, all created, a look at our limits tells you that you got the maximum number of PVC allowed for this storage class. Let's see what happens next...

```console
kubectl apply -n control -f pvc-quotasc-4.yaml
```

Oh! An Error... well that's what we expected as we want to limit the creation, right?
Before we continue, let's clean up a little bit:

```console
kubectl delete pvc -n control --all
```

Time to look at the capacity quotas...

```console
kubectl describe quota sc-resource-limit -n control
```

Each PVC you are going to use is 5GB.

```console
kubectl apply -n control -f pvc-5Gi-1.yaml
```

A quick check:

```console
kubectl describe quota sc-resource-limit -n control
```

Given the size of the second PVC file, the creation should fail in this namespace

```console
kubectl apply -n control -f pvc-5Gi-2.yaml
```

And as expected, our limits are working. 

Before starting the second part of this scenario, let's clean up

```console
kubectl delete pvc -n control 5gb-1
kubectl delete resourcequota -n control --all
```

We will use the LimitRange object type to control the maximum size of the volumes a user can create in this namespace. 

```console
kubectl apply -n control -f lr-pvc.yaml
```

Let's verify:

```console
kubectl describe -n control limitrange storagelimits
```

Now that we have create a 2Gi limit, let's try to create a 5Gi volume...

```console
kubectl apply -n control -f pvc-5Gi-1.yaml
```

Magical, right? By the way, the NetApp Trident CSI driver from this lab has a similar parameter called _limitVolumeSize_ that controls the maximum capacity of a PVC per Trident Backend. As we told you: sometimes there are multiple ways to achieve the same result. 
# :trident: Scenario 05 - About Generic Ephemeral Volumes
___
**Remember: All needed files are in the folder */root/kcdlondon/lab/scenario05*. Please ensure that you are in this folder. You can do this with the command**
```console
cd /root/kcdlondon/lab/scenario05
```
___
When talking about CSI drivers in K8s, we often refer to Persistent Volumes. It is indeed the most common use of such CSI driver. There are multiple benefits of using persistent volumes, one of them being that the volumes remains after the application is gone (ya, that is actually why it is called _persistent_).  

For some use cases, you may need a volume for your application to store files that are absolutely not important & can be deleted alongside the application when you dont need it anymore. That is where Ephemeral Volumes could be useful.

Kubernetes proposes different types of ephemeral volumes:

- emptyDir
- configMap
- CSI ephemeral volumes 
- **generic ephemeral volumes** 

This chapter focuses on the last category which was introduced as an alpha feature in Kubernetes 1.19 (Beta in K8S 1.21 & GA in K8S 1.23).  

The construct of a POD manifest with Generic Ephemeral Volumes is pretty similar to what you would see with StatefulSets, ie the volume definition is included in the POD object. This folder contains a simple busybox pod manifest. You will see that :

- a volume is created alongside the POD that will mount it
- when the POD is deleted, the volume follows the same path & disappears

First let's create our app:

```console
kubectl apply -f my-app.yaml
```

Now discover what has happened:

```console
kubectl get pod,pvc
```

Can we see the mount?

```console
kubectl exec my-app -- df -h /scratch
```

Can we write to it?

```console
kubectl exec my-app  -- sh -c 'echo "Hello ephemaral volume" > /scratch/test.txt'
```

Is the input really saved?

```console
kubectl exec my-app  -- more /scratch/test.txt
```

Nice. What happens if we delete the app now?

```console
kubectl delete -f my-app.yaml
kubectl get pod,pvc
```

Note that creating this kind of pod does not display a _pvc created_ message. 

# :trident: The End :trident:

Thank you for participating in this workshop. We hope it was fun and you've learned something. We tried to cover the basics, there is a lot more to learn and talk. If you want to discuss further, come to our booth or feel free to reach out to us online

Hendrik Land: [Linkedin](https://www.linkedin.com/in/hendrik-land/) / [E-Mail](mailto:hendrik.land@netapp.com)

Johannes Wagner: [Linkedin](https://www.linkedin.com/in/johwagner/) / [E-Mail](mailto:johannes.wagner@netapp.com)



