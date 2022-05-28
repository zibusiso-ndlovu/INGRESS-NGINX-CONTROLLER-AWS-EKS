## Clone the Ingress Controller repo and change into the deployments folder:

* git clone https://github.com/nginxinc/kubernetes-ingress.git --branch v2.2.2
* cd kubernetes-ingress/deployments

## Configure RBAC

### Create a namespace and a service account for the Ingress Controller:
* kubectl apply -f common/ns-and-sa.yaml
### Create a cluster role and cluster role binding for the service account:
* kubectl apply -f rbac/rbac.yaml
### (App Protect only) Create the App Protect role and role binding:
* kubectl apply -f rbac/ap-rbac.yaml
### (App Protect DoS only) Create the App Protect DoS role and role binding:
* kubectl apply -f rbac/apdos-rbac.yaml

## Create Common Resources
In this section, we create resources common for most of the Ingress Controller installations:

### Create a secret with a TLS certificate and a key for the default server in NGINX:
* kubectl apply -f common/default-server-secret.yaml

Note: The default server returns the Not Found page with the 404 status code for all requests for domains for which there are no Ingress rules defined. For testing purposes we include a self-signed certificate and key that we generated. However, we recommend that you use your own certificate and key.

### Create an IngressClass resource:
*  kubectl apply -f common/ingress-class.yaml

If you would like to set the Ingress Controller as the default one, uncomment the annotation ingressclass.kubernetes.io/is-default-class. With this annotation set to true all the new Ingresses without an ingressClassName field specified will be assigned this IngressClass.

Note: The Ingress Controller will fail to start without an IngressClass resource.

### Create Custom Resources
Note: By default, it is required to create custom resource definitions for VirtualServer, VirtualServerRoute, TransportServer and Policy. Otherwise, the Ingress Controller pods will not become Ready. If you’d like to disable that requirement, configure -enable-custom-resources command-line argument to false and skip this section.

#### Create custom resource definitions for VirtualServer and VirtualServerRoute, TransportServer and Policy resources:
* kubectl apply -f common/crds/k8s.nginx.org_virtualservers.yaml
* kubectl apply -f common/crds/k8s.nginx.org_virtualserverroutes.yaml
* kubectl apply -f common/crds/k8s.nginx.org_transportservers.yaml
* kubectl apply -f common/crds/k8s.nginx.org_policies.yaml

#### If you would like to use the TCP and UDP load balancing features of the Ingress Controller, create the following additional resources:
#### Create a custom resource definition for GlobalConfiguration resource
* kubectl apply -f common/crds/k8s.nginx.org_globalconfigurations.yaml

### Resources for NGINX App Protect

#### Create a custom resource definition for APPolicy, APLogConf and APUserSig:

*  kubectl apply -f common/crds/appprotect.f5.com_aplogconfs.yaml
* kubectl apply -f common/crds/appprotect.f5.com_appolicies.yaml
* kubectl apply -f common/crds/appprotect.f5.com_apusersigs.yaml

### Resources for NGINX App Protect DoS
If you would like to use the App Protect DoS module, create the following additional resources:

#### Create a custom resource definition for APDosPolicy, APDosLogConf and DosProtectedResource:

*  kubectl apply -f common/crds/appprotectdos.f5.com_apdoslogconfs.yaml
* kubectl apply -f common/crds/appprotectdos.f5.com_apdospolicy.yaml
* kubectl apply -f common/crds/appprotectdos.f5.com_dosprotectedresources.yaml

# Deploy the Ingress Controller
We include two options for deploying the Ingress Controller:

* Deployment. Use a Deployment if you plan to dynamically change the number of Ingress Controller replicas.
* DaemonSet. Use a DaemonSet for deploying the Ingress Controller on every node or a subset of nodes.

Before creating a Deployment or Daemonset resource, make sure to update the command-line arguments of the Ingress Controller container in the corresponding manifest file according to your requirements.

### Deploy Arbitrator for NGINX App Protect DoS
If you would like to use the App Protect DoS module, need to add arbitrator deployment.

build your own image and push it to your private Docker registry by following the instructions from here.

#### run the Arbitrator by using a Deployment and Service
* kubectl apply -f deployment/appprotect-dos-arb.yaml
* kubectl apply -f service/appprotect-dos-arb-svc.yaml

## Run the Ingress Controller
Use a Deployment. When you run the Ingress Controller by using a Deployment, by default, Kubernetes will create one Ingress Controller pod.

* kubectl apply -f deployment/nginx-ingress.yaml

## Check that the Ingress Controller is Running

* kubectl get pods --namespace=nginx-ingress

## Create a Service for the Ingress Controller Pods

### Use a NodePort service.
* kubectl create -f service/nodeport.yaml

### Use a LoadBalancer service:
* kubectl apply -f service/loadbalancer-aws-elb.yaml

#### Configuring a LoadBalancer Service to Use NLB:
In service/loadbalancer-aws-elb.yaml, add the following annotation to the existing or new LoadBalancer service:


Kubernetes will allocate a Classic Load Balancer (ELB) in TCP mode with the PROXY protocol enabled to pass the client’s information (the IP address and the port). NGINX must be configured to use the PROXY protocol:

service.beta.kubernetes.io/aws-load-balancer-type: nlb

* kubectl apply -f service/loadbalancer-aws-elb.yaml


* Add the following keys to the config map file nginx-config.yaml from the Step 2:

  proxy-protocol: "True"
  real-ip-header: "proxy_protocol"
  set-real-ip-from: "0.0.0.0/0"

* kubectl apply -f common/nginx-config.yaml

### Use the public IP of the load balancer to access the Ingress Controller. To get the public IP:

In case of AWS ELB, the public IP is not reported by kubectl, because the ELB IP addresses are not static. In general, you should rely on the ELB DNS name instead of the ELB IP addresses. However, for testing purposes, you can get the DNS name of the ELB using kubectl describe and then run nslookup to find the associated IP address:

* kubectl describe svc nginx-ingress --namespace=nginx-ingress

You can resolve the DNS name into an IP address using nslookup:

* nslookup <dns-name>


SOURCE: https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/



# INGRESS-NGINX-CONTROLLER-AWS-EKS
