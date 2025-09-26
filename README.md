# eks-all-in-one

all tools in one project

## 1. Infrastructure

### 1.1 Create eks cluster

1. Create cluster

- eskctl yaml file examples  
  https://github.com/eksctl-io/eksctl/tree/main/examples

- Set env variables

  ```
  export CLUSTER_NAME=cluster-all-in-one
  export REGION=ap-southeast-2
  export ACCOUNT_ID=<your-account-id>
  ```

- Create cluster by eskctl

  1. command line

  ```
  eksctl create cluster --name $CLUSTER_NAME --region $REGION --spot --instance-types=t3.medium --nodes-max=4 --node-volume-size=20 --dry-run

  # EKS auto mode
  eksctl create cluster --name $CLUSTER_NAME --region $REGION --enable-auto-mode --dry-run
  ```

  2. yaml file

  ```
  eksctl create cluster -f eks-all-in-one.yaml --dry-run
  ```

- Create node group

  ```
  # create
  eksctl create nodegroup --cluster=$CLUSTER_NAME --spot --instance-types=t3.medium --nodes-max=4 --node-volume-size=20 --dry-run
  eksctl create nodegroup --config-file=eks-all-in-one.yaml --dry-run
  eksctl create nodegroup --config-file=eks-all-in-one.yaml

  # scale
  eksctl scale nodegroup --cluster=<clusterName> --nodes=<desiredCount> --name=<nodegroupName> [ --nodes-min=<minSize> ] [ --nodes-max=<maxSize> ] --wait

  # delete
  eksctl delete nodegroup --config-file=eks-all-in-one.yaml
  eksctl delete nodegroup --config-file=eks-all-in-one.yaml --approve

  eksctl delete nodegroup --cluster=<clusterName> --name=<nodegroupName>

  # auto mode - node pool
  kubectl apply -f eks-node-pool.yaml
  ```

- Delete cluster

  ```
  eksctl delete cluster --name $CLUSTER_NAME --region $REGION
  ```

- Suitable Instance Types

  ```
  x86_64 (1vCPU, burstable, 2GB Memory)
  instanceTypes: ["t2.small", "t3.small", "t3a.small"]

  x86_64 (2vCPU, burstable, 4GB Memory)
  instanceTypes: ["t2.medium", "t3.medium", "t3a.medium"]

  arm64 (1vCPU, AWS Graviton2/3, 2GB Memory, except for t4g.medium)
  instanceTypes: ["t4g.medium", "c6g.medium", "c7g.medium", "m6g.medium", "m7g.medium"]

  instanceTypes: ["c3.large","c4.large","c5.large","c5d.large","c5n.large","c5a.large"]
  ```

2. Update the kubeconfig file (~/.kube/config)

   ```
   aws eks update-kubeconfig --name $CLUSTER_NAME
   ```

3. Verify the configuration

   ```
   kubectl get nodes
   ```

### 1.2 Install IAM OIDC provider (Optional)

```
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve
```

### 1.3 Install ALB controller (Service Account)

- Create IAM policy

  ```
  curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json

  aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
  ```

- Create IAM role (Service Account)

  ```
  eksctl create iamserviceaccount \
    --cluster $CLUSTER_NAME \
    --region $REGION \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
  ```

### 1.3a Install ALB controller (Helm)

- Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
```

- Update the repo

```
helm repo update eks
```

- Helm install

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$REGION \
  --set vpcId=<your-vpc-id>
```

- Verify that the deployments are running.

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### 1.4b Install ALB Controller (yaml)

- Download Load Balancer Controller yaml file

```
curl -Lo v2_13_3_full.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.13.3/v2_13_3_full.yaml
```

- Remove the ServiceAccount section in the manifest

```
sed -i.bak -e '730,738d' ./v2_13_3_full.yaml
```

- Replace cluster name in the Deployment spec section of the file with the name of your cluster by replacing my-cluster with the name of your cluster

```
sed -i.bak -e 's|your-cluster-name|$CLUSTER_NAME|' ./v2_13_3_full.yaml
```

- Deploy AWS Load Balancer Controller file

```
kubectl apply -f v2_13_3_full.yaml
```

- Download the IngressClass and IngressClassParams manifest to your cluster. And apply the manifest to your cluster

```
curl -Lo v2_13_3_ingclass.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.13.3/v2_13_3_ingclass.yaml

kubectl apply -f v2_13_3_ingclass.yaml
```

### 1.4c Nginx Ingress Controller

- install

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

kubectl create namespace ingress-nginx

helm install ingress-nginx ingress-nginx --namespace ingress-nginx

# create deployment
kubectl create deployment demo  --image=httpd  --port=80
# expose deployment as a service
kubectl expose deployment demo
#
kubectl create ingress demo --class=nginx \
  --rule /=demo:80
```

### 1.5 EBS CSI Plugin configuration

```
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

```
eksctl create addon --name aws-ebs-csi-driver --cluster $CLUSTER_NAME --service-account-role-arn arn:aws:iam::763015899447:role/AmazonEKS_EBS_CSI_DriverRole --force
```

## 2. Observability

### 2.1 Monitoring (metrics)

- Prometheus Helm Chart

  ```
  kubectl create ns monitoring

  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update

  # Alert Manger - custom_kube_prometheus_stack.yml
  helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f ./custom_kube_prometheus_stack.yml
  ```

  ```
  NOTES:
  kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=monitoring"

  Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

  Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000

  Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
  ```

- Prometheus UI:

  - port-forward

  ```
  kubectl port-forward service/prometheus-operated -n monitoring 9090:9090
  ```

  NOTE: If you are using an EC2 Instance or Cloud VM, you need to pass --address 0.0.0.0 to the above command. Then you can access the UI on instance-ip:port

  - LoadBalancer
  
  ```
  kubectl patch svc monitoring-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
  ```

  - Ingress
  

- Grafana UI: (admin:prom-operator)

  ```
  kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
  ```

- Alertmanager UI:

  ```
  kubectl port-forward service/alertmanager-operated -n monitoring 9093:9093
  ```

- Configure Alertmanager

  - Before configuring Alertmanager, we need credentials to send emails. For this project, we are using Gmail, but any SMTP provider like AWS SES can be used. so please grab the credentials for that.

  - Open your Google account settings and search App password & create a new password & put the password in day-4/alerts-alertmanager-servicemonitor-manifest/email-secret.yml

  - One last thing, please add your email id in the day-4/alerts-alertmanager-servicemonitor-manifest/alertmanagerconfig.yml

  - **HighCpuUsage**: Triggers a warning alert if the average CPU usage across instances exceeds 50% for more than 5 minutes.

  - **PodRestart**: Triggers a critical alert immediately if any pod restarts more than 2 times.

  ```
  kubectl apply -k alerts-alertmanager-servicemonitor-manifest/
  ```

- Access from browser

  ```
  http://127.0.0.1:9090/
  http://127.0.0.1:8080/ (admin:prom-operator)
  http://127.0.0.1:9093/

  http://monitoring-kube-prometheus-prometheus.monitoring:9090/
  ```

### 2.2 Logging (logs)

#### 2.2.1 Elasticsearch

```
kubectl create namespace logging

helm repo add elastic https://helm.elastic.co

helm install elasticsearch \
 --set replicas=1 \
 --set volumeClaimTemplate.storageClassName=gp2 \
 --set persistence.labels.enabled=true elastic/elasticsearch -n logging
```

Retrieve Elasticsearch Username & Password

```
# for username (elastic)
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
# for password
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
```

#### 2.2.2 Kibana

```
helm install kibana --set service.type=LoadBalancer elastic/kibana -n logging

# uninstall
helm uninstall kibana -n logging --no-hooks

# manual uninstall
kubectl delete cm kibana-kibana-helm-scripts -n logging
kubectl delete secret sh.helm.release.v1.kibana.v1 -n logging
kubectl delete sa pre-install-kibana-kibana -n logging
kubectl delete sa post-delete-kibana-kibana -n logging
kubectl delete role pre-install-kibana-kibana -n logging
kubectl delete role post-delete-kibana-kibana -n logging
kubectl delete rolebinding pre-install-kibana-kibana -n logging
kubectl delete rolebinding post-delete-kibana-kibana -n logging
kubectl delete job pre-install-kibana-kibana -n logging
kubectl delete job post-delete-kibana-kibana -n logging
```

#### 2.2.3 Fluent Bit

```
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit -f fluentbit-values.yaml -n logging

# upgrade after editing yaml file
helm upgrade fluent-bit fluent/fluent-bit -f fluentbit-values.yaml -n logging
```

### 2.3 Tracing (Traces)

- Export Elasticsearch CA Certificate

  ```
  kubectl get secret elasticsearch-master-certs -n logging -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca-cert.pem
  ```

- TLS

  ```
  kubectl create ns tracing

  #Jaeger's TLS Certificate
  kubectl create configmap jaeger-tls --from-file=ca-cert.pem -n tracing

  #Elasticsearch TLS
  kubectl create secret generic es-tls-secret --from-file=ca-cert.pem -n tracing
  ```

- Jaeger Helm Repository

  ```
  helm repo add jaegertracing https://jaegertracing.github.io/helm-charts

  helm repo update
  ```

- Install Jaeger with Custom Values

  Note: Please update the password field and other related field in the jaeger-values.yaml file with the password retrieved earlier in day-4 at step 6: (i.e NJyO47UqeYBsoaEU)"

  ```
  helm install jaeger jaegertracing/jaeger -n tracing --values jaeger-values.yaml
  ```

  - Port Forward Jaeger Query Service

  ```
  kubectl port-forward svc/jaeger-query 8080:80 -n tracing
  ```

## Misc

### Reference

- Amazon EKS User Guide

  https://docs.aws.amazon.com/eks/latest/userguide/quickstart.html

### Sample workload

Game 2048

```
kubectl create namespace game-2048

kubectl apply -n game-2048 -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.0/docs/examples/2048/2048_full.yaml
```

### Sample workload

```
kubectl create ns dev

kubectl apply -k kubernetes-manifest/
```

- Write Custom Metrics

  - Please take a look at day-4/application/service-a/index.js file to learn more about custom metrics. below is the brief overview
  - Express Setup: Initializes an Express application and sets up logging with Morgan.
  - Logging with Pino: Defines a custom logging function using Pino for structured logging.
  - Prometheus Metrics with prom-client: Integrates Prometheus for monitoring HTTP requests using the prom-client library:
  - http_requests_total: counter
  - http_request_duration_seconds: histogram
  - http_request_duration_summary_seconds: summary
  - node_gauge_example: gauge for tracking async task duration

- Basic Routes:
  - / : Returns a "Running" status.
  - /healthy: Returns the health status of the server.
  - /serverError: Simulates a 500 Internal Server Error.
  - /notFound: Simulates a 404 Not Found error.
  - /logs: Generates logs using the custom logging function.
  - /crash: Simulates a server crash by exiting the process.
  - /example: Tracks async task duration with a gauge.
  - /metrics: Exposes Prometheus metrics endpoint.
  - /call-service-b: To call service b & receive data from service b

```
kubectl run busybox-crash --image=busybox -- /bin/sh -c "exit 1"
```

### Folder Structure

- ./EKS  
  Three tier architecture - Stan's Robot Shop

- ./kubernetes-manifest  
  Sample workload (metrics, logs, traces)

- ./fluentbit-values.yaml  
  Helm chart custom file

- ./jaeger-values.yaml  
  Helm chart custom file

Amazon EC2 instance type naming conventions  
https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-type-names.html

DO NOT assign instance type support less pods with the ones support more  
https://github.com/aws/amazon-vpc-cni-k8s/blob/master/misc/eni-max-pods.txt

```
t2.2xlarge 44
t2.large 35
t2.medium 17
t2.micro 4
t2.nano 4
t2.small 11
t2.xlarge 44
t3.2xlarge 58
t3.large 35
t3.medium 17
t3.micro 4
t3.nano 4
t3.small 11
t3.xlarge 58
t3a.2xlarge 58
t3a.large 35
t3a.medium 17
t3a.micro 4
t3a.nano 4
t3a.small 8
t3a.xlarge 58
t4g.2xlarge 58
t4g.large 35
t4g.medium 17
t4g.micro 4
t4g.nano 4
t4g.small 11
t4g.xlarge 58
m7g.12xlarge 234
m7g.16xlarge 737
m7g.2xlarge 58
m7g.4xlarge 234
m7g.8xlarge 234
m7g.large 29
m7g.medium 8
m7g.metal 737
m7g.xlarge 58
```
