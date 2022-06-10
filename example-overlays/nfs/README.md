This example shows how the OpenMPF shared data volume (`openmpf-shared-data`) 
could be configured to use NFS. This example assumes you have read the 
"Configure shared data volume" section of the root README. This example uses 
minikube, so it can be useful for developers that need to test a multi-host 
deployment locally. 

Parts of this example might be useful to a cluster administrator that needs to 
set up an NFS volume provisioner. In general, using a `ReadWriteMany` volume
type from your cloud provider will be a more robust solution. If you are 
deploying on bare metal or cannot use your cloud provider's `ReadWriteMany` 
volume, the
[`nfs-subdir-external-provisioner`](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
can be used. `nfs-subdir-external-provisioner` only provisions the volumes, so
it requires access to an existing NFS server.


Deploy locally with minikube:
=================================
- Start an NFS server. For local testing, the
  `k8s.gcr.io/volume-nfs:0.8` Docker image can be used. A more robust
  solution should be used in production.
```bash
docker run --rm -d --privileged --name nfs-server  -p 2049:2049 -p 20048:20048 -p 111:111 k8s.gcr.io/volume-nfs:0.8
```
- If you have an existing minikube cluster with a single node, you will need to 
  delete that first with `minikube delete --all`
- Start a minikube cluster with at least two nodes: `minikube start --nodes 2`.
- Assuming you are in the same directory as this README, start 
  `nfs-subdir-external-provisioner` with: 
  `kubectl apply -k nfs-subdir-external-provisioner`
- Start OpenMPF: `kubectl apply -k .`
- Wait for Workflow Manager to start.
- Expose Workflow Manager: 
  `kubectl port-forward service/workflow-manager 8080:8080`
- Go to <http://localhost:8080/workflow-manager> in a browser.
- When you are done, tear down OpenMPF with `kubectl delete -k .`
- Wait for the OpenMPF pods to stop. Check that they have stopped by running:
  `kubectl get pods`
- Remove `nfs-subdir-external-provisioner` by running: 
  `kubectl delete -k nfs-subdir-external-provisioner`
- Stop the NFS server by running: `docker stop nfs-server`
