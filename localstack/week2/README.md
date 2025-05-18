## 2주차 LocalStack

> 목표 :  실습을 통해 LocalStack 익숙해지기

### LocalStack 기본 환경 설정

※ 실습을 위해서는 Local Stack, AWS CLI(옵션), awslocal, docker가 설치되어 있어야 함

1. **Local Stack 설치** 
- 해당 링크에서 LocalStack 설치
- https://github.com/localstack/localstack-cli/releases/tag/v4.4.0

2. **LocalStack AWS CLI(`awslocal`) 설치**

    ```bash
    pip install awscli-local[ver1]
    ```

- LocalStack에서 제공하는 S3, Lambda, DynamoDB 등 로컬 AWS 서비스에 명령어로 접근하기 위해서는 `awslocal`이 필요함
- AWS CLI와 비슷하게 사용이 되지만, 내부적으로 `--endpoint-url=http://localhost:4566`을 자동으로 붙여주는 편리함
    
    ```bash
    # 아래 두 명령어는 동일한 역할
    aws --endpoint-url=http://localhost:4566 s3 ls
    
    awslocal s3 ls
    ```
    
- 만약 aws cli로 접근시, `aws configure`에서 여러 설정들을 기본적으로 지정해줘야 정상 작동 함
    
    ```bash
    $ aws configure
    AWS Access Key ID [****************test]: test
    AWS Secret Access Key [****************test]: test
    Default region name [us-east-1]: us-east-1
    Default output format [None]: 
    ```
    

3. **Docker desktop 실행**
    - LocalStack의 경우 기본적으로 도커 컨테이너 위에서 작동하기 때문에 도커가 돌아가는 환경이어야 함
    

### 실습 폴더 생성

1. **프로젝트 루트 폴더에서 Localstack 실행**
    
    ```bash
    $ localstack start -d
    
         __                     _______ __             __
        / /   ____  _________ _/ / ___// /_____ ______/ /__
       / /   / __ \/ ___/ __ `/ /\__ \/ __/ __ `/ ___/ //_/
      / /___/ /_/ / /__/ /_/ / /___/ / /_/ /_/ / /__/ ,<
     /_____/\____/\___/\__,_/_//____/\__/\__,_/\___/_/|_|
    
    - LocalStack CLI: 4.4.0
    - Profile: default
    - App: https://app.localstack.cloud
    
    [17:00:15] starting LocalStack in Docker mode 🐳               localstack.py:512
               preparing environment                               bootstrap.py:1322
               configuring container                               bootstrap.py:1330
               starting container                                  bootstrap.py:1340
    [17:00:16] detaching                                           bootstrap.py:1344
    ```
    
    - Localstack에서 실행 가능한 서비스 아래와 같이 확인 가능
        
    ```bash
    $ localstack status services
    
    ┏━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━┓
    ┃ Service                  ┃ Status      ┃
    ┡━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━┩
    │ acm                      │ ✔ available │
    │ apigateway               │ ✔ available │
    │ cloudformation           │ ✔ available │
    │ cloudwatch               │ ✔ available │
    │ config                   │ ✔ available │
    │ dynamodb                 │ ✔ available │
    │ dynamodbstreams          │ ✔ available │
    │ ec2                      │ ✔ available │
    │ es                       │ ✔ available │
    │ events                   │ ✔ available │
    │ firehose                 │ ✔ available │
    ...
    ```
        

### S3 정적 웹사이트 배포 실습

1. **정적 웹사이트 역할의 `index.html` 파일 생성**
    
    ```bash
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta http-equiv="Content-Type" content="text/html" />
        <meta  charset="utf-8"  />
        <title>Static Website</title>
      </head>
      <body>
        <p>Static Website deployed locally over S3 using LocalStack</p>
        <p>안녕하세요</p>
      </body>
    </html>
    ```
    

1. **`testwebsite`라는 S3 버킷을 생성**
    
    ```bash
    $ awslocal s3api create-bucket --bucket testwebsite
    ```
    
    - `localstack status services` 확인 시 S3의 경우 running 상태로 변경되어있음
    <br/><br/>
2. **버킷 액세스 정책**
    - `bucket_policy.json` 파일 생성
        
        ```bash
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Sid": "PublicReadGetObject",
              "Effect": "Allow",
              "Principal": "*",
              "Action": "s3:GetObject",
              "Resource": "arn:aws:s3:::testwebsite/*"
            }
          ]
        }
        ```
        
        - 모든 사용자가 버킷 안 객체에 대해 `GetObject` 읽기 요청 가능한 정책
    - 정책 적용
        
        ```bash
        $ awslocal s3api put-bucket-policy --bucket testwebsite --policy file://bucket_policy.json
        ```
        <br/>

1. **정적 웹사이트 파일 업로드**
    
    ```bash
    awslocal s3 sync ./ s3://testwebsite
    ```
    
    - 현재 디렉토리(`./`)에 있는 파일들을 S3 버킷에 업로드
    - `sync`는 디렉토리 내용을 S3와 동기화하는 명령어
    - 만약 `index.html` 파일을 수정하고 새로 반영하고 싶어도, 해당 명령어를 입력해주면 됨
    <br/><br/>

1. **정적 웹사이트 호스팅 설정**
    
    ```bash
    awslocal s3 website s3://testwebsite/ --index-document index.html
    ```
    
    - 버킷에 정적 웹사이트 호스팅 기능 활성화
    - index.html 파일을 설정
    <br/><br/>

1. **웹사이트 접속**
    - 실제 AWS 환경에서 S3 정적 웹사이트 배포 시 두 가지 방식으로 전용 URL(endpoint) 제공 ⇒ `http://<BUCKET_NAME>.s3-website-<REGION>.amazonaws.com` or `http://<BUCKET_NAME>.s3-website.<REGION>.amazonaws.com`
    - 하지만 LocalStack 환경에서는 이를 모방해 로컬에서도 정적 웹사이트를 볼 수 있도록 하며, S3 Website Endpoint를 별도로 제공 ⇒ `http://<BUCKET_NAME>.s3-website.localhost.localstack.cloud:4566`
    - LocalStack은 `localhost.localstack.cloud` 도메인과 기본 포트로 4566을 사용함

<br/>

---
<br/>

## 동일한 실습 Terraform 코드화

1. **Provider 설정**
    - Terraform이 LocalStack을 AWS처럼 인식하도록 provider 설정
        
        ```hcl
        provider "aws" {
          access_key                  = "test"
          secret_key                  = "test"
          region                      = "us-east-1"
        
          s3_use_path_style           = false
          skip_credentials_validation = true
          skip_metadata_api_check     = true
          skip_requesting_account_id  = true
        
          endpoints {
            s3 = "http://s3.localhost.localstack.cloud:4566"
          }
        }
        ```
        
    - 실제 AWS 인증을 필요로 하지 않기 때문에, 임의의 키를 사용해도 무방
    - `endpoints`를 통해 LocalStack의 S3 주소를 지정해줌

1. **변수 지정 - variables**
    
    ```hcl
    variable "bucket_name" {
      description = "Name of the s3 bucket. Must be unique."
      type        = string
    }
    
    variable "tags" {
      description = "Tags to set on the bucket."
      type        = map(string)
      default     = {}
    }
    ```
    

1. **출력값 설정 - outputs**
    
    ```hcl
    output "arn" {
      value = aws_s3_bucket.s3_bucket.arn
    }
    output "name" {
      value = aws_s3_bucket.s3_bucket.id
    }
    output "domain" {
      value = aws_s3_bucket_website_configuration.s3_bucket.website_domain
    }
    output "website_endpoint" {
      value = aws_s3_bucket_website_configuration.s3_bucket.website_endpoint
    }
    ```
    
    - ARN, 도메인, 웹사이트 접속 주소 등을 확인할 수 있게 outputs.tf 파일 생성

1. **리소스 정의 → main.tf**
    
        

    **S3 버킷 생성**

    ```hcl
    resource "aws_s3_bucket" "s3_bucket" {
    bucket = var.bucket_name
    tags   = var.tags
    }
    ```

    **정적 웹사이트 설정**

    ```hcl
    resource "aws_s3_bucket_website_configuration" "s3_bucket" {
    bucket = aws_s3_bucket.s3_bucket.id

    index_document {
        suffix = "index.html"
    }

    error_document {
        key = "error.html"
    }
    }

    ```

    **버킷 정책 및 ACL 설정**

    ```hcl
    resource "aws_s3_bucket_acl" "s3_bucket" {
    bucket = aws_s3_bucket.s3_bucket.id
    acl    = "public-read"
    }

    resource "aws_s3_bucket_policy" "s3_bucket" {
    bucket = aws_s3_bucket.s3_bucket.id

    policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
        {
            Sid       = "PublicReadGetObject"
            Effect    = "Allow"
            Principal = "*"
            Action    = "s3:GetObject"
            Resource = [
            aws_s3_bucket.s3_bucket.arn,
            "${aws_s3_bucket.s3_bucket.arn}/*",
            ]
        }
        ]
    })
    }

    ```

    **HTML 파일 업로드**

    ```hcl
    resource "aws_s3_object" "object_www" {
    depends_on   = [aws_s3_bucket.s3_bucket]
    for_each     = fileset("${path.root}", "*.html")
    bucket       = var.bucket_name
    key          = basename(each.value)
    source       = each.value
    etag         = filemd5("${each.value}")
    content_type = "text/html"
    acl          = "public-read"
    }

    ```

    **객체 업로드**

    ```hcl
    resource "aws_s3_object" "object_assets" {
    depends_on = [aws_s3_bucket.s3_bucket]
    for_each   = fileset(path.module, "assets/*")
    bucket     = var.bucket_name
    key        = each.value
    source     = "${each.value}"
    etag       = filemd5("${each.value}")
    acl        = "public-read"
    }
    ```

1. **Terraform 명령어 실행**

    ```bash
    terraform init         # 초기화
    terraform plan         # 실행 계획 확인
    terraform apply        # 인프라 생성 및 배포
    ```

    -  이후 지정한 `http://<버킷명>.s3-website.localhost.localstack.cloud:4566` 접근 시 정적 웹사이트 접속 가능

**※ Tflocal 명령어**

- LocalStack 전용 CLI인 `tflocal`을 쓸 경우 `provider.tf`을 설정해주지 않아도 작동 가능
- Terraform 디렉토리에 임시 provider 파일을 생성하여 LocalStack 아래와 같은 전용 provider 설정 포함
    
    ```hcl
    provider "aws" {
      access_key                  = "test"
      secret_key                  = "test"
      region                      = "us-east-1"
      s3_use_path_style           = false
      skip_credentials_validation = true
      skip_metadata_api_check     = true
      skip_requesting_account_id  = true
    
      endpoints {
        s3 = "http://s3.localhost.localstack.cloud:4566"
      }
    }
    ```
    

※ 참고로 [LocalStack App](https://app.localstack.cloud/dashboard) 에서 실행하고 있는 LocalStack 인스턴스를 가시적으로 확인 가능

### Reference

- https://docs.localstack.cloud/tutorials/s3-static-website-terraform/
- https://docs.localstack.cloud/user-guide/integrations/aws-cli/#localstack-aws-cli-awslocal
- https://docs.localstack.cloud/getting-started/auth-token/