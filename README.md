Note:

- KMS key module will need to be applied first with `terraform apply -target=module.kms`

## State Move

```
terraform state mv 'module.eks_blueprints.module.aws_eks_managed_node_groups["mg_4"].aws_iam_role_policy_attachment.managed_ng_AmazonEC2ContainerRegistryReadOnly' 'module.eks_blueprints.module.aws_eks_managed_node_groups["mg_4"].aws_iam_role_policy_attachment.managed_ng["arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"]'

terraform state mv 'module.eks_blueprints.module.aws_eks_managed_node_groups["mg_4"].aws_iam_role_policy_attachment.managed_ng_AmazonEKSWorkerNodePolicy' 'module.eks_blueprints.module.aws_eks_managed_node_groups["mg_4"].aws_iam_role_policy_attachment.managed_ng["arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"]'

terraform state mv 'module.eks_blueprints.module.aws_eks_managed_node_groups["mg_4"].aws_iam_role_policy_attachment.managed_ng_AmazonEKS_CNI_Policy' 'module.eks_blueprints.module.aws_eks_managed_node_groups["mg_4"].aws_iam_role_policy_attachment.managed_ng["arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"]'

terraform state mv 'module.eks_blueprints.module.aws_eks_managed_node_groups["mg_4"].aws_iam_role_policy_attachment.managed_ng_AmazonSSMManagedInstanceCore' 'module.eks_blueprints.module.aws_eks_managed_node_groups["mg_4"].aws_iam_role_policy_attachment.managed_ng["arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"]'
```


## Diff

```diff
provider "aws" {
  region = local.region
}

provider "kubernetes" {
  host                   = module.eks_blueprints.eks_cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks_blueprints.eks_cluster_certificate_authority_data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    # This requires the awscli to be installed locally where Terraform is executed
    args = ["eks", "get-token", "--cluster-name", module.eks_blueprints.eks_cluster_id]
  }
}

provider "helm" {
  kubernetes {
    host                   = module.eks_blueprints.eks_cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks_blueprints.eks_cluster_certificate_authority_data)

    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      # This requires the awscli to be installed locally where Terraform is executed
      args = ["eks", "get-token", "--cluster-name", module.eks_blueprints.eks_cluster_id]
    }
  }
}

data "aws_caller_identity" "current" {}
data "aws_availability_zones" "available" {}

locals {
  tenant      = "hsaap"
  environment = "sandbox"
  zone        = "sydney"
  region      = "us-west-2"

  name = join("-", [local.tenant, local.environment, local.zone])

  vpc_cidr = "10.0.0.0/16"
  azs      = slice(data.aws_availability_zones.available.names, 0, 3)

  tags = {
    Foo         = "Bar"
+   created-by  = "Terraform v1.0.1"
+   environment = local.environment
+   tenant      = local.tenant
+   zone        = local.zone
+   org         = ""
+   resource    = "eks"
+   name        = "${local.name}-eks"
  }
}

module "eks_blueprints" {
-  source = "github.com/aws-ia/terraform-aws-eks-blueprints?ref=v4.0.4"
+  source = "github.com/aws-ia/terraform-aws-eks-blueprints?ref=v4.0.7"

-  tenant            = local.tenant
-  environment       = local.environment
-  zone              = local.zone
-  terraform_version = "Terraform v1.0.1"

  # EKS CONTROL PLANE VARIABLES
  cluster_name    = local.name
  cluster_version = "1.21"

+ iam_role_name = "${local.name}-eks-cluster-role"

  # EKS Cluster VPC and Subnet mandatory config
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnets

  cluster_kms_key_arn = module.kms.key_arn

  # on initial build set the public_access to true
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  # Workspace access API endpoint
  cluster_security_group_additional_rules = {
    ingress_workspace_api = {
      description = "AWS Workspace access to API endpoint"
      protocol    = "TCP"
      from_port   = 443
      to_port     = 443
      type        = "ingress"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  # API access to Nodes required by ALB controller
  node_security_group_additional_rules = {
    ingress_self_all = {
      description                   = "ALB WebHook"
      protocol                      = "TCP"
      from_port                     = 9443
      to_port                       = 9443
      type                          = "ingress"
      source_cluster_security_group = true
    }
  }

  # EKS MANAGED NODE GROUPS
  managed_node_groups = {
    mg_4 = {
      node_group_name = "managed-ondemand"
      instance_types  = ["c5.large"]
      desired_size    = 1
      max_size        = 3
      min_size        = 1
    }
  }

  # Dynamically generated application teams (see settings & locals)
  application_teams = {
    team-red = {
      "labels" = {
        "appName"     = "read-team-app",
        "projectName" = "project-red",
        "environment" = "example",
        "domain"      = "example",
        "uuid"        = "example",
        "billingCode" = "example",
        "branch"      = "example"
      }
      "quota" = {
        "requests.cpu"    = "1000m",
        "requests.memory" = "4Gi",
        "limits.cpu"      = "2000m",
        "limits.memory"   = "8Gi",
        "pods"            = "10",
        "secrets"         = "10",
        "services"        = "10"
      }
      users = [
        data.aws_caller_identity.current.arn
      ]
    }
  }

  # platform teams
  platform_teams = {
    hsaap-platform-dev = {
      "labels" = {
        "appName"     = "hsaap-platform-app",
        "projectName" = "hsaap-platform-dev",
      }
      users = [
        data.aws_caller_identity.current.arn,
      ]
    }
  }

  tags = local.tags
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.0"

  name = local.name
  cidr = local.vpc_cidr
  azs  = local.azs

  public_subnets  = [for k, v in local.azs : cidrsubnet(local.vpc_cidr, 8, k)]
  private_subnets = [for k, v in local.azs : cidrsubnet(local.vpc_cidr, 8, k + 10)]

  enable_nat_gateway   = true
  create_igw           = true
  enable_dns_hostnames = true
  single_nat_gateway   = true

  public_subnet_tags = {
    "kubernetes.io/cluster/${local.name}" = "shared"
    "kubernetes.io/role/elb"              = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${local.name}" = "shared"
    "kubernetes.io/role/internal-elb"     = "1"
  }
}

module "kms" {
  source  = "terraform-aws-modules/kms/aws"
  version = "~> 1.3"

  # Aliases
  aliases = [local.name]
}
```