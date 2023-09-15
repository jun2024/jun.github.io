---
title: "[Terraform] Terraform, Provider, Outputs, Resources, Data sources 이게 다 뭘까?"
excerpt: "언제 쓰는 거지"

categories:
  - Terraform
tags:
  - [Terraform, 테라폼]

permalink: /terraform/what-is-that-keywords/

toc: true
toc_sticky: true

date: 2023-09-13
last_modified_at: 2023-09-14
---

## Terraform 공부 중🤯   
   
Terraform, Provider, Outputs, Resources, Data sources 이게 다 뭘까?      
   
*[Terraform 공식 문서 참고]*(https://developer.hashicorp.com/terraform/language/files)

---

### 1. Terraform Block   

terraform 블록은 Required_providers 혹은 Terraform에 대한 구성을 설정하는데 사용된다고 한다.   
음... 테라폼에 대한 자체적인 설정은 그렇다 치고, Required_providers가 뭔지 모르겠다.   
테라폼 공식문서를 뒤져보자.🙌   

- provider에서 공급자를 지정하기 위해 특정 공급자에 대한 소스 주소 및 버전을 지정할 때 사용

- Q. 근데 왜 그냥 공급자 지정해도 잘 됨?
거의 모든 공급자에는 모든 리소스 유형에 대한 접두사로 사용되는 기본 값이 지정되어 있어서 특정 소스 혹은 특정 버전을 지정하려고 하는게 아니면 안 쓴다고 한다.   

```tf
terraform {

  required_providers {
    aws = {                            # aws라는 provider를 지정
      source  = "hashicorp/aws"        # aws의 소스 주소 지정
      version = "~> 5.7.0"             # aws provider의 버전을 5.7.0 이상으로 지정
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

  required_version = "~> 1.3"          # terraform 최소 필요 버전
}
```

---

### 2. provider Block   

Provider는 Terraform이 클라우드 서비스 공급자, SaaS 공급자 및 기타 API와 상호작용을 하기 위한 플러그인이다.
플러그인을 통해 지정된 Provier는 Terraform이 관리할 수 있는 리소스 유형 및 데이터 소스 세트를 추가한다.
공급자가 없으면 Terraform으로 인프라 관리가 불가능하기 때문에 필수로 작성해야 하고, 여러가지 provider 별로 설정을 해줘야 한다.   
```tf
provider "aws" {
  region   = "ap-northeast-2"
  profile  = "default"
}
```

   ---
   
### 3. Resource Block   

각 리소스 블록은 가상 네트워크, 컴퓨팅 인스턴스 또는 DNS 레코드와 같은 상위 수준 구성 요소와 같은 하나 이상의 인프라 개체를 설명한다.   
리소스 선언에는 다양한 고급 기능들이 포함될 수 있고, 리소스 유형(aws_vpc) & 이름(seoul_vpc_01)은 특정 리소스에 대한 식별자 역할을 하기 때문에 모듈 내에서 고유해야 한다.   
- 참고: 리소스 이름은 문자 또는 밑줄로 시작하고, 문자, 숫자, 밑줄 및 대시만 포함할 수 있다.   

```tf
resource "aws_vpc" "seoul_vpc_01" {
    cidr_block = "10.10.0.0/16"
    enable_dns_hostnames = true
    enable_dns_support = true
    instance_tenancy = "default"
    azs  = slice(data.aws_availability_zones.available.names, 0, 3)

    tags = {
      Name = "seoul_vpc_01"
      Platform = "Terraform"
    }
}
```

---   

### 4. Data Sources
   
Data Sources는 provider를 통해 선언된 공급자의 인프라에 이미 생성되어 있는 리소스를 호출한다고 생각하면 쉽다.
예를 들어서, AWS에 VPC를 생성하기 위해서는 resource 블럭을 사용해야 한다. 그런데 VPC를 생성할때 가용영역(AZ)을 지정할때에는 data 블럭을 사용해야한다.
AWS에서 AZ는 terraform으로 생성하는 것이 아니라 이미 생성되어 있어서 해당 리소스를 호출하기만 하면 된다.


```tf
data "aws_availability_zones" "available" {
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}
```

 ---

   
### 5. Outputs

outputs 블럭은 지정한 값이나 terraform 구성에 대한 정보를 노출할 수 있다.  

 ```tf
output "public_ip" {
  value       = aws_instance.terraform_test.public_ip
  description = "The public IP of the Instance"
}

output "public_dns" {
  value       = aws_instance.terraform_test.public_dns
  description = "The Public dns of the Instance"
}

output "private_ip" {
  value       = aws_instance.terraform_test.private_ip
  description = "The Private_ip of the Instance"
}
```

---   

끝!

***   
   
