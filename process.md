This procedure has the goal to deploy apps on AKS with HTTPS and certificated enabled.

Generate certificates could be expensive and difficult but here merging two official Microsoft articles I show that is possible to do it for free as well.

Hope you enjoy!

Link to video where I get hands on -- > https://youtu.be/iTqk42-9ZB4

--- Using powershell pre tasks

https://kubernetes.io/docs/tasks/tools/install-kubectl/

https://www.ntweekly.com/2020/11/16/install-helm-on-a-windows-10-machine/

 

# Pre install chocalatey to aux in the helm and kubs commands installation

 

$ Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

 

$ choco install kubernetes-helm

$ choco install kubernetes-cli

 

--- Azure CLI installation

https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-powershell

 

---

$ az login

 

--- AKS Kubernetes installation

https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

 

 

# $ az group create --name myResourceGroup --location eastus

 

I could've used $ az account list-locations and choosing europe but here we go default

 

$ az group create --name drummer --location eastus

 

--- AKS Cluster creating

 

# $ az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --enable-addons monitoring --generate-ssh-keys

 

$ az aks create --resource-group drummer --name drummer-eks --node-count 2 --enable-addons monitoring --generate-ssh-keys

 

 

--- Merge the credentials into .\.kube\config

 

$ az aks get-credentials --resource-group drummer --name drummer-eks

 

--- Show the kubes

$ kubectl get nodes

$ kubectl get pods --all-namespaces

$ kubectl get pods -A

 

--- checking resources

$ kubectl top nodes

$ kubectl top pods -A

 

PS. Master not managed by us only the workers master by Azure

 

--- Creating HTTPS ingress controller on AKS (before deploying an application let's secure it)

https://docs.microsoft.com/en-us/azure/aks/ingress-tls

 

creating using NGINX

 

$ kubectl create namespace ingress-basic

$ helm repo add ingress-nginx2 https://kubernetes.github.io/ingress-nginx

 

 

--- For Linux ---

helm install nginx-ingress ingress-nginx/ingress-nginx \

   --namespace ingress-basic \

   --set controller.replicaCount=2 \

   --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \

   --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \

   --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux

---end---

 

--- On PowerShell

 

$ helm install nginx-ingress2 ingress-nginx/ingress-nginx2 --namespace ingress-basic --set controller.replicaCount=2 --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux

 

---viewing---

 

$ kubectl get pods -n ingress-basic

 

$ kubectl get svc -n ingress-basic

 

$ kubectl get svc -n ingress-basic -o yaml

 

---DNS will not be covered

 

I am gonna to use the LB URL that was created instead

 

--- on Linux

export IP="x.x.x.x"

export DNSNAME="ilovedrums"

 

--- powershell

Set-Variable IP XXX.XXX.XXX.XXX

Set-Variable DNSNAME ilovedrums

 

# Get the resource-id of the public ip

--- powershell

$ Set-Variable PUBLICIPID $(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)

$ Write-Output $PUBLICIPID

 

 

 

--- linux

$ export PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)

$ echo $PUBLICIPID

 

--------

--------

 

# Update public ip address with DNS name

$ az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME

 

= output FQDN

= "fqdn": "ilovedrums.eastus.cloudapp.azure.com",

 

# Display the FQDN

az network public-ip show --ids $PUBLICIPID --query "[dnsSettings.fqdn]" --output tsv

 

not secure yet
 

DNS to IP resolving...

----

----

 

--- Installing cert-manager

 

$ kubectl label namespace ingress-basic cert-manager.io/disable-validation=true

$ helm repo add jetstack https://charts.jetstack.io

$ helm repo update

 

$ helm install cert-manager jetstack/cert-manager --namespace ingress-basic --version v0.16.1 --set installCRDs=true --set nodeSelector."kubernetes\.io/os"=linux --set webhook.nodeSelector."kubernetes\.io/os"=linux --set cainjector.nodeSelector."kubernetes\.io/os"=linux

 

--- Create CA cluster

 

YAML file

###############################################

apiVersion: cert-manager.io/v1alpha2

kind: ClusterIssuer

metadata:

 name: letsencrypt

spec:

 acme:

   server: https://acme-v02.api.letsencrypt.org/directory

   email: MY_EMAIL_ADDRESS

   privateKeySecretRef:

     name: letsencrypt

   solvers:

   - http01:

       ingress:

         class: nginx

         podTemplate:

           spec:

             nodeSelector:

               "kubernetes.io/os": linux

###################################################

 

$ kubectl apply -f cluster-issuer.yaml

 

-- Running a demo application deploy

 

creating the yamls files

aks-helloworld-one.yaml

#########################################

apiVersion: apps/v1

kind: Deployment

metadata:

 name: aks-helloworld-one

spec:

 replicas: 1

 selector:

   matchLabels:

     app: aks-helloworld-one

 template:

   metadata:

     labels:

       app: aks-helloworld-one

   spec:

     containers:

     - name: aks-helloworld-one

       image: mcr.microsoft.com/azuredocs/aks-helloworld:v1

       ports:

       - containerPort: 80

       env:

       - name: TITLE

         value: "Welcome to Azure Kubernetes Service (AKS)"

---

apiVersion: v1

kind: Service

metadata:

 name: aks-helloworld-one

spec:

 type: ClusterIP

 ports:

 - port: 80

 selector:

   app: aks-helloworld-one

#########################################

aks-helloworld-two.yaml

#########################################

apiVersion: apps/v1

kind: Deployment

metadata:

 name: aks-helloworld-two

spec:

 replicas: 1

 selector:

   matchLabels:

     app: aks-helloworld-two

 template:

   metadata:

     labels:

       app: aks-helloworld-two

   spec:

     containers:

     - name: aks-helloworld-two

       image: mcr.microsoft.com/azuredocs/aks-helloworld:v1

       ports:

       - containerPort: 80

       env:

       - name: TITLE

         value: "AKS Ingress Demo"

---

apiVersion: v1

kind: Service

metadata:

 name: aks-helloworld-two

spec:

 type: ClusterIP

 ports:

 - port: 80

 selector:

   app: aks-helloworld-two

#########################################

 

--- applying

$ kubectl apply -f aks-helloworld-one.yaml --namespace ingress-basic

$ kubectl apply -f aks-helloworld-two.yaml --namespace ingress-basic

 

---checking the deploys

$ kubectl get pods -n ingress-basic

 

--- creating the ingress route

 

yaml file hello-world-ingress.yaml

#####################################

apiVersion: networking.k8s.io/v1beta1

kind: Ingress

metadata:

 name: hello-world-ingress

 annotations:

   kubernetes.io/ingress.class: nginx

   nginx.ingress.kubernetes.io/rewrite-target: /$1

   nginx.ingress.kubernetes.io/use-regex: "true"

   cert-manager.io/cluster-issuer: letsencrypt

spec:

 tls:

 - hosts:

   - hello-world-ingress.MY_CUSTOM_DOMAIN

   secretName: tls-secret

 rules:

 - host: hello-world-ingress.MY_CUSTOM_DOMAIN

   http:

     paths:

     - backend:

         serviceName: aks-helloworld-one

         servicePort: 80

       path: /hello-world-one(/|$)(.*)

     - backend:

         serviceName: aks-helloworld-two

         servicePort: 80

       path: /hello-world-two(/|$)(.*)

     - backend:

         serviceName: aks-helloworld-one

         servicePort: 80

       path: /(.*)

---

apiVersion: networking.k8s.io/v1beta1

kind: Ingress

metadata:

 name: hello-world-ingress-static

 annotations:

   kubernetes.io/ingress.class: nginx

   nginx.ingress.kubernetes.io/rewrite-target: /static/$2

   nginx.ingress.kubernetes.io/use-regex: "true"

   cert-manager.io/cluster-issuer: letsencrypt

spec:

 tls:

 - hosts:

   - hello-world-ingress.MY_CUSTOM_DOMAIN

   secretName: tls-secret

 rules:

 - host: hello-world-ingress.MY_CUSTOM_DOMAIN

   http:

     paths:

     - backend:

         serviceName: aks-helloworld-one

         servicePort: 80

       path: /static(/|$)(.*)

 

##############################

 

PS. replace the host(s) values to your FQND / domain name

 

--- applying

$ kubectl apply -f hello-world-ingress.yaml --namespace ingress-basic

 

---checking the certificate :))

 

$ kubectl get certificate -n ingress-basic

 

$ kubectl describe certificate -n ingress-basic

 

 

 

 

 

 

 

Extras...

If do you want to replace or edit hello-world-one.yaml into the

$ kubectl replace -f FILENAME

$ kubectl edit (RESOURCE/NAME | -f FILENAME)
