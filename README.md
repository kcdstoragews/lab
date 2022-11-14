# Everything you always wanted to know about storage in Kubernetes (But were too afraid to ask)

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
**Remember All needed files are in the folder /root/kcdlondon/lab/scenario01 please ensure that you are in this folder now you can do this with the command "cd root/kcdlondon/lab/scenario01"**
____
In this scenario, you will create two storage classes, discovery their capabilities, create pvcs and do some basic troubleshooting. 
### 1. Backends and storage classes
You are using NetApp Astra Trident in this lab. It is running in the namespace *trident*.
The backends in this environment are allready created. Take a short time to review them:

    kubectl get tbe -n trident

First let's create two storage classes. We've prepared the necessary file already in the folder. There is one storage class prepared for the nas backend and one for san.

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
1. This storace class will be the default in this cluster (look at annotations)
2. NetApp Astra Trident is responsible for the provisioning of PVCs with this storage class (look at provisioner)
3. There are some parameters needed for this provisioner. In our case we have to tell them the backend type and also where to create the volumes
4. This volume could be expanded after it's creation.

Now let's compare with the one for san:

    cat sc-csi-ontap-san-eco.yaml

You get a quiet similar output here:

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: storage-class-san-economy
    provisioner: csi.trident.netapp.io
    parameters:
      backendType: "ontap-san-economy"
      storagePools: "san-eco:aggr1"
      fsType: "ext4"
    mountOptions:
      - discard
    reclaimPolicy: Retain
    allowVolumeExpansion: true

The biggest differences are: 
1. As you are now asking for a block device, you will need a file system on it to make a usage possible. *ext*4 is specified here (look at fsType in parameters Section)
2. A reclaim policy is specified. What this means will be tested in a scenario later.

If you want to dive into the whole concept of storage classes, this is well documented here: https://kubernetes.io/docs/concepts/storage/storage-classes/

After all this theory, let's just add the storace classes to your cluster:

     kubectl create -f sc-csi-ontap-nas.yaml
     kubectl create -f sc-csi-ontap-san-eco.yaml 

You can discover all existing storage clusters with a simple command:

    kubectl get sc

If you want to see more details a *describe* will show you way more details. Let's do this

    kubectl describe sc storage-class-nas

The output shows you all details. Remember, we haven't specified a *reclaimPolicy*. Therefore the default vaule of *delete* could be observed in the output. Again, we will find out what the difference is, a little bit later.

### 2. PVCs & PVs

As your cluster has now a CSI driver installed, specified backends and also ready to use storage classes, you are set to ask for storage. But don't be afraid. You will not have to open a ticket at your storage guys or do some weird storage magic. We want a persistent volume, so let's claim one.  
This is done with a so called persistent volume claim. 

There are two files in your scenario01 folder, *firstpvc.yaml* and *secondpvc.yaml* both a requesting a 5GiB Volume, now let's get this storage into a namespace we create first and call it *funwithpvcs*...

    kubectl create namespace funwithpvcs
    kubectl apply -f firstpvc.yaml -n funwithpvcs
    kubectl apply -f secondpvc.yaml -n funwithpvcs



## Scenario 02 - running out of space? Let's expand the volume
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


## Scenario 03 snapshots, clones etc

Szenario 13

## Scenario 04 Deployments, Stateful sets etc

Szenario 11

## Scenario 05

## Scenario 04
