# Pathnex EKS on GitLab CI/CD

GitLab tests and builds the Pathnex application, plans Amazon EKS infrastructure
with Terraform, deploys the application by image digest, and installs Prometheus,
Grafana, and Alertmanager. The staging cluster uses a managed Spot node group
and the default VPC in the selected AWS region.

The default deployment is `pathnex-gitlab-staging` in Mumbai (`ap-south-1`).
GitLab-hosted runners execute the pipeline. Your Mac only needs AWS CLI and
`kubectl` if you want local cluster access, plus Python 3 for local app tests.

## 1. Set up the GitLab project

1. Create a **private blank project** named `pathnex-eks-gitlab`. Leave
   **Initialize repository with a README** unchecked when publishing an existing
   local Git repository. [Create a GitLab project](https://docs.gitlab.com/user/project/)
2. Add an SSH public key to GitLab and verify it with
   `ssh -T git@gitlab.com`. Copy the project's **Clone with SSH** URL.
   [GitLab SSH setup](https://docs.gitlab.com/user/ssh/)
3. In **Settings → Repository → Branch rules**, protect `main` and make it the
   default branch. In **Settings → CI/CD → Runners**, verify that GitLab-hosted
   runners are enabled. Enable the project's **Container Registry** if needed.
   [Protected branches](https://docs.gitlab.com/user/project/repository/branches/protected/)
   [Hosted runners](https://docs.gitlab.com/ci/runners/hosted_runners/)

The staging jobs use the protected default branch. The optional AWS identity
check and the trust policy example below use `main`.

## 2. Configure AWS OIDC and the deployment role

In the AWS account that will own EKS, open **IAM → Identity providers**.
Add or reuse an **OpenID Connect** provider with URL `https://gitlab.com` and
audience `sts.amazonaws.com`. Basic GitLab OIDC federation works with
**Trusted identity propagation** left unconfigured. If a shared provider uses
propagation, its required mappings must match real GitLab token claims, such
as `sub`. [GitLab AWS OIDC setup](https://docs.gitlab.com/ci/cloud_services/aws/)

Open **IAM → Roles → Create role**, choose **Custom trust policy**, and use
the following policy. Replace `123456789012` with your AWS account ID.
If your GitLab project path differs, update the `sub` condition to match it.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/gitlab.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "gitlab.com:aud": "sts.amazonaws.com",
          "gitlab.com:sub": "project_path:pathnex/pathnex-eks-gitlab:ref_type:branch:ref:main"
        }
      }
    }
  ]
}
```

Name the role `pathnex-gitlab-terraform` and copy the role ARN for GitLab.
The pipeline requests its own OIDC token and exchanges it for temporary AWS
credentials. There is no AWS access key or OIDC token to generate manually.

Have your AWS administrator attach permissions for the operations below.
The trust policy controls who can assume the role. Its permissions policies
control what the role can do.

| Area | Required access |
| --- | --- |
| Terraform networking | Read the default VPC and its subnets. The VPC must have available subnets in at least two Availability Zones. |
| EKS | Manage the cluster, managed node group, add-ons, and Pod Identity associations defined in the Terraform files. Allow initial creation of EKS service-linked roles where required. |
| IAM | Manage the cluster, worker-node, and Pod Identity roles in `iam.tf`, including policy attachments and `iam:PassRole` scoped to those roles. |
| Terraform state | Access the S3 bucket and exact state and lock objects described below. |
| NodePort ingress | Allow `ec2:DescribeVpcs` and `ec2:DescribeSecurityGroupRules`. Scope `ec2:AuthorizeSecurityGroupIngress` and `ec2:ModifySecurityGroupRules` to the EKS cluster security group. |

The CI role also needs Kubernetes access to deploy the app and cluster-wide
monitoring resources. This lab relies on the cluster creator's access for a
cluster provisioned by the same role. For an existing cluster created by
another identity, grant the CI role appropriate cluster access before deploying.
The runner needs network access to the Kubernetes API. This lab enables the
authenticated public endpoint for hosted runners.

## 3. Create the Terraform state bucket

1. In **AWS → S3 → Create bucket**, pick a globally unique bucket name and note
   its region. Keep **Block all public access** enabled, enable **Bucket
   Versioning**, and use default encryption.
2. Grant the OIDC role `s3:ListBucket` on the bucket scoped to the state
   prefix, and `s3:GetObject` and `s3:PutObject` on
   `pathnex/staging/terraform.tfstate`.
3. Grant `s3:GetObject`, `s3:PutObject`, and `s3:DeleteObject` on
   `pathnex/staging/terraform.tfstate.tflock`. If using a customer-managed KMS
   key, grant access to that key as well. [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)

For an already provisioned cluster, use the same bucket and key used to create
it. If the state is local, migrate it to S3. If another configuration created
the resources, import them into this Terraform state before approving a plan
that proposes creating or replacing them.

## 4. Generate the GitLab registry deploy token

1. Open the GitLab project and go to **Settings → Repository → Deploy tokens**.
2. Create a project deploy token named `pathnex-eks-pull`. Set an expiration
   date that fits your lab schedule and select **`read_registry`**.
3. Click **Create deploy token** and copy both the **Username** and **Token**
   from the confirmation screen. GitLab shows the token value once. The
   username is often in the form `gitlab+deploy-token-123`. Use the two values
   from the same token. [GitLab deploy tokens](https://docs.gitlab.com/user/project/deploy_tokens/)

GitLab provides credentials for the build job to push images automatically.
EKS uses the deploy token to pull images. The deploy job creates or updates the
`gitlab-registry` Kubernetes pull secret from the variables below.

## 5. Add the GitLab CI/CD variables

Go to **Settings → CI/CD → Variables → Add variable**. For each entry, select
**Type: Variable**, **Environment scope: All (default)**, and **Protect
variable**. Use **Masked** or **Masked and hidden** for the token. Paste values
directly without quotes or spaces. Use the pencil icon to edit an existing
entry. [GitLab CI/CD variables](https://docs.gitlab.com/ci/variables/)

| Key | Value | Required |
| --- | --- | --- |
| `AWS_ROLE_ARN` | The full IAM role ARN, for example `arn:aws:iam::123456789012:role/pathnex-gitlab-terraform` | Yes |
| `TF_STATE_BUCKET` | The S3 bucket name, without an ARN or `s3://` prefix | Yes |
| `PATHNEX_REGISTRY_USER` | The deploy token **Username** from step 4 | Yes |
| `PATHNEX_REGISTRY_TOKEN` | The deploy token **Token** from step 4 | Yes |
| `TF_STATE_REGION` | The S3 bucket region if different from `AWS_REGION` | Optional |
| `AWS_REGION` | Override the EKS region. Default `ap-south-1`. | Optional |
| `EKS_CLUSTER_NAME` | Override the cluster name. Default `pathnex-gitlab-staging`. | Optional |
| `PATHNEX_NAMESPACE` | Application namespace. Default `pathnex`. | Optional |
| `PATHNEX_MONITORING_NAMESPACE` | Monitoring namespace. Default `pathnex-monitoring`. | Optional |
| `PATHNEX_APP_NODE_PORT` | Pathnex API NodePort. Default `30081`. | Optional |
| `PATHNEX_APP_SOURCE_CIDR` | Trusted IPv4 CIDR for Pathnex API access. Default is the primary EKS VPC CIDR. | Optional |
| `PATHNEX_GRAFANA_NODE_PORT` | Grafana NodePort. Default `30080`. | Optional |
| `PATHNEX_GRAFANA_SOURCE_CIDR` | A trusted IPv4 CIDR for Grafana access. Default is the primary EKS VPC CIDR, discovered in AWS. | Optional |

`read_registry` is selected on the deploy token. The pipeline looks for the
username and token under the two `PATHNEX_REGISTRY_*` keys above.

## 6. Push and run the pipeline

If the source is still local, commit the project files, add the GitLab SSH
remote, and push `main`. Review the staged files before committing and keep
credentials, plans, and Terraform state outside the commit.

```bash
git status --short
git add .
git diff --cached --name-only
git commit -m "Add Pathnex EKS deployment pipeline"
git remote add origin git@gitlab.com:YOUR_NAMESPACE/pathnex-eks-gitlab.git
git push -u origin main
```

If GitLab already has the project, push your next commit or start a new pipeline
on `main` from **Build → Pipelines**. Open the pipeline to follow the jobs.

| Step | What to expect |
| --- | --- |
| `pathnex:aws_identity` | Optional manual check. Play it to confirm that the AWS output shows your account and the expected assumed role. |
| `pathnex:test` and `pathnex:build` | Tests the API and publishes the container. Find the image under **Deploy → Container Registry**. |
| `pathnex:terraform_plan` | Always plans on protected `main` and saves the result. |
| `pathnex:terraform_review` and `pathnex:terraform_finalize` | No-change plans skip Terraform apply and continue automatically. When changes exist, review the parent plan log, open the review child pipeline, and play `pathnex:approve_terraform_apply`. The parent applies the saved plan. |
| `pathnex:deploy_staging` | Verifies the deploy token can read the exact image, then deploys Pathnex with Helm. |
| `pathnex:deploy_monitoring` | Installs Prometheus, Grafana, and Alertmanager with Helm. |
| `pathnex:expose_pathnex` | Adds or updates the AWS security-group rule for the Pathnex API NodePort. |
| `pathnex:expose_grafana` | Creates the Grafana NodePort Service and adds or updates the AWS security-group rule. |

## 7. Verify Pathnex and Grafana

For local Kubernetes access, an AWS administrator can add your IAM user or role
under **EKS → cluster → Access → Create access entry**. If the cluster is in
`CONFIG_MAP` mode, enable **EKS API and ConfigMap** first. That preserves existing
ConfigMap access. Choose permissions for the tasks you need. `AmazonEKSViewPolicy`
can list pods. Node listing requires a separate Kubernetes grant or a policy that
includes nodes. Reading the Grafana password requires access to its Secret.
[EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
[EKS access policy permissions](https://docs.aws.amazon.com/eks/latest/userguide/access-policy-permissions.html)

```bash
aws sts get-caller-identity
aws eks update-kubeconfig --region ap-south-1 --name pathnex-gitlab-staging
kubectl -n pathnex get pods
kubectl -n pathnex-monitoring get pods
kubectl -n pathnex-monitoring get svc pathnex-grafana-nodeport
kubectl -n pathnex get svc pathnex
```

Open `http://<reachable-node-ip>:30081/` to see the Pathnex API and the deployed
commit SHA. Open `http://<reachable-node-ip>:30080` for Grafana. The exposure jobs
print the node addresses. Both NodePorts allow the primary VPC CIDR by default.
Use a VPC-connected client or a VPN with an allowed source address. Direct
internet access requires a reachable public node address and the respective
optional source CIDR set to your public IP with `/32`.

Sign in as `admin`. If you need the initial password, read it in a local
terminal with an authorized Kubernetes identity:

```bash
kubectl -n pathnex-monitoring get secret pathnex-monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d
echo
```

The monitoring stack collects cluster metrics. Pathnex-specific metrics require
application instrumentation and a ServiceMonitor.

## Troubleshooting

- **A protected variable is missing.** Confirm the exact key, **Variable**
  type, **All** environment scope, and that `main` is protected.
- **STS rejects the OIDC token.** Compare the audience and exact project and
  branch subject in IAM. For `Required source key does not exist in identity
  token`, inspect any trusted identity propagation mappings on the provider.
- **Image pull returns 401 or 403.** Check that the deploy token is active and
  has `read_registry`. Update both GitLab values from the same token and retry
  `pathnex:deploy_staging`.
- **A NodePort exposure job fails with AWS access denied.** Check the role's EC2
  discovery and security-group rule permissions in step 2, then retry
  the failed exposure job.
- **The Terraform plan wants to recreate an existing cluster.** Check the state
  bucket, key, region, and cluster name before approving apply.

## Local development

Run the application and tests locally:

```bash
python3 -m unittest discover -s pathnex -v
python3 pathnex/server.py
```

From another terminal, run `curl -fsS http://localhost:8080/`. To build locally
with Docker, run `docker build -t pathnex:local pathnex`. To verify the Helm
chart, run `helm lint deploy/pathnex`.

## Destroy the lab from GitLab

When you are finished with the lab, open the latest completed pipeline on
protected `main`. In the **destroy** stage, click **Play** on
`pathnex:destroy_lab`. That is the only action needed. The teardown child
pipeline creates a destroy plan from the same staging state, removes the
Grafana NodePort and both Helm releases, and applies the saved destroy plan.
Open the child pipeline to follow the logs and check that
`pathnex:terraform_destroy` succeeds.

The destroy button is optional, so a normal deployment can complete without
teardown. A new deployment pipeline can propose recreating the lab after it
has been destroyed.

The OIDC role needs delete permissions for the lab resources in Terraform
state. If the cleanup job finds a LoadBalancer Service or Ingress, remove
the lab-owned load balancer resources first while EKS can reconcile them,
then retry the cleanup job. The state bucket, OIDC provider, and deploy
token are shared setup resources outside this destroy plan.

### Local teardown fallback

EKS and worker nodes incur AWS charges. To remove the lab, first remove the
Grafana NodePort Service and uninstall the Helm releases with a Kubernetes
identity authorized for those operations:

```bash
kubectl -n pathnex-monitoring delete svc pathnex-grafana-nodeport --ignore-not-found
helm uninstall pathnex-monitoring -n pathnex-monitoring
helm uninstall pathnex -n pathnex
```

Then, using Terraform 1.10 or newer and an authorized AWS identity, initialize
against the **same** state bucket, bucket region, and key used by CI. Match the
region and cluster name in GitLab before reviewing the destroy plan:

```bash
cp -n terraform.tfvars.example terraform.tfvars
terraform init \
  -backend-config="bucket=YOUR_STATE_BUCKET" \
  -backend-config="region=YOUR_STATE_BUCKET_REGION" \
  -backend-config="key=pathnex/staging/terraform.tfstate"
terraform plan -destroy -out=pathnex-destroy.tfplan
terraform apply pathnex-destroy.tfplan
```

Review the destroy plan before running the final command. Retain the state
bucket until AWS confirms cleanup. Revoke the GitLab deploy token when the lab
no longer needs registry access.

## Project layout

| Path | Purpose |
| --- | --- |
| `.gitlab-ci.yml` and `ci/` | GitLab jobs, conditional Terraform review, and NodePort security-group rules |
| `*.tf` and `terraform.tfvars.example` | Remote-state configuration and EKS, IAM, node group, and add-on definitions |
| `pathnex/` | API, tests, and Dockerfile |
| `deploy/pathnex/` | Application Helm chart |
| `deploy/monitoring/` | Monitoring Helm values |
