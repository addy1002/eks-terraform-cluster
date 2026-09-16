# EKS Cluster on AWS — Provisioned with Terraform

Production-style Amazon EKS cluster built entirely with Terraform: VPC, IAM, EKS control plane, and a managed node group — deployed end-to-end with a containerized Flask app to prove the pipeline works.

## Architecture

```
                        Internet
                            |
                     Internet Gateway
                            |
                    ┌───────────────┐
                    │      VPC       │  10.0.0.0/16
                    │  (ap-south-1)  │
                    └───────┬────────┘
                            |
        ┌───────────────────┴───────────────────┐
        │                                         │
  Public Subnet A                          Public Subnet B
  10.0.1.0/24 (ap-south-1a)          10.0.2.0/24 (ap-south-1b)
        │                                         │
        └───────────────┬─────────────────────────┘
                         |
                 EKS Control Plane
              (aws_eks_cluster, K8s 1.36)
                         |
              EKS Managed Node Group
             (2x t3.micro worker nodes)
                         |
              Kubernetes Deployment
           (Flask app, 2 replicas, image
            pulled from Amazon ECR)
                         |
               LoadBalancer Service
              (AWS Elastic Load Balancer)
                         |
                    End user / curl
```

## What's provisioned

| Component | Details |
|---|---|
| Remote state | S3 backend + DynamoDB lock table (state locking, safe concurrent applies) |
| Networking | VPC, 2 public subnets across AZs, Internet Gateway, route table + associations |
| IAM | Cluster role (`AmazonEKSClusterPolicy`) and node role (`AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`) |
| EKS Control Plane | Kubernetes 1.36 (current standard-support version) |
| Node Group | Managed node group, `t3.micro` (Free Tier eligible), autoscaling 1–3 nodes |
| App | Flask container built with Docker, pushed to Amazon ECR, deployed via Kubernetes Deployment + LoadBalancer Service |

## Project structure

```
.
├── main.tf                    # all resources: VPC, IAM, EKS, node group
├── variables.tf                # input variable declarations
├── terraform.tfvars.example    # sample values — copy to terraform.tfvars and fill in
├── terraform.tf                 # backend + provider configuration
└── .gitignore                  # excludes state files, tfvars, plan files
```

No hardcoded values — every environment-specific value (region, CIDR blocks, cluster name, instance types, scaling limits) is driven through `variables.tf` + `terraform.tfvars`.

## How to deploy

```bash
git clone https://github.com/addy1002/eks-terraform-cluster.git
cd eks-terraform-cluster
cp terraform.tfvars.example terraform.tfvars
# edit terraform.tfvars with your own values

terraform init
terraform plan
terraform apply
```

Cluster provisioning takes ~5–6 minutes for the control plane, ~2 minutes for the node group.

### Deploying the sample app

```bash
aws eks update-kubeconfig --region <your-region> --name <your-cluster-name>
kubectl apply -f flask-app/deployment.yaml
kubectl get svc addy-flask-app-service   # grab the EXTERNAL-IP
curl http://<EXTERNAL-IP>
```

## Troubleshooting story: state drift from session switching

**Problem:** While managing this cluster from both AWS CloudShell and this EC2 instance interchangeably, the node group's `desired_size` was occasionally changed outside of Terraform (manual `kubectl scale` / console edits during testing). On the next `terraform apply`, Terraform detected this as drift and tried to force the node count back to the value in `terraform.tfvars` — fighting any manual scaling done for testing.

**Fix:** Added a `lifecycle` block on the node group resource:

```hcl
lifecycle {
  ignore_changes = [scaling_config[0].desired_size]
}
```

This tells Terraform to stop tracking `desired_size` as a managed attribute after initial creation, so manual or autoscaler-driven changes to node count don't get reverted on the next apply — while `min_size` and `max_size` boundaries are still enforced from code. This is the same pattern used in production when an autoscaler or manual intervention needs authority over a specific field that Terraform would otherwise fight.

## Cost notes

- EKS control plane: **not covered by AWS Free Tier** — charged $0.10/hour (or $0.60/hour if the Kubernetes version falls out of standard support)
- `t3.micro` worker nodes: Free Tier eligible (750 hrs/month)
- **Run `terraform destroy` when done testing** to avoid ongoing charges

## Tech stack

Terraform · AWS (VPC, EKS, IAM, ECR, EC2) · Kubernetes · Docker · Flask
## Troubleshooting story: orphaned load balancer blocking `terraform destroy`

**Problem:** After deploying the Flask app with a Kubernetes `Service` of `type: LoadBalancer`, AWS provisioned a Classic Load Balancer and an associated security group (`k8s-elb-...`) automatically — outside of Terraform's knowledge, since these were created by the AWS cloud-controller-manager reacting to the Kubernetes Service object, not by a Terraform resource.

When running `terraform destroy`, the subnets and Internet Gateway got stuck in a "Still destroying..." loop for over 15 minutes. The root cause: the orphaned ELB's network interfaces were still attached to the subnets, and its security group was still referencing the VPC — both invisible to Terraform's dependency graph.

**Fix:**
1. Identify the leftover load balancer: `aws elb describe-load-balancers`
2. Delete it: `aws elb delete-load-balancer --load-balancer-name <name>`
3. Delete the orphaned security group it created: `aws ec2 delete-security-group --group-id <id>`
4. `terraform destroy` automatically resumed and completed within 2–3 minutes — no restart needed, since Terraform retries in the background.

**Takeaway:** Any Kubernetes resource that provisions cloud infrastructure directly (LoadBalancer Services, EBS-backed PersistentVolumes, etc.) creates resources Terraform doesn't track. Always delete Kubernetes-managed cloud resources (`kubectl delete service <name>`, or drain the workload) **before** running `terraform destroy` on the underlying cluster — otherwise you'll hit this exact dependency deadlock.
