## 1. 개요

* **사용 기술**: Terraform, Azure CLI, AKS, kubectl
* **목표**:

  * Terraform으로 Azure 리소스를 자동 배포
  * AKS 클러스터 생성
  * 샘플 마이크로서비스 애플리케이션 배포

---

## 2. Terraform 코드 구성

| 파일명            | 내용 설명                   |
| -------------- | ----------------------- |
| `providers.tf` | 필요한 provider 선언 및 버전 지정 |
| `main.tf`      | 리소스 그룹 및 AKS 클러스터 생성    |
| `ssh.tf`       | SSH 키 생성 (AzAPI 사용)     |
| `variables.tf` | 입력 변수 정의                |
| `outputs.tf`   | 클러스터 정보 출력              |

> 주요 리소스: `azurerm_kubernetes_cluster`, `azapi_resource`, `random_pet`

---

## 3. 실습 단계

### 디렉토리 준비 및 코드 작성

```bash
mkdir terraform-aks && cd terraform-aks
# 위에서 제공된 .tf 파일들 작성
```

### 초기화 및 실행 계획 생성

```bash
terraform init -upgrade
terraform plan -out main.tfplan
```

### 실행 계획 적용

```bash
terraform apply main.tfplan
```

### 클러스터 정보 추출 및 kubeconfig 설정

```bash
echo "$(terraform output kube_config)" > ./azurek8s
export KUBECONFIG=./azurek8s
kubectl get nodes
```

---

## 4. 결과 확인 및 애플리케이션 배포

### 🐰 RabbitMQ, 🛒 Order Service 등 샘플 배포

* `aks-store-quickstart.yaml` 작성 및 배포

```bash
kubectl apply -f aks-store-quickstart.yaml
kubectl get all
```

![image](https://github.com/user-attachments/assets/0f67b7e2-e787-48dd-8624-13a7c68a303e)
![image](https://github.com/user-attachments/assets/75ffe15c-07d8-43fb-84d7-077f8586c55c)


---
## 5. 앞으로 도전과제
azure 학생용 계정은 GPU를 클러스터 노드에 할당받지 못한다는 제약 사항 발생, 따라서 GPU node를 어떻게 얻을 수 있는지 방법을 찾는 중

---

## 📚 참고 문서

* [공식 빠른 시작 문서 (Azure Docs)](https://learn.microsoft.com/ko-kr/azure/aks/terraform-aks)
* [Terraform Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)


