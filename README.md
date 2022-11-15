# :trident: Everything you always wanted to know about storage in Kubernetes (But were too afraid to ask) 

# Introduction


You will work in the NetApp Lab on Demand Environment, but we`ve prepared some more resources for you than the typical Lab Guide offers. Before we can start, there are some preparations to do.

You will connect to a Windows Jumphost from which you can access the training environment.  
It will provide you a K8s Cluster Version 1.22.3, a preconfigures storage system and the CSI driver we will use to demonstrate you how easy it is to use persistent storage in K8s.

## Preparation

1. Plesase choose a Username from the following Document:  
https://tinyurl.com/kcdlondon

2. Access the lab environment:  
https://lod-bootcamp.netapp.com

3. Request the Lab *Using Trident with Kubernetes and ONTAP v5.0* and connect to the jumphost

4. Open *Putty* and connect the the Host *rhel3* with the following credentials:  
username: *root*   
password: *Netapp1!*

5. We've prepared some exercises for you that are hosted in this github repo. To have them available on your training environment, please clone it with the following commands:

        cd /root
        mkdir kcdlondon
        cd kcdlondon
        git clone https://github.com/kcdstoragews/lab

    You should now have several directories available. Wherever you need files for a scenario, they are placed in the associated folder with the same name. 

6. As this lab is used for different things and has a lot of stuff in it that might be confusing, please run this little cleanup script which removes all things we don't need in our workshop, updates the environment to a recent version and creates some necessary stuff.   
Please run the following commands:

       cd /root/kcdlondon/lab/prework
       sh prework.sh


# :trident: Scenario 01 - storage classes, persistent volumes & persistent volume claims 
____
**Remember All needed files are in the folder */root/kcdlondon/lab/scenario01* please ensure that you are in this folder now you can do this with the command "*cd /root/kcdlondon/lab/scenario01*"**
____
In this scenario, you will create two StorageClasses, discovery their capabilities, create pvcs and do some basic troubleshooting. 
## 1. Backends and StorageClasses
You are using NetApp Astra Trident in this lab. It is running in the namespace *trident*.
The backends in this environment are allready created. Take a short time to review them:

    kubectl get tbc -n trident

First let's create two StorageClasses. We've prepared the necessary file already in the folder. There is one storage class prepared for the nas backend and one for san.

The file you will use for nas is called *sc-csi-ontap-nas.yaml*  
The command...

    cat sc-csi-ontap-nas.yaml 

...will provide you the following output of the file:

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: storage-class-nas
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: csi.trident.netapp.io
    parameters:
      backendType: "ontap-nas"
      storagePools: "nas-default:aggr1"
    allowVolumeExpansion: true 

You can see the following things:
1. This StoraceClass will be the default in this cluster (look at annotations)
2. NetApp Astra Trident is responsible for the provisioning of PVCs with this storage class (look at provisioner)
3. There are some parameters needed for this provisioner. In our case we have to tell them the backend type and also where to create the volumes
4. This volume could be expanded after it's creation.

Now let's compare with the one for san:

    cat sc-csi-ontap-san.yaml

You get a quiet similar output here:

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: storage-class-san
    provisioner: csi.trident.netapp.io
    parameters:
      backendType: "ontap-san"
      storagePools: "san-secured:aggr1"
      fsType: "ext4"
    mountOptions:
       - discard
    reclaimPolicy: Retain
    allowVolumeExpansion: true

The biggest differences are: 
1. As you are now asking for a block device, you will need a file system on it to make a usage possible. *ext*4 is specified here (look at fsType in parameters Section)
2. A reclaim policy is specified. What this means will be explained.

If you want to dive into the whole concept of storage classes, this is well documented here: https://kubernetes.io/docs/concepts/storage/storage-classes/

After all this theory, let's just add the StoraceClasses to your cluster:

     kubectl create -f sc-csi-ontap-nas.yaml
     kubectl create -f sc-csi-ontap-san.yaml 

You can discover all existing StorageClasses with a simple command:

    kubectl get sc

If you want to see more details a *describe* will show you way more details. Let's do this

    kubectl describe sc storage-class-nas

The output shows you all details. Remember, we haven't specified a *reclaimPolicy*. Therefore the default vaule of *Delete* could be observed in the output. Again, we will find out what the difference is, a little bit later.

## 2. PVCs & PVs

As your cluster has now a CSI driver installed, specified backends and also ready to use StorageClasses, you are set to ask for storage. But don't be afraid. You will not have to open a ticket at your storage guys or do some weird storage magic. We want a persistent volume, so let's claim one.  
The workflow isn't complex but important to understand. As we are using Trident in this lab, we used it also for describing the workflow. However the workflow is pretty similar in all other CSI drivers.

1. A user creates a PersistentVolumeClaim requesting a new PersistentVolume of a particular size from a Kubernetes StorageClass that was previously configured by someone.
2. The Kubernetes StorageClass identifies Trident as its provisioner and includes parameters that tell Trident how to provision a volume for the requested class.
3. Trident looks at its own StorageClass with the same name that identifies the matching Backends and StoragePools that it can use to provision volumes for the class.
4. Trident provisions storage on a matching backend and creates two objects: a PersistentVolume in Kubernetes that tells Kubernetes how to find, mount, and treat the volume, and a volume in Trident that retains the relationship between the PersistentVolume and the actual storage.
5. Kubernetes binds the PersistentVolumeClaim to the new PersistentVolume. Pods that include the PersistentVolumeClaim mount that PersistentVolume on any host that it runs on.

There are two files in your scenario01 folder, *firstpvc.yaml* and *secondpvc.yaml* both a requesting a 5GiB Volume, now let's get this storage into a namespace we create first and call it *funwithpvcs*...

    kubectl create namespace funwithpvcs
    kubectl apply -f firstpvc.yaml -n funwithpvcs
    kubectl apply -f secondpvc.yaml -n funwithpvcs

Kubernetes provides the output, that both persistent volume claimes have been created. Great... or not? Let's have a look

    kubectl get pvc -n funwithpvcs

You can see that the PVC named *firstpvc* has a volume, and is in status *Bound*. The PVC with the name *secondpvc* looks not that healthy, it is still in Status *Pending*. This means that the request is ongoing, K8s tries to get what you want, but for whatever reason it doesn't work.   
Lucky that we can describe objects and see what happens!

    kubectl describe pvc secondpvc -n funwithpvcs

Ok we can see, that there is an issue with the StorageClass. But why?  
Everything that is requested in the PVC will be handed over to the provisioner that is defined in the StorageClass. In this case Trident gets a request for a RWX volume with 5GiB and for the backend "ontap-san-economy".   
In contrast to K8s, the CSI Driver is aware what is possible and what not. It recognizes that a RWX volume isn't possible at this backend type as this backend can only serve ROX and RWO. 

If you want to have your second pvc also running and still need RWX access mode, we have to modify the yaml file. Just switch the storage class to *storage-class-nas*. This StorageClass has a backend type that is able to do RWX. Unfortunately a lot of things in a PVC are immutable after creation so before we can see whether your change is working or not, you have to delete the pvc again.

    kubectl delete -f secondpvc.yaml -n funwithpvcs

After you have deleted the PVC, changed the StorageClass in the pvc file and applied it again, you should see that both pvcs are now bound.

    kubectl get pvc -n funwithpvcs

    NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
    firstpvc    Bound    pvc-eb00c989-fddb-4224-aa56-a8918064b9fb   5Gi        RWO            storage-class-san           16m
    secondpvc   Bound    pvc-50f6c56b-3575-43b3-ae16-5b99b35d9a59   5Gi        RWX            storage-class-nas           8s

Earlier we mentioned that a *PersistentVolume* is also created. Maybe you ask yourself where to see them. It is pretty easy, let's have a look at our recently created ones:

    kubectl get pv -n funwithpvcs

    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS        REASON   AGE
    pvc-0e860943-3fae-4dae-80e9-82e2259089c8   5Gi        RWX            Delete           Bound    funwithpvcs/secondpvc   storage-class-nas            99s
    pvc-4adaf625-fc86-42e8-a481-92b02addfdbc   5Gi        RWO            Retain           Bound    funwithpvcs/firstpvc    storage-class-san            2m43s


You remember the ReclaimPolicy we definied in our StorageClass? We can see here that one PV has an other ReclaimPolicy than the other. Let's delete both PVCs and see what will happen

    kubectl delete -f firstpvc.yaml -n funwithpvcs
    kubectl delete -f secondpvc.yaml -n funwithpvcs

Let's have a look how PVCs and PVs look now in our namespace

    kubectl get pvc -n funwithpvcs
    kubectl get pv -n funwithpvcs

Magic, both PVCs are gone (well... we advised k8s to remove them...) but one PV is still there? No not real magic, just the normal behaviour of the specified ReclaimPolicy. As told before, the default ReclaimPolicy is *Delete*. This means as soon as the corresponding PVC is deleted, the PV will be deleted too. In some use cases this would delete valuable data. To avoid this, you can set the ReclaimPolicy to *Retain*. If the PVC is deleted now, the PV will change its Status from *Bound* to *Released*. The PV could be used again.  

Awesome, you are now able to request storage...but as long as no appliaction is using that, there is no real sense of having persistent storage. Let's create an application that is able to do something with the storage. For this purpose we will use *Ghost* a light weight web portal. There are some files in our scenario01 directory:

- ghost-pvc.yaml to manage the persistent storage of this app
- ghost-deployment.yaml that will define how to manage the app
- ghost-service.yaml to expose the app

You are going to create this app in its own namespace which will be *ghost*. You will use the created StorageClass for nas.

    kubectl create namespace ghost

Have a look at the file for the pvc and create it afterwards:

    kubectl create -f ghost-pvc.yaml -n ghost


Now as the PVC is there, Have a look at the file for the deployment and create it afterwards:

    kubectl create -f ghost-deploy.yaml -n ghost

We have an app, we have storage for the app, to access it we finally need a service:

    kubectl create -f ghost-service.yaml -n ghost

You can see a summary of what you've done with the following command:

    kubectl get -n ghost all,pvc,pv

It takes about 40 seconds for the POD to be in a running state The Ghost service is configured with a NodePort type, which means you can access it from every node of the cluster on port 30080. Give it a try ! => http://192.168.0.63:30080

Let's see if the */var/lib/ghost/content* folder is indeed mounted to the NFS PVC that was created.

     kubectl exec -n ghost $(kubectl -n ghost get pod -o name) -- df /var/lib/ghost/content

You should be able to see that it is mounted. Let's have a look into the folder

    kubectl exec -n ghost $(kubectl -n ghost get pod -o name) -- ls /var/lib/ghost/content

The data is there, perfect. If you want to, you can easily clean up a little bit:

    kubectl delete ns ghost

## :trident: Scenario 02 - running out of space? Let's expand the volume 
____
**Remember All needed files are in the folder */root/kcdlondon/lab/scenario02* please ensure that you are in this folder now you can do this with the command "*cd /root/kcdlondon/lab/scenario02*"**
____
Sometimes you need more space than you thought before. For sure you could create a new volume, copy the data and work with the new bigger one but it is way easier, to just expand the existing.

First let's check the StorageClasses

    kubectl get sc 

Look at the column *ALLOWVOLUMEEXPANSION*, as we specified earlier, both StorageClasses are set to *true* which means, pvcs that are created with this StorageClass could be resized.  
NFS Resizing was introduced in K8S 1.11, while iSCSI resizing was introduced in K8S 1.16 (CSI)

Now let's create a PVC & a Centos POD using this PVC, in their own namespace.

    kubectl create namespace resize
    kubectl create -n resize -f pvc.yaml
    kubectl create -n resize -f pod-busybox-nas.yaml

Wait until the pod is in running state - you can check this with the command

    kubectl get pod -n resize

Finaly you should be able to see that the 5G volume is indeed mounted into the POD

    kubectl -n resize exec busyboxfile -- df -h /data

Resizing a PVC can be done in different ways. We will here edit the definition of the PVC & manually modify it.  
Look for the *storage* parameter in the spec part of the definition & change the value (here for the example, we will use 15GB)
The provided command will open the pvc definition.

    kubectl -n resize edit pvc pvc-to-resize-file

change the size to 15Gi like in this example:

    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 15Gi
      storageClassName: storage-class-nas
      volumeMode: Filesystem

you can insert something by pressing "i", exit the editor by pressing "ESC", type in :wq! to save&exit. 

Everything happens dynamically without any interruption. The results can be observed with the following commands:

    kubectl -n resize get pvc
    kubectl -n resize exec busyboxfile -- df -h /data

This could also have been achieved by using the _kubectl patch_ command. Try the following one:

```bash
kubectl patch -n resize pvc pvc-to-resize-file -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

If you want to, clean up a little bit

    kubectl delete namespace resize


# :trident: Scenario 03 -  snapshots, clones etc 
___
**Remember All needed files are in the folder */root/kcdlondon/lab/scenario03* please ensure that you are in this folder now you can do this with the command "*cd /root/kcdlondon/lab/scenario03*"**
___
CSI Snapshots have been promoted GA with Kubernetes 1.20.  
While snapshots can be used for many use cases, we will see here 2 different ones, which share the same beginning:

- Restore the snapshot in the current application
- Create a new POD which uses a PVC created from the snapshot

There is also a chapter that will show you the impact of deletion between PVC, Snapshots & Clones (spoiler alert: no impact).  

We would recommended checking that the CSI Snapshot feature is actually enabled on this platform.  

This [link](https://github.com/kubernetes-csi/external-snapshotter) is a good read if you want to know more details about installing the CSI Snapshotter.  
The **CRD** & **Snapshot-Controller** to enable this feature have already been installed in this cluster. Let's see what we find:

```bash
kubectl get crd | grep volumesnapshot
```
will show us the crds
```bash
volumesnapshotclasses.snapshot.storage.k8s.io         2020-08-29T21:08:34Z
volumesnapshotcontents.snapshot.storage.k8s.io        2020-08-29T21:08:55Z
volumesnapshots.snapshot.storage.k8s.io               2020-08-29T21:09:13Z
```
```bash
kubectl get all -n snapshot-controller
```
will show us all ressources in the namespace snapshot-controller
```bash
NAME                                      READY   STATUS    RESTARTS         AGE
pod/snapshot-controller-bb7675d55-7jctt   1/1     Running   12 (6d18h ago)   203d
pod/snapshot-controller-bb7675d55-qwfns   1/1     Running   14 (6d18h ago)   203d

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/snapshot-controller   2/2     2            2           203d

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/snapshot-controller-bb7675d55   2         2         2       203d
```

Aside from the 3 CRD & the Controller StatefulSet, the following objects have also been created during the installation of the CSI Snapshot feature:

- serviceaccount/snapshot-controller
- clusterrole.rbac.authorization.k8s.io/snapshot-controller-runner
- clusterrolebinding.rbac.authorization.k8s.io/snapshot-controller-role
- role.rbac.authorization.k8s.io/snapshot-controller-leaderelection
- rolebinding.rbac.authorization.k8s.io/snapshot-controller-leaderelection

Finally, you need to create a _VolumeSnapshotClass_ object that points to the Trident driver.

```bash
kubectl create -f sc-volumesnapshot.yaml
```
You can see this *VolumeSnapshotClass* with the following command:

```bash
kubectl get volumesnapshotclass
```

Note that the _deletionpolicy_ parameter could also be set to _Retain_.

The _volume snapshot_ feature is now ready to be tested.

The following will lead you in the management of snapshots with a simple lightweight container BusyBox.

We've prepared all the necessary files for you to save a little time. Please prepare the environment with the following commands:

```bash
kubectl create namespace busybox
kubectl create -n busybox -f busybox.yaml
kubectl get -n busybox all,pvc
```
The last line will provide you an output of what you have done before. There should be one running pod and a pvc with 10Gi.

Before you create a snapshot so, let's create a file in our PVC, that will be deleted once the snapshot is created.  
That way, there is a difference between the current filesystem & the snapshot content.  

```bash
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- sh -c 'echo "KCD UK 2022 are fun" > /data/test.txt'
```
This creates the text file test.txt and enter the text *KCD UK 2022 are fun" into the file. You can show yourself the file with the following command:

```bash
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- more /data/test.txt
```

Creating a snapshot of this volume is very simple:

```bash
kubectl create -n busybox -f pvc-snapshot.yaml
```
After it is created you can observe its details:
```bash
kubectl get volumesnapshot -n busybox
```
Your snapshot has been created !  

To see an effect, you should delete the test.txt now
```bash
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- rm -f /data/test.txt
```
If you want to verify, that the data is really gone, feel free to try out the command from above that has shown you the text of the file:

```bash
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- more /data/test.txt
```

One of the useful things of a snapshot is, that you can create a clone from this snapshot. 
If you take a look a the PVC manifest (_pvc_from_snap.yaml_), you can notice the reference to the snapshot:

```bash
  dataSource:
    name: mydata-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

Let's see how that turns out:

```bash
$ kubectl create -n busybox -f pvc_from_snap.yaml
```
This will create a new pvc which could be used instantly in an application. You can see it if you have a look at the pvcs in your namespace:

    kubectl get pvc -n busybox

Recover the data of your application

When it comes to data recovery, there are many ways to do so. If you want to recover only one file, you could browser through the .snapshot folders & copy/paste what you need. However, if you want to recover everything, you could very well update your application manifest to point to the clone, which is what we are going to see:

```bash
kubectl patch -n busybox deploy busybox -p '{"spec":{"template":{"spec":{"volumes":[{"name":"volume","persistentVolumeClaim":{"claimName":"mydata-from-snap"}}]}}}}'
```

That will trigger a new POD creation with the updated configuration

Now, if you look at the files this POD has access to (the PVC), you will see that the *lost data* (file: test.txt) is back!

```bash
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- ls -l /data/
```
or even better, lets have a look at the text:
```bash
kubectl exec -n busybox $(kubectl get pod -n busybox -o name) -- more /data/test.txt
```
Tadaaa, you have restored your data!  
Keep in mind that some applications may need some extra care once the data is restored (databases for instance).  
& that is why NetApp Astra is taking care of !

As in every scenario, a little clean up at the end:
```bash
$ kubectl delete ns busybox
```

# :trident: Scenario 04 - Consumption control 
___
**Remember All needed files are in the folder */root/kcdlondon/lab/scenario04* please ensure that you are in this folder now you can do this with the command "*cd /root/kcdlondon/lab/scenario04*"**
___
There are many different areas you can control the consumption. We will concentrate for now on the possibilities of K8s. However please remember: Sometimes the things could also be achived at storage or csi driver level, even such things, that are not possible in K8s.

You can create different objects to control the storage consumption directly in Kubernetes:

- LimitRange: controls the maximum (& minimum) size for each claim in a namespace
- ResourceQuotas: limits the number of PVC or the amount of cumulative storage in a namespace

For this scenario we will create and work in the namespace *control*.

You will create two types of quotas:

1. limit the number of PVC a user can create
2. limit the total capacity a user can create  

    kubectl create namespace control
    kubectl create -n control -f rq-pvc-count-limit.yaml
    kubectl create -n control -f rq-sc-resource-limit.yaml

You can see the specified ressource quotas with the following command:

    kubectl get resourcequota -n control

Nice they are there but what do they do? Let's have closer look:

    kubectl describe quota pvc-count-limit -n control

Ok we see some limitations... but how do they work? Let's create some PVCs to find out

    kubectl create -n control -f pvc-quotasc-1.yaml
    kubectl create -n control -f pvc-quotasc-2.yaml

Again, have a look at the ressource limits:

    kubectl describe quota pvc-count-limit -n control

2 in use, great, let's add a third one

    kubectl create -n control -f pvc-quotasc-3.yaml

So far so good, all created, a look at our limits should tell you that you got the maximum number of PVC allowed for this storage class. Let's see what happens next...

    kubectl create -n control -f pvc-quotasc-4.yaml

Oh! An Error...n well that's what we expected as we want to limit the creation, right?
Before we continue, let's clean up a little bit:

    kubectl delete pvc -n control --all

Time to look at the capacity quotas

    kubectl describe quota sc-resource-limit -n control

Each PVC you are going to use is 5GB.

    kubectl create -n control -f pvc-5Gi-1.yaml

A short control:

    kubectl describe quota sc-resource-limit -n control

Seeing the size of the second PVC file, the creation should fail in this namespace

    kubectl create -n control -f pvc-5Gi-2.yaml

And as expected, our limits are working. 

Before starting the second part of this scenario, let's clean up

    kubectl delete pvc -n control 5gb-1
    kubectl delete resourcequota -n control --all

We will use the LimitRange object type to control the maximum size of the volumes we create in a namespace. However, you can also decide to use this object type to control compute & memory limits.

    kubectl create -n control -f lr-pvc.yaml

Let's investigate what you've done

    kubectl describe -n control limitrange storagelimits

Now that we have create a 2Gi limit, let's try to create a 5Gi volume, operation that should fail.

    kubectl create -n control -f pvc-5Gi-1.yaml

Magical, right? By the way, the used CSI Driver NetApp Trident has a similar parameter called _limitVolumeSize_ that controls the maximum capacity of a PVC per Trident Backend. As we told you: sometimes there are more ways than just one. 
# :trident: Scenario 05 - About Generic Ephemeral Volumes
___
**Remember All needed files are in the folder */root/kcdlondon/lab/scenario05* please ensure that you are in this folder now you can do this with the command "*cd /root/kcdlondon/lab/scenario05*"**
___
When talking about Trident, we often refer to Persistent Volumes. It is indeed the most common use of such CSI driver. There are multiple benefits of using persistent volumes, one of them being that the volumes remains after the application is gone (ya, that is actually why it is called _persistent_).  

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

    kubectl create -f my-app.yaml

Now discover what has happened:

    kubectl get pod,pvc

Can we see the mount?

    kubectl exec my-app -- df -h /scratch

Can we write to it?

    kubectl exec my-app  -- sh -c 'echo "Hello ephemaral volume" > /scratch/test.txt'

Is the input really saved?

    kubectl exec my-app  -- more /scratch/test.txt

Nice. What happens if we delete the app now?

    kubectl delete -f gev.yaml
    kubectl get pod,pvc

Note that creating this kind of pod does not display a _pvc created_ message. 

# :trident: The End :trident:

Thank you for participating in this workshop we hope it was fun and you've learned something. We tried to cover the basics, there is a lot more to learn and talk. If you want to discuss further, come to our booth or feel free to reach out to us digital

Hendrik Land: [Linkedin](https://www.linkedin.com/in/hendrik-land/) / [E-Mail](mailto:hendrik.land@netapp.com)

Johannes Wagner: [Linkedin](https://www.linkedin.com/in/johwagner/) / [E-Mail](mailto:johannes.wagner@netapp.com)



