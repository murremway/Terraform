# Terraform

The Terraform file that you just executed is divided into two blocks:

main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "2.47.0"

  name                 = "k8s-vpc"
  cidr                 = "172.16.0.0/16"
  private_subnets      = ["172.16.1.0/24", "172.16.2.0/24", "172.16.3.0/24"]
  public_subnets       = ["172.16.4.0/24", "172.16.5.0/24", "172.16.6.0/24"]

  # truncated
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "12.2.0"

  cluster_name    = "${local.cluster_name}"
  cluster_version = "1.17"
  subnets         = module.vpc.private_subnets

  vpc_id = module.vpc.vpc_id

  # truncated
}
The first block is the VPC module.

In this part, you instruct Terraform to create:

A VPC.
Three private and three public subnets.
A single NAT gateway.
Tags for the subnets.
The tags for subnets are quite crucial as those are used by AWS to automatically provision public and internal load balancers in the appropriate subnets.

The second important block in the Terraform file is the EKS cluster module:

main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "2.47.0"

  name                 = "k8s-vpc"
  cidr                 = "172.16.0.0/16"
  private_subnets      = ["172.16.1.0/24", "172.16.2.0/24", "172.16.3.0/24"]
  public_subnets       = ["172.16.4.0/24", "172.16.5.0/24", "172.16.6.0/24"]

  # truncated
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "12.2.0"

  cluster_name    = "${local.cluster_name}"
  cluster_version = "1.17"
  subnets         = module.vpc.private_subnets

  vpc_id = module.vpc.vpc_id

  # truncated
}
The EKS module is in charge of:

Creating the control plane.
Setting up autoscaling groups.
Setting up the proper security groups.
Creating the kubeconfig file.
And more.
Notice how the EKS cluster has to be created into a VPC.

Also, the worker nodes for your Kubernetes cluster should be deployed in the private subnets.

You can define those two constraints with:

main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "2.47.0"

  # truncated
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "12.2.0"

  cluster_name    = "${local.cluster_name}"
  cluster_version = "1.17"
  subnets         = module.vpc.private_subnets

  vpc_id = module.vpc.vpc_id

  # truncated
}
The last part of the Terraform file is the following:

main.tf
data "aws_eks_cluster" "cluster" {
  name = module.eks.cluster_id
}

data "aws_eks_cluster_auth" "cluster" {
  name = module.eks.cluster_id
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
  load_config_file       = false
  version                = "~> 1.11"
}
