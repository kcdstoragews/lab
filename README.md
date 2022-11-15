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


## :trident: Scenario 01 - storage classes, persistent volumes & persistent volume claims 
____
**Remember All needed files are in the folder */root/kcdlondon/lab/scenario01* please ensure that you are in this folder now you can do this with the command "*cd /root/kcdlondon/lab/scenario01*"**
____
In this scenario, you will create two StorageClasses, discovery their capabilities, create pvcs and do some basic troubleshooting. 
### 1. Backends and StorageClasses
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

### 2. PVCs & PVs

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
Sometimes you need more space than you thought before. For sure you could create a new volume, copy the data and work with the new bigger one but it is way easier, to just expand the existing.

First lets's create a PVC and a Centos POD using this PVC, in their own namespace.

    kubectl create namespace resize
    kubectl create -n resize -f pvc.yaml

Review your created pvc

    kubectl -n resize get pvc,pv

As you can see, the created pv has 5GiB and is bound to the pvc *pvc-to-resize-file*

    NAME                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
    persistentvolumeclaim/pvc-to-resize-file   Bound    pvc-7eeea3f7-1bea-458b-9824-1dd442222d55   5Gi        RWX            storage-class-nas   2s
    NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS        REASON   AGE
    persistentvolume/pvc-7eeea3f7-1bea-458b-9824-1dd442222d55   5Gi        RWX            Delete           Bound    resize/pvc-to-resize-file   storage-class-nas            1s

Now let's create a pod that has the pvc mounted

    kubectl create -n resize -f pod-busybox-nas.yaml
pod/busyboxfile created

$ kubectl -n resize get pod --watch
NAME          READY   STATUS              RESTARTS   AGE
busyboxfile   0/1     ContainerCreating   0          5s
busyboxfile   1/1     Running             0          15s


## :trident: Scenario 03 snapshots, clones etc 

Szenario 13

## :trident: Scenario 04 Deployments, Stateful sets etc 

Szenario 11

## :trident: Scenario 05 

## :trident: Scenario 04 
