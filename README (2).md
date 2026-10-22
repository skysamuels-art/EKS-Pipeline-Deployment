# Flask on AWS EKS: Terraform, Jenkins and GitOps

A Flask "Hello, World!" app deployed to Amazon EKS. Terraform builds the AWS infrastructure, Docker packages the app, and Helm defines the Kubernetes resources. Two deployment paths are included: **Jenkins** (`main` branch) and **GitHub Actions + Argo CD** (`gitops` branch).

![Architecture diagram: a developer pushes to GitHub; the main branch deploys through Jenkins and the gitops branch deploys through GitHub Actions and Argo CD, both into one EKS cluster behind an ALB in a VPC](images/architecture.png)

## At a glance

| | |
| --- | --- |
| Region | `us-west-1` |
| EKS cluster | `tc2-eks` |
| ECR repository | `hello-world` |
| GitHub repository | `skysamuels-art/tech-2a-app` |
| App port | `5000` |
| Scaling | HPA: 1 to 12 pods. Node group: 1 to 4 nodes, starting at 2 |

## How it works

| Component | Role |
| --- | --- |
| Terraform | Creates the VPC (2 public and 2 private subnets, NAT gateway), EKS, ECR, IAM and the Jenkins EC2 instance |
| Helm | Templates the Deployment, Service, Ingress and HPA |
| ALB controller | Creates the public load balancer from the Ingress |
| Metrics Server | Feeds CPU metrics to the HPA |
| Jenkins (`main`) | Builds the image, pushes to ECR, deploys with Helm |
| GitHub Actions (`gitops`) | Builds and pushes the image, commits the new tag to `k8s/values.yaml` |
| Argo CD (`gitops`) | Syncs the Helm chart from Git to EKS |

## Prerequisites

- AWS account with CLI credentials configured
- Python, Git, Docker Desktop, Terraform, kubectl, Helm
- Commands run in PowerShell from the project root unless noted. Replace the account ID (`060255765406`) with your own.

## Repository structure

| Path | Contents |
| --- | --- |
| `app.py`, `requirements.txt`, `Dockerfile` | Flask app and container |
| `infra/` | Terraform files (kept locally; no state or secrets in Git) |
| `k8s/` | Helm chart: `Chart.yaml`, `values.yaml`, `templates/` |
| `Jenkinsfile` | Jenkins pipeline (`main`) |
| `.github/workflows/` | GitHub Actions workflows (`gitops`) |
| `argocd/application.yaml` | Argo CD application definition |
| `images/` | Architecture diagram |

## Deployment

### 1. Test locally

```powershell
python -m pip install -r requirements.txt
python app.py
```

Stop with **Ctrl+C**, then test the container:

```powershell
docker build -t tech-2a-app .
docker run --rm -p 5000:5000 tech-2a-app
```

Open http://localhost:5000 and confirm **Hello, World!**

### 2. Provision infrastructure

```powershell
aws sts get-caller-identity
cd infra
terraform init
terraform validate
terraform plan
terraform apply
terraform output
cd ..
```

Review the plan before applying. Save the ECR URL, load balancer controller role ARN, VPC ID and Jenkins public IP.

### 3. Push the image to ECR

```powershell
aws ecr get-login-password --region us-west-1 | docker login --username AWS --password-stdin 060255765406.dkr.ecr.us-west-1.amazonaws.com
docker tag tech-2a-app:latest 060255765406.dkr.ecr.us-west-1.amazonaws.com/hello-world:latest
docker push 060255765406.dkr.ecr.us-west-1.amazonaws.com/hello-world:latest
```

### 4. Deploy with Helm

Set `image.repository` and `image.tag` in `k8s/values.yaml` to the pushed image.

```powershell
aws eks update-kubeconfig --region us-west-1 --name tc2-eks
kubectl get nodes
helm lint ./k8s
helm upgrade --install hello-world ./k8s --namespace default --wait --timeout 10m
kubectl rollout status deployment/hello-world
```

### 5. Enable public access and HPA metrics

Get the controller role and VPC ID:

```powershell
aws iam get-role --role-name tc2-eks-lb-controller --query Role.Arn --output text
aws eks describe-cluster --name tc2-eks --region us-west-1 --query cluster.resourcesVpcConfig.vpcId --output text
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install the ALB controller, replacing `YOUR_VPC_ID`:

```powershell
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=tc2-eks --set region=us-west-1 --set vpcId=YOUR_VPC_ID --set serviceAccount.create=true --set serviceAccount.name=aws-load-balancer-controller --set 'serviceAccount.annotations.eks\.amazonaws\.com/role-arn=arn:aws:iam::060255765406:role/tc2-eks-lb-controller'
kubectl rollout status deployment/aws-load-balancer-controller -n kube-system
kubectl get ingress hello-world
```

Open `http://` plus the Ingress **ADDRESS**. Then install Metrics Server:

```powershell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl rollout status deployment/metrics-server -n kube-system
kubectl top pods
kubectl get hpa hello-world
```

## CI/CD

Use one deployment owner at a time: Jenkins **or** Argo CD. Disable the Jenkins deploy job before Argo CD manages the same app.

### 6. Jenkins (`main`)

1. Run Jenkins in Docker on the Terraform-provisioned EC2 instance, with persistent `jenkins_home` and Docker engine access.
2. Install the Docker CLI, AWS CLI, kubectl and Helm in the Jenkins image.
3. Add credentials: `github-read` (GitHub PAT) and `tech2-aws` (**Kind: AWS Credentials**).
4. Create a Pipeline job: **Pipeline script from SCM**, branch `*/main`, script path `Jenkinsfile`.
5. Enable **GitHub hook trigger for GITScm polling** and add a GitHub webhook to `http://YOUR_JENKINS_IP:8080/github-webhook/` (content type `application/json`).
6. Allow Jenkins to reach the EKS API on port 443 and grant its AWS identity EKS deploy access.
7. Push to `main` and verify the build, ECR push, Helm deploy and rollout.

Use the current EC2 IP after any rebuild. Keep credentials out of Git.

### 7. GitHub Actions and Argo CD (`gitops`)

1. Create the `gitops` branch. Keep the Jenkins workflow on `main`.
2. Create the GitHub OIDC provider and an IAM role. Restrict the trust policy to this repository's `gitops` subject, audience `sts.amazonaws.com`, and grant ECR push permissions.
3. Add `.github/workflows/deploy.yml` to sign in through OIDC, build and push a uniquely tagged image, and commit the tag to `k8s/values.yaml`.
4. Install Argo CD in EKS and connect it to the repository (private repos need read credentials).
5. Point `argocd/application.yaml` at branch `gitops`, path `k8s`, namespace `default`, release `hello-world`. Enable automatic sync and let the HPA control replicas.

```powershell
kubectl apply -f argocd/application.yaml
kubectl get applications -n argocd
```

Test it: change the greeting in `app.py`, push to `gitops`, and confirm the Actions run succeeds, the tag changes in Git, and Argo CD shows **Synced and Healthy**.

### 8. Verify

```powershell
kubectl get pods -l app=hello-world
kubectl rollout status deployment/hello-world
kubectl get deployment hello-world -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl get ingress hello-world
```

The deployed image should match the pipeline's tag. Open the ALB URL and check the greeting.

## Notes

- Build and test the container locally before deploying.
- Keep the image tag in `k8s/values.yaml` consistent with ECR.
- The HPA scales pods. Scaling worker nodes needs Cluster Autoscaler, which is not installed.

## Project status

Verified in this rebuild: manual Helm deploy, ALB controller, Metrics Server, Docker build and ECR push.

Still to verify: Jenkins deploy after the EC2 replacement, and the current GitOps setup. Cluster Autoscaler and Trivy scanning are not completed.

## Submission checklist

- [ ] Repository is private and shared with your mentor
- [ ] Terraform source files included, with no state files or secrets
- [ ] `images/architecture.png` uploaded next to this README
- [ ] Current application URL provided
