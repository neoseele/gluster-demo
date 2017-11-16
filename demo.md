## Deploy Gluster

### Create the GKE cluster

_must have minimal 3 nodes to run a gluster cluster_

```sh
gcloud container clusters create kube-test \
--zone=us-central1-b \
--machine-type=n1-standard-2 \
--num-nodes=3 \
--image-type=COS \
--node-labels=storagenode=glusterfs \
--tags=ssh \
--local-ssd-count=1 \
--scopes cloud-platform,storage-rw,logging-write,monitoring-write,service-control,service-management
```

### Modprobe / Cleanup SSD

```sh
i=1
for node in `kubectl get nodes -o jsonpath='{.items[*].metadata.name}'`;
do
  echo "* ${node}";
  gcloud compute ssh $node --zone us-central1-b -- 'sudo sh -c "modprobe dm_thin_pool; modprobe dm_snapshot; modprobe dm_mirror"'
  gcloud compute ssh $node --zone us-central1-b -- 'sudo sh -c "umount /dev/sdb && dd if=/dev/zero of=/dev/sdb bs=512 count=100"'
  # gcloud compute ssh $node --zone us-central1-b -- 'sudo sysctl -w net.bridge.bridge-nf-call-iptables=1'
  ((i+=1))
done
```

### Pre Deploy
```sh
ADMIN_KEY='12qwaszx34erdfcv'

kubectl create secret generic heketi-admin-secret \
  --from-literal=key=$ADMIN_KEY \
  --type=kubernetes.io/glusterfs
```

```sh
cat << EOF | kubectl apply -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: nm-toolbox
spec:
  containers:
    - name: nm-toolbox
      image: gcr.io/nmiu-play/nm-toolbox:latest
      command:
      - '/bin/sh'
      - '-c'
      - while true; do sleep 5; done;
      env:
        - name: HEKETI_ADMIN_SECRET
          valueFrom:
            secretKeyRef:
              name: heketi-admin-secret
              key: key
EOF
```

### Deploy

```sh
kubectl exec nm-toolbox deploy.sh
```

### Create service (ext)

```sh
cat << EOF | kubectl apply -f -
---
kind: Service
apiVersion: v1
metadata:
  name: heketi-ext
  labels:
    glusterfs: heketi-ext-service
    heketi: ext-service
  annotations:
    description: Exposes Heketi Service Externally
spec:
  type: LoadBalancer
  selector:
    glusterfs: heketi-pod
  ports:
  - name: heketi
    port: 18080
    targetPort: 8080
EOF
```

### Create storage class

```sh
HEKETI="$(kubectl describe service heketi-ext | grep Ingress: | awk '{print $3}'):18080"
echo $HEKETI
```
_use the heketi-ext-service for resturl_

```sh
cat << EOF | kubectl apply -f -
---
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: gluster-heketi
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://${HEKETI}"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-admin-secret"
EOF
```
_mark gluster-heketi as default storage class_

```sh
kubectl patch storageclass gluster-heketi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### Heketi Cli

_In nm-toolbox pod_

```sh
heketi-cli-admin volume list

## reload topology
build-topology.rb > topology.json
heketi-cli-admin topology load --json=topology.json
```

## Clean Up

* Delete load balancer type services
```sh
kubectl get service --all-namespaces
```

* Delete the cluster
```sh
gcloud container clusters delete kube-test --zone us-central1-b
```