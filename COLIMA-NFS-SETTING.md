# Colima NFS Setting

## 1.Set Up NFS Server
1. Create Share folders from host
```
mkdir -p $HOME/k8s-nfs-share
chmod 777 $HOME/k8s-nfs-share

```

2. Setup NFS Exports

https://gist.github.com/doole/93a18db2c3b134d185d74c9e3f7bbd4c

```
echo "$HOME/k8s-nfs-share -alldirs -mapall=$(id -u):$(id -g) localhost" | sudo tee -a /etc/exports

```

or 


```
/Users/toe/k8s-nfs-share -alldirs -mapall=nobody
/Users/toe/k8s-nfs-share -alldirs -mapall=toe -network 192.168.0.0 -mask 255.225.255.0
/Users/toe/k8s-nfs-share -alldirs -mapall=nobody:nogroup -network 192.168.30.0 -mask 255.255.255.0

this work

/Users/toe/k8s-nfs-share -alldirs -mapall=nobody:nogroup -network 192.168.5.0 -mask 255.255.255.0
/Users/toe/k8s-nfs-share -alldirs -mapall=nobody:nogroup -network 192.168.64.0 -mask 255.255.255.0 # this line work
```

3. Restart NFS Server

```
sudo nfsd restart
```

4. test mount
```
showmount -e
```


5. try mount on other system
```
# alpine 


# macos

```

## 2.Get Colima IP

1. Get Colima IP
```
colima list
```

or

```
colima ssh -- ip addr show eth0 | grep inet
```

## 3. ติดตั้ง NFS Subdir External Provisioner ใน K8s

https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

1. เพิ่ม helm chart
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=TOE-MacBookPro-3.local \
    --set nfs.path=$HOME/k8s-nfs-share \
    --set storageClass.name=nfs-client


fix 
helm upgrade --install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.64.2 \
    --set nfs.path=$HOME/k8s-nfs-share \
    --set storageClass.name=nfs-client
```


## 4. Test

1. Create test pvc file
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
  namespace: smlerp
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs-pod
  namespace: smlerp
spec:
  containers:
    - name: test-nfs-pod
      image: busybox
      command: ["/bin/sh", "-c", "while true; do date >> /mnt/test.txt; sleep 5; done"]
      volumeMounts:
        - name: nfs-volume
          mountPath: /mnt
  volumes:
    - name: nfs-volume
      persistentVolumeClaim:
        claimName: test-nfs-pvc
```

2. Create PVC & POD
```
kubectl create namespace smlerp --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f pvc_nfs/test-nfs.yaml

kubectl get pods -n smlerp
```



3. Check files

```
cat $HOME/k8s-nfs-share/smlerp-test-nfs-pvc-pvc-*/test.txt
```



## try mound in vm

```
sudo mount -t nfs TOE-MacBookPro-3.local:/Users/toe/k8s-nfs-share /mnt
```