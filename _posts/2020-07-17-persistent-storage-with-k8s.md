---
layout: post
title: Persistent storage with kubernetes
subtitle:
tags: [containers]
---

Do you think storage is complicated in kubernetes, yes it is.

PV, PVC, StorageClass sounds unfamiliar to you ? I hope this post will help you.

![Storage img](https://i.ytimg.com/vi/OulmwTYTauI/maxresdefault.jpg)


## Storage components in a cluster

**PV** : Persistent volume is like a logical volume in LVM which is linked to the physical storage.  It's a resource created manually by administrator or dynamically with storageclass (don't pay attention about dynamic provisionning for now, I'll discusss about it later). A PV is a cluster-wide resource.

**PVC** : A Persistent volume claim consumed the storage in the PV and is requested by a user . You ask for a space size, and PVC is gonna get this space on PV. A PVC is a resource which resides in a namespace.

PV and PVC works together. One *provide* the storage, the other *request* the storage.

Kubernetes provides two ways to manage persistent volumes: *static* and *dynamic*

### Static Provisioning

Static provisioning is when an administrator define the PV beforehand, then create the PVC to consume the storage defined in the PV.

Here the process to attach a volume to a pod

1. Create manually the PV(s) with the storage capacity and access mode desired.

2. Create a PVC with the storage size we want to request from the PV

3. Lastly, we configure the PVC in the pod specification to uses the storage.

### Example of static provisionning

*Creation of the PV with 10Gi of storage using hostPath type*

```bash
apiVersion: v1  
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```


*Creation of the PVC and define a 3G of storage to be request to the PV defined above*


```
kind: PersistentVolumeClaim

apiVersion: v1  
metadata:
  name: task-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
 
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim  #3 Specify the pvc name defined in metadata field (section above)
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```


### Dynamic Provisioning


To avoid to create the PV beforehand, k8s introduced the concept of *StorageClass*

Using dynamic provisioning reduces administrative overhead involved in manually creating PV. Automatically allocating and deallocating PV in response to persistent volume claims can help to reduce wasted storage that is allocated but never used.

A StorageClass defines the parameters we want for the dynamic PV creation:

- provisionner (NFS, GCE PD, Azure Disk, EBS..)

- Mount options

- parameters: Depends on the provisioner type. Could be type of volume, type of filesystem, encryption)

To allow dynamic provisioning of a PV, we specify the storage class in the PVC. 
Specify a storage class will trigger the dynamic provisioning of the PV. So we don't need to create the PV in advance.

Here the process to attach a volume to a pod:

1. Create the storageClass first (GCE-PD, AzureDisk, AWS EBS) 

2. Create the PVC and reference the storageclass created before

3. Then, we configure the PVC in the pod specification to use it. PV is gonna be dynamically created using the provisioner defined in the storageClass.


### Example of dynamic provisionning

*Creation of a storageClass for GCE persistent disk*

```
apiVersion: storage.k8s.io/v1 
kind: StorageClass
metadata:
  name: my-storageclass
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

*Create the PVC, define the desired request storage, reference the storageclass name defined above*

```
kind: PersistentVolumeClaim
apiVersion: v1  
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: my-storageclass
  resources:
    requests:
      storage: 30Gi
```

*Specify the PVC name defined in metadata field*


```
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: claim1
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

After applying these changes, a new PV will be created and the pod will use it.


A PV can have different *status* and *reclaim policies*, they are here to define the lifecycle of a volume :



# Conclusion

Storage is a whole topic itself, and you defintely need expertise before using a database with persistent volumes.

I skept status and reclaim policy as it would bring more complexity to this post.

I would recommend to avoid using persistent storage for the first workloads that you are going to create/migrate and put stateless applications first.

Then, when you'll need, use dynanic volume provisionning instead of the static way for an easier management.



<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "NewsArticle",
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": "https://ptran32.github.io/2020-07-17-persistent-storage-with-k8s/"
  },
  "headline": "Persistent storage with kubernetes",
  "description": "How Persistent storage works in kubernetes. Persistent volume and persistent volume claim",
  "image": "https://i.ytimg.com/vi/OulmwTYTauI/maxresdefault.jpg",  
  "author": {
    "@type": "Person",
    "name": "Patrice"
  },  
  "publisher": {
    "@type": "Organization",
    "name": "Patrice",
    "logo": {
      "@type": "ImageObject",
      "url": ""
    }
  },
  "datePublished": "2020-07-17",
  "dateModified": "2020-07-17"
}
</script>