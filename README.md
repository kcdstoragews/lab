# Everything you always wanted to know about storage in Kubernetes (But were too afraid to ask)

# Introduction


You will work in the NetApp Lab on Demand Environment, but we`ve prepared some more resources for you than the typical Lab Guide offers. Before we can start, there are some preparations to do.

You will connect to a Windows Jumphost from which you can access the training environment.  
It will provide you a K8s Cluster Version 1.22.3, a preconfigures storage system and the CSI driver we will use to demonstrate you how easy it is to use persistent storage in K8s.

## Preparation

1. Plesase choose a Username from the following Document:  
https://hierstehtdanneingoogledoc

2. Access the lab environment:  
https://lod-bootcamp.netapp.com

3. Request the Lab *Using Trident with Kubernetes and ONTAP v5.0* and connect to the jumphost

4. Open *Putty* and connect the the Host *rhel3* with the following credentials:  
username: *root*   
password: *Netapp1!*

5. We've prepared some exercises for you that are hosted in this github repo. To have them available on your training environment, please clone it with the following command:

        git clone https://github.com/tobedone/kcdlondonworkshop 

    You should now have several directories available. Wherever we need files for a scenario, they are placed in the associated folder with the same name. 

6. As this lab is used for different things and has a lot of stuff in it that might be confusing, please run this little cleanup script which removes all things we don't need in our workshop
________
=> All in One Script von Yves recyclen?
________

        sh cleanup.sh

Ok, we are ready to go, have fun with the workshop


## Scenario 01 - storage classes, persistent volumes & persistent volume claims
In this scenario, you will create two storage classes, discovery their capabilities, create pvcs and do some basic troubleshooting. 

We are using NetApp Astra Trident in this lab. It is running in the namespace *trident*.
The backends in this environment are allready created. Take a short time to review them:

    kubectl get tbe -n trident

First let's create two storage classes. We've prepared the necessary file already in the folder. There is one storage class prepared for the nas backend and one for san.

Review them and apply them afterwards.
________
=> todo: copy the storageclasses sc-csi-ontap-nas.yaml (SC2) and backend-san-eco-default.yaml(SC5)
___________


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
