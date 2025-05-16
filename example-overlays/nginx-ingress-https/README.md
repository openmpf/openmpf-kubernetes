This directory contains an example of how you could configure ingress for 
OpenMPF. This example uses minikube, so you will need to make changes to 
`ingress.yaml` to use this in production. You will need change the 
`wfm.mpf.local` and `amq.mpf.local` hostnames to the actual hostnames used
by your cluster. Other changes may be necessary depending on how your cluster's
ingress controller is configured.

Run example in minikube:
=========================
- Start minikube if it isn't already running: `minikube start`
- Enable ingress addon: `minikube addons enable ingress`
- `cd openmpf-docker/kustomize/overlays/examples/nginx-ingress`
- Generate certificate and keys:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=*.mpf.local/O=*.mpf.local"
```
- Deploy OpenMPF: `kubectl apply -k .`
- Wait for WorkflowManager to start.
- Run `kubectl get ingress` to get the ingress IP address
- Use ingress IP address to add a line to `/etc/hosts` like 
  `192.168.49.2 wfm.mpf.local amq.mpf.local`
- To access WorkflowManager go to `https://wfm.mpf.local/workflow-manager` 
  in a browser.
- To access the ActiveMQ management page go to `https://amq.mpf.local` in a 
  browser.
