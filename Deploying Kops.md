KOPS POC Steps

kops binary:

```
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x ./kops
sudo mv ./kops /usr/local/bin/
```


kubectl binary:

```
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

aws cli binary:

```
apt-get update && apt-get install -y python-pip libpython-dev
pip install awscli
```



Step 1  - Setup ENV PARAMS

```
export AWS_ACCESS_KEY_ID={access_key}
export AWS_SECRET_ACCESS_KEY={secret_key}
export AWS_PROFILE={profile}
export KOPS_STATE_STORE=s3://{bucket}
export NAME={deployment_domain}
export AWS_REGION={region}
export KOPS_AWS_REGION={region}
```

Step 1a - create ssh key

```
ssh-keygen -b 4096 -t rsa  ###(use name cg-staging-k8s & no passphrase)
chmod 750 <pub file of key created above>
```

Step 2 - Define the cluster

```
kops create cluster \
    --cloud aws \
    --vpc vpc-e99e618c \
    --network-cidr {vpc_cidr} \
    --node-count 5 \
    --zones {zones} \
    --master-count 3 \
    --master-zones {master_zones} \
    --node-size m4.large \
    --master-size m3.medium \
    --api-loadbalancer-type internal \
    --topology private \
    --networking calico \
    --subnets={main_subnets} \
    --utility-subnets={util_subnet} \
    --image {ami} \
    --dns private \
    --ssh-public-key <pub key from above ssh key creation> \
    --v 10 \
    ${NAME}
```


Step 3 - Modify cluster config

```
<some steps if needed>
```

Step 4 - Deploy cluster for first time

```
kops update cluster ${NAME} --yes
```

Step 5 - Query for cluster to come up

```
kops validate cluster ${NAME} | grep "is ready"
```

Your looking for a posiive hit on grep with a response back of `Your cluster ${NAME} is ready`

Step 6 - Validate kubectl works

```
kubectl cluster-info
```

You should get back URLs for Kubernetes Master and KubeDNS

Step 7 - Deploy nginx ingress controller

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.27.0/deploy/static/mandatory.yaml
```

Next pull in current file from `https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.27.0/deploy/static/provider/aws/service-l4.yaml`

And add under annotations: `service.beta.kubernetes.io/aws-load-balancer-internal: "true"`

We then apply that modified template:

```
kubectl apply -f <modied template file for internal elb>
```

And the config map patch

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.27.0/deploy/static/provider/aws/patch-configmap-l4.yaml
```

Step 8 - Validate nginx-ingress is up

```
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
```

You should see at least 1 pod running - if so you can break out of watch

Step 9 - Define configmap for kube-dns forwarding:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-dns
  namespace: kube-system
data:
  upstreamNameservers: |
    ["{dns_recursor}"]
```

Then apply it:

```
kubectl apply -f <dns config map file>
```

Step 10 - Setup Internal Route53 domain for service URIs

```
All manual for now but we need to automate it
New internal domain based on KOPS NAME ENV
New CNAME wildcard record to the AWS DNS name of the L4 NLB created in step 7 for nginx ingress controller
```
