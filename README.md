# eks template

simple repo to create and manage an eks cluster with eksctl, along with aws lb ingress controller.

## prerequisites

- aws cli
- kubectl
- helm
- eksctl

## getting started

- setup env vals

  ```sh
  export EKS_CLUSTER_NAME=eks-demo
  export AWS_REGION=ap-south-1
  export AWS_ACCOUNT=123456789012
  ```

- create eks cluster
  eg.

  ```
  eksctl create cluster -f eksctl_cluster.yaml
  ```

- install cni plugin (eg. flannel)

  ```sh
  kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
  ```

- create deployment and service, test locally with port forward on `localhost:8000`

  ```sh
  kubectl create deployment web --image nginx --replicas 4 --port 80

  kubectl expose deployment web --port 80 --target-port 80

  kubectl port-forward services/web 8000:80
  ```

### setting up aws lb ingress

- download & create the iam pol for lb controller

  ```sh
  curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json
  aws iam create-policy \
      --policy-name AWSLoadBalancerControllerIAMPolicy \
      --policy-document file://iam_policy.json
  ```

- create oidc provider & service account

  ```sh
  eksctl utils associate-iam-oidc-provider --region=$AWS_REGION --cluster=$EKS_CLUSTER_NAME --approve
  ```

  ```sh
  eksctl create iamserviceaccount \
    --cluster=$EKS_CLUSTER_NAME \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::$AWS_ACCOUNT:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve
  ```

- install the `aws-load-balancer-controller` with `helm`

  ```sh
  helm repo add eks https://aws.github.io/eks-charts

  helm repo update

  helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=$CLUSTER_NAME   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller
  ```

- setup ingress

  save the following file as `ingress.yaml`

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    namespace: default
    name: demo-ingress
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
  spec:
    ingressClassName: alb
    rules:
      - http:
          paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: web
                  port:
                    number: 80
  ```

  deploy ingress

  ```sh
  kubectl apply -f ingress.yaml
  ```

  get the ALB's DNS with `kubectl get ingress`

  NOTE: give it some time to deploy the ALB.

- update cloudflare DNS record CNAME to point to alb's DNS. make sure to enable proxy status

and voila! your app with HTTPS on EKS is deployed! 🚀

---

### note about tls/https

the last step above will provide tls connection from your machine to cloudflare, but not between cloudflare and ALB.

to get tls directly to alb:

- provision an acm cert for your domain
- attach the cert to the alb's https listener

## clean up

clean up all the resources in reverse order

```
kubectl delete ingress demo-ingress

helm del aws-load-balancer-controller -n kube-system

kubectl delete service web
kubectl delete deployments.apps web

eksctl delete cluster -f eksctl_cluster.yaml
```

## refs

- https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html
- https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html
- https://github.com/flannel-io/flannel
