### Opereto Ingress

1. Allow only HTTPS (port 443)
1. Allow regexp path
1. Add valid certificates


If you are not familiar with ingress yet, please read this first: [What is Ingress?](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress)

## Create ingress service using Ngnix on GKE

Step 1: Create The Tiller Service Account
```console
kubectl create serviceaccount tiller --namespace kube-system
```

Step 2: Bind The Tiller Service Account To The Cluster-Admin Role
```console
kubectl create -f tiller-clusterrolebinding.yaml
```

Step 3: Update The Existing Tiller Deployment
```console
helm init --service-account tiller --upgrade
```

Step 4: Test The New Helm RBAC Rules
```console
helm ls
```

Step 5: Create a Kubernetes secret
```console
kubectl create secret tls clustersecret --key /tmp/tls.key --cert /tmp/tls.crt
```

Step 6: Install nginx-ingress using Helm 
```console
helm install --name nginx-ingress stable/nginx-ingress --set rbac.create=true
```

Step 7: Install Opereto Ingress service
```console
kubectl create -f node_ingress.yaml
```

Step 8: Update the nginx controller and nginx controller config map (nginx-ingress-controller)

**nginx controller config map**
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
```


**nginx controller**    
```console
spec:
      containers:
      - args:
        - /nginx-ingress-controller
        ...
        ...
        - --default-ssl-certificate=default/clustersecret      
```
Step 9: Add autoscaling to ingress controller and increase the replica to more than 1 (optional but recommended)


### Troubleshooting

To run the ngnix in debug mode, change in nginx-ingress-controller config map:

```console
  disable-access-log: "false"
  error-log-level: "debug"
```

Learn more about Nginx/GKE Ingress:
1. https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/
1. https://cloud.google.com/community/tutorials/nginx-ingress-gke
1. https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap
1. https://kubernetes.github.io/ingress-nginx/user-guide/tls/
