### Opereto Ingress Example (GKE/Nginx)

1. Allow only HTTPS (port 443)
1. Allow regexp path
1. Add valid certificates

If you are not familier with ingress yet, please read this first: [What is Ingress?](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress)
This example assumes that you installed opereto cluster using the default namespace (e.g. opereto). Otherwise, please modify the namespace in all kubectl commands and yaml files given in this example.

## Create ingree service using Ngnix on GKE

Step 1: Create The Tiller Service Account
```console
kubectl -n opereto create serviceaccount tiller
```

Step 2: Bind The Tiller Service Account To The Cluster-Admin Role
```console
kubectl -n opereto create -f tiller-clusterrolebinding.yaml
```

Step 3: Update The Existing Tiller Deployment
```console
helm init --tiller-namespace opereto --service-account tiller --upgrade
```

Step 4: Test The New Helm RBAC Rules
```console
helm ls
```

Step 5: Create a Kubernetes secret
```console
kubectl -n opereto create secret tls clustersecret --key /tmp/tls.key --cert /tmp/tls.crt
```

Step 6: Install nginx-ingress using Helm 
```console
helm install --tiller-namespace opereto --namespace opereto --name nginx-ingress stable/nginx-ingress --set rbac.create=true
```

Step 7: Install Opereto Ingress service (opereto-lb)

1. Replace ssl.example.com in the node_ingress.yaml with the relevant url
1. Create the Ingress service
```console
kubectl -n opereto create -f node_ingress.yaml
```

Step 8: Update the nginx controller and config map (nginx-ingress-controller)

1. Create the custom headers config map
```console
kubectl -n opereto create -f custom-headers.yaml
```

**nginx controller configmap**
```console
...
data:
  disable-access-log: "true"
  error-log-level: error
  keep-alive: 75s
  proxy-body-size: 200m
  ssl_ciphers: EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH;
  ssl_session_cache: builtin:1000  shared:SSL:10m;
  ssl_session_timeout: 10m;
  proxy-connect-timeout: "300"
  proxy-read-timeout: "300"
  proxy-send-timeout: "300"
  proxy-set-headers: "opereto/custom-headers"
```


**nginx-ingress-controller (workload)**    
```console
spec:
      containers:
      - args:
        - /nginx-ingress-controller
        ...
        ...
        - --default-ssl-certificate=default/clustersecret      
```

Step 9: Add autoscaling to ingress controller and backend and increase the replica to more than 1 (optional but recommended)


**nginx-ingress-controller service**:
Add the external IP attached to the load balancer

```console
spec:
  ...
      loadBalancerIP: 1.1.1.1
  
```

### Troubleshooting

To run the ngnix in debug mode, change in nginx-ingress-controller config map:

```console
  disable-access-log: "false"
  error-log-level: "debug"
```

To display the ingress controller config:
```console
kubectl -n opereto exec nginx-ingress-controller-7dbfc54c45-mt2kx -- cat nginx.conf
```

Learn more about Nginx/GKE Ingress:
1. https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/
1. https://cloud.google.com/community/tutorials/nginx-ingress-gke
1. https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap
1. https://kubernetes.github.io/ingress-nginx/user-guide/tls/
