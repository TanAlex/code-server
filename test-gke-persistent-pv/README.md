# README for test GKE persistent disk

```
gcloud compute disks create gke-pv  --zone=northamerica-northeast1-a --size=50GB

kk apply -f storage-class-zonal.yaml

kk apply -f disk-pv-zonal.yaml

kk get pv
kk get pvc

kk apply -f app-pod.yaml

kk exec -it nginx-app-pod -- bash

root@nginx-app-pod:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          95G  4.6G   90G   5% /
tmpfs            64M     0   64M   0% /dev
tmpfs           1.8G     0  1.8G   0% /sys/fs/cgroup
shm              64M     0   64M   0% /dev/shm
/dev/sda1        95G  4.6G   90G   5% /etc/hosts
/dev/sdd         49G   24K   49G   1% /usr/share/nginx/html  <==  This is mounted to the PV
```

Test regional persistent disk

```
# create a new regional disk from the zonal one we created above
https://cloud.google.com/compute/docs/disks/create-disk-from-source#create-regional-clone
PROJECT_ID=<your project id>
REGION=northamerica-northeast1
ZONE1=${REGION}-a
ZONE2=${REGION}-b
SOURCE_DISK_NAME=gke-pv
TARGET_DISK_NAME=my-regional-disk-1
gcloud compute disks create $TARGET_DISK_NAME \
  --description="zonal to regional cloned disk" \
  --region=$REGION \
  --source-disk=$SOURCE_DISK_NAME \
  --source-disk-zone=$ZONE1 \
  --replica-zones=$ZONE1,$ZONE2 \
  --project=$PROJECT_ID

# Resize it to be 210G
https://cloud.google.com/compute/docs/disks/regional-persistent-disk#resize_repd
gcloud compute disks resize $TARGET_DISK_NAME \
    --region=$REGION  \
    --size=200

kk apply -f storage-class-reginal.yaml

kk apply -f disk-pv-regional.yaml

kk get pv
kk get pvc

kk apply -f app-pod.yaml

kk exec -it nginx-app-pod -- bash
```

Here is the process to resize the 50G disk to 200G from the node

```
kk get node
# login to the node in the zone with the regional-disk
gcloud compute ssh --project $PROJECT_ID --zone northamerica-northeast1-b \
gke-ttan-lab-zonal-gke-default-pool-341fee45-q521

# resize disk reference
# https://cloud.google.com/compute/docs/disks/resize-persistent-disk
df -Th
lsblk
sdb         8:16   0  200G  0 disk /home/kubernetes/containerized_mounter/rootfs/var/lib/kubelet/pods/646941e0-e768-439f-98e6-c108d4e5

# Resize the disk
resize2fs /dev/sdb
# Result like this
resize2fs 1.46.2 (28-Feb-2021)
Filesystem at /dev/sdb is mounted on /var/lib/kubelet/plugins/kubernetes.io/csi/pv/app-storage/globalmount; on-line resizing required
old_desc_blocks = 7, new_desc_blocks = 25
The filesystem on /dev/sdb is now 52428800 (4k) blocks long.

# now if you go back to the pod, you will find the mount is now changed from 50G to 200G
df -h 
/dev/sdb        197G   28K  197G   1% /usr/share/nginx/html
```