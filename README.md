
Deploying a Simple Microservices App using AWS Load Balancer Controller (Domain Configuration)
=========================================================================

aws-lb-controller-domain-configuration/
│
├── app1/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── app2/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── app3/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
│
└── k8s/
    ├── app1-deployment.yaml
    ├── app2-deployment.yaml
    ├── app3-deployment.yaml
    └── ingress.yaml

🔹 STEP 1 — Launch Ubuntu VM 
				Install AWS CLI, kubectl, eksctl
				Attach the IAM Role

sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

🔹 STEP 2 — Setup EKS Cluster
				eksctl create cluster --name neeraj-cluster --region ap-southeast-2 --node-type t2.medium  --zones ap-southeast-2a,ap-southeast-2b

🔹 STEP 3 — Install Docker
				Build Images, Push Images

🔹 STEP 4 — OIDC
eksctl utils associate-iam-oidc-provider \
--region ap-southeast-2 \
--cluster neeraj-cluster \
--approve

🔹 STEP 5 — IAM Policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy-v3 \
  --policy-document file://iam_policy.json

🔹 STEP 6 — IRSA
eksctl create iamserviceaccount \
  --cluster neeraj-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::560185625463:policy/AWSLoadBalancerControllerIAMPolicy-v3 \
  --approve \
  --region ap-southeast-2

🔹 STEP 7 — Install Controller using helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=neeraj-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-southeast-2

kubectl get pods -n kube-system

🔹 STEP 8 — Write the Deployment and Service Manifest Files

🔹 STEP 9 — Write the Ingress Manifest File

🔹 STEP 10 — Apply all Manifests & Access the App for Path Based Routing 
http://<ALB-DNS>/app1
http://<ALB-DNS>/app2
http://<ALB-DNS>/app3

http://k8s-microservicesgrou-322f257afd-1308123168.ap-southeast-2.elb.amazonaws.com/app3

🔹 STEP 11 — Cleanup Commands
Delete Ingress
kubectl delete ingress microservices-ingress
Wait ~1–2 minutes (ALB deletion is async)

Delete Services
kubectl delete svc app1-service app2-service app3-service

Delete Deployments (Pods go away)
kubectl delete deploy app1 app2 app3

Delete everything (shortcut if needed)
kubectl delete all --all

🔹 STEP 12 — Create SSL Certificate (Pre-requisite - Domain)


🔹 STEP 13 — HTTPS Connection with Domain Configuration
					
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/success-codes: '200'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    # HTTPS Configuration
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    # Certificate ARN - Replace with your ACM certificate ARN
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:XXXXXXXXXXXX:certificate/xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/backend-protocol: HTTP
    alb.ingress.kubernetes.io/group.name: microservices-group
spec:
  ingressClassName: alb
  rules:
  - host: learndevops01.click
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
      - path: /app3
        pathType: Prefix
        backend:
          service:
            name: app3-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  # Optional: Add www domain redirect
  - host: www.learndevops01.click
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80

🔹 STEP 14 — Sub-domain Based Routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/success-codes: '200'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    # HTTPS Configuration
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    # Certificate ARN - Replace with your ACM certificate ARN
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-southeast-2:XXXXXXXXXXXX:certificate/xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    alb.ingress.kubernetes.io/backend-protocol: HTTP
    alb.ingress.kubernetes.io/group.name: microservices-group
spec:
  ingressClassName: alb
  rules:
  # Subdomain for App 1 - Customer Service
  - host: app1.learndevops01.click
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  
  # Subdomain for App 2 - Analytics Dashboard
  - host: app2.learndevops01.click
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  
  # Subdomain for App 3 - API Gateway
  - host: app3.learndevops01.click
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app3-service
            port:
              number: 80
  
  # Optional: Main domain redirect to App1
  - host: learndevops01.click
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  
  # Optional: www domain redirect to main domain
  - host: www.learndevops01.click
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80




