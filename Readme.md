# Fun with K8s Storage (and FTP)
In this repository I explore on a) how to install an FTP server on GKE and b) how to share storage accross pods (in the same and different namespaces).

**This is just a try-out - I do not go into all details and do not focus on security aspects. - So please don't just copy&paste**


# Keep in mind
Before we start, let's keep the following rules in mind:
- A PVC is exclusively bound to a PV.
You cannot bind 2 PVCs to the same PV.
- A PVC can be used by multiple pods, but only in the same namespace, as the PVC is a namespaced resource.
- If you use NFS & you want to like multiple PV/PVC on 1 NFS export, use Dynamic Provisioning. 
- A NFS Provisioner will create a Storage Class and the PVs dynamically as a directory of the nfs export.
- You can deploy an NFS provisioner in your cluster, the NFS provisioner(pod) will have access to the NFS folder(hostpath), and each time a pod requests a volume, the NFS provisioner will mount it in the NFS directory on behalf of the pod.
- Passive-mode FTP requires multiple high-level ports to be accessible from the client side. In this example I use a Kubernetes Service of type LoadBalancer to let GKE automatically provision the External Network GCLB (Layer 4) and according Firewall rules. In the service manifest I specify port 21 for the controle connection and 4 other ports for the data connection. In the corresponding deployment.yaml of the ftp server I use environment variables to set the min and max port range for the ftp server.

# Overview
This repository contains 3 namespaces:
- **ns1:** Creates 2 different deployments which access the same PVC for a NFS share in RWM mode and therefore can read/write the same share. The deployments access the root path of the NFS share and can also see the directories created by the apps in ns2 and ns3.
- **ns2 & ns3:** 


# Create Filestore
```
gcloud services enable file.googleapis.com
FSID=ftptest-filestore
FSNAME=ftptest_share
PROJECT=hewagner-anthos-sse
ZONE=us-central1-a
NETWORK=default
```

## Create Filestore instance with 1TB (=minimum size)
```
gcloud filestore instances create ${FSID} \
    --project=${PROJECT} \
    --zone=${ZONE} \
    --tier=STANDARD \
    --file-share=name=$FSNAME,capacity=1TB \
    --network=name=$NETWORK
```

## Get Filestore IP address
```
FSADDR=$(gcloud filestore instances describe ${FSID} \
     --project=${PROJECT} \
     --zone=${ZONE} \
     --format="value(networks.ipAddresses[0])")
echo $FSADDR
```

## Create an instance of NFS-Client Provisioner 
...connected to the Filestore instance you created earlier via its IP address (${FSADDR}). The NFS-Client Provisioner creates a new storage class: nfs-client. Persistent volume claims against that storage class will be fulfilled by creating persistent volumes backed by directories under the /ftptest_share directory on the Filestore instance's managed storage.
https://github.com/helm/charts/tree/master/stable/nfs-client-provisioner 

```
helm install nfs-cp stable/nfs-client-provisioner --set nfs.server=${FSADDR} --set nfs.path=/ftptest_share --set storageClass.accessModes=ReadWriteMany
watch kubectl get po -l app=nfs-client-provisioner
```

# Deploy FTP Server(s) and Reader pods
```
kubectl apply -f ns1,ns2,ns3
```

## Get ext. IP of ftp services
```
SVC2IP=$(kubectl get service ftptest-service -n ns2 -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')

SVC3IP=$(kubectl get service ftptest-service -n ns3 -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo $SVC2IP
echo $SVC3IP
```

# Access FTP server
Open Filezilla and connect with: 
HOST = $SVC2IP
Username = hewa
Password = hewagner12345
Port = 21

Create folders and upload some files.
Do the same for $SVC3IP.

## Check if you can read the files from another pod
```
PODNAME2=$(kubectl get pods -n ns2 --selector=app=reader -o jsonpath='{.items[0].metadata.name}')
kubectl exec $PODNAME2 --namespace ns2 -c reader -ti sh
```

## show files which where uploaded via filezilla
```
cd home/test/hewa
ls
```

Repeat the same for ns 3
```
PODNAME3=$(kubectl get pods -n ns3 --selector=app=reader -o jsonpath='{.items[0].metadata.name}')
kubectl exec $PODNAME3 --namespace ns3 -c reader -ti sh
```

# Verify Filestore volume directory creation
## Create an f1-micro Compute Engine instance
```
gcloud compute --project=${PROJECT} instances create check-nfs-provisioner     --zone=${ZONE}     --machine-type=f1-micro     --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append     --image=debian-9-stretch-v20201014     --image-project=debian-cloud     --boot-disk-size=10GB     --boot-disk-type=pd-standard     --boot-disk-device-name=check-nfs-provisioner
```

## Install the nfs-common package on check-nfs-provisioner
```
gcloud compute ssh check-nfs-provisioner --command "sudo apt update -y && sudo apt install nfs-common -y"
```

## Mount the Filestore volume on check-nfs-provisioner
```
gcloud compute ssh check-nfs-provisioner --command "sudo mkdir /mnt/gke-volumes && sudo mount ${FSADDR}:/ftptest_share /mnt/gke-volumes"
```

## gcloud compute ssh check-nfs-provisioner --command  "sudo find /mnt/gke-volumes -type d"
```
gcloud compute ssh check-nfs-provisioner --command  "sudo find /mnt/gke-volumes -type d"
```

# Create alert if fileshare capacity is less than xx%
Go to stackdriver and configure an alert