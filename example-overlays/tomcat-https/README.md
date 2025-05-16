In this example, we configure Workflow Manager's Tomcat server to use HTTPS. 
Generally, configuring the ingress controller to use HTTPS is a better option 
and there is an example of this in the `nginx-ingress-https` directory. 
In the `nginx-ingress-https` example, the connection between the client and 
ingress controller uses HTTPS and the connection between the ingress controller
and Tomcat uses HTTP. This example is different because it configures Tomcat
itself to use HTTPS.


Run example in minikube:
=========================
- Start minikube if it isn't already running: `minikube start`
- Generate certificate and keys:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=*.mpf.local/O=*.mpf.local"
```
- Export certificate and key in pkcs12 format. The command will prompt for a 
  password. This password must be added to the `wfm-https` secret's 
  `KEYSTORE_PASSWORD` key in `kustomization.yaml`. In this example, `mpf123`
  was used as the password.
```bash
openssl pkcs12 -export -in tls.crt -inkey tls.key -out mycert.p12
```
- Start OpenMPF: 
```bash
kubectl apply -k .
```
- Wait for Workflow Manager to start.
- Expose Workflow Manager's https port: 
```bash
kubectl port-forward service/workflow-manager 8443
```
- Go to `https://localhost:8443/workflow-manager` in a browser
