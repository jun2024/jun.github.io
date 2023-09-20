---
title: "[Terraform] Terraform으로 EKS 구축하기"
excerpt: "그대로 따라하면 다 되더라"

categories:
  - Terraform
tags:
  - [Terraform, EKS, Terraform cloud]

permalink: /terraform/create-eks-using-terraform/

toc: true
toc_sticky: true

date: 2023-09-19
last_modified_at: 2023-09-19
---

## 🦥 Terraform으로 EKS 구축하기

이번엔 Terraform으로 EKS를 구축하려고 한다.   
Hashicorp에서 아주 친절하게 EKS 구축을 위한 가이드를 제공하고 있다!!   
그것만 따라서 구축하면 금방할 수 있다!!   

---   

*[Terraform 공식 문서 참고]*(https://developer.hashicorp.com/terraform/tutorials/kubernetes/eks#optional-configure-terraform-kubernetes-provider)   

사전에 AWS Credential 연결, Terraform cloud 연동, AWS CLI 설치, kubectl 설치 등 진행해야 할 작업들이 있다.   
해당 작업들은 공식 문서를 따라서 진행하면 금방 할 수 있어서 실제 EKS 구축으로 바로 들어간다!   

1. Terraform Provider를 지정한다.   
AWS를 사용하기 위해서 hashicorp/aws를 공급자로 지정해주고, 나머지는 필요에 따라 추가해준다.   

```tf
terraform {

  cloud {
    organization = "jun2024"
    workspaces {
      name = "learn-terraform-eks"
    }
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.7.0"
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.5.1"
    }

    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0.4"
    }

    cloudinit = {
      source  = "hashicorp/cloudinit"
      version = "~> 2.3.2"
    }
  }

  required_version = "~> 1.3"
}
```

2. EKS 클러스터 프로비저닝을 위한 모듈 설정   

`main.tf`를 이용해서 VPC 및 EKS * Add-on까지 생성한다.   
EKS 관련해서 Kube-proxy & CoreDNS & VPC-CNI addon을 추가했다.   
```tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.15.3"

  cluster_name    = local.cluster_name
  cluster_version = "1.27"

   # EKS Add-On 정의
  cluster_addons = {
    coredns = {
      most_recent                 = true
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "OVERWRITE"
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent                 = true
      before_compute              = true  # worker-node 보다 vpc-cni가 먼저 배포되어서 pod IP 할당 이슈가 발생하지 않게 한다.
      resolve_conflicts_on_create = "OVERWRITE"
      resolve_conflicts_on_update = "OVERWRITE"
      service_account_role_arn    = module.irsa_vpc_cni.iam_role_arn  # IRSA(k8s ServiceAccount에 IAM 역할을 사용한다)
      configuration_values        = jsonencode({
        env = {
          ENABLE_PREFIX_DELEGATION = "true"  # prefix assignment mode 활성화
          WARM_PREFIX_TARGET       = "1"  # 기본 권장 값
        }
      })
    }
  }

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_group_defaults = {
    ami_type = "AL2_x86_64"

  }

  eks_managed_node_groups = {
    one = {
      name = "node-group-1"

      instance_types = ["t3.small"]

      min_size     = 1
      max_size     = 6
      desired_size = 2

      vpc_security_group_ids = [
        aws_security_group.node_group_one.id
      ]
    }

    two = {
      name = "node-group-2"

      instance_types = ["t3.small"]

      min_size     = 1
      max_size     = 6
      desired_size = 2

      vpc_security_group_ids = [
        aws_security_group.node_group_two.id
      ]
    }
  }
}
```

3. AWS Region 설정   
`variables.tf`에서 AWS Region을 서울 리전으로 수정했다.   
```tf
variable "region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-2"
}
```

4. terraform init & plan & apply 를 통해 적용   
terraform CLI를 통해 실행시켜주고, 기다리면 EKS 생성이 완료된다!   
```bash
$ terraform init
$ terraform plan
$ terraform apply
```

5. EKS에 생성된 클러스터 확인   
kubectl로 k8s의 클러스터에 접근하기 위해서 생성된 클러스터를 등록합니다.   
```bash
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)
```
등록 후 `kubectl config current-context`명령어로 클러스터 등록 확인   
```bash
$ terraform output -raw cluster_name
terraform-eks-nQRQ

$ kubectl config current-context
arn:aws:eks:ap-northeast-2:255825666737:cluster/terraform-eks-nQRQ
```   

6. 생성된 클러스터와 Node 정보 확인   
```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://{control-plane}.yl4.ap-northeast-2.eks.amazonaws.com
CoreDNS is running at https://{control-plane}.yl4.ap-northeast-2.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes
NAME                                              STATUS   ROLES    AGE     VERSION
ip-10-10-1-193.ap-northeast-2.compute.internal    Ready    <none>   4h35m   v1.27.4-eks-8ccc7ba
ip-10-10-1-87.ap-northeast-2.compute.internal     Ready    <none>   4h36m   v1.27.4-eks-8ccc7ba
ip-10-10-11-60.ap-northeast-2.compute.internal    Ready    <none>   4h36m   v1.27.4-eks-8ccc7ba
ip-10-10-21-171.ap-northeast-2.compute.internal   Ready    <none>   4h35m   v1.27.4-eks-8ccc7ba
```

node 정보까지 잘 뜬다면 EKS 구축은 잘 되었다고 보면 된다!   

7. 생성한 리소스 삭제   
기존에 cloudformation을 사용할때도 생성한 스택을 삭제하면 모든 리소스를 dependency따라서 차례로 삭제했었는데, Terraform도 명령어 한 줄이면 생성한 모든 리소스를 삭제시킨다!   

```bash
$ terraform destroy
```

입력 후 AWS에서 VPC / EC2 / EKS 서비스 등에서 확인해보면 모든 리소스가 삭제되었다! 🙌


---

다음번에는 여기에 테스트로 nginx를 올려볼건데, ingress도 같이 올려보겠다!!