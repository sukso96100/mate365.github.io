---
title: Azure App Service와 Azure Pipelines을 이용한 Django 앱 배포 자동화
authors:
- youngbin-han # 저자 프로필 페이지 경로 입력
date: 2021-03-08T11:11:44+09:00
tags:
- Django
- Azure
- App Service
- Azure DevOps
- Azure Pipelines
ShowToc: true # 글 개요 보여줄지 여부
---
최근 사내에서 클라우드 3사 통합 빌링 시스템 개발에 참여하면서, Django 기반 웹앱 개발에도 참여했고 이를 Azure App Service 에 배포하는 것을 구성하기도 했습니다. 이 글에서는, 개발 초기에 앱 배포를 위해 인프라와 배포 파이프라인을 어떻게 구성했는지 살펴보고, 배포 파이프라인을 어떻게 개선해 왔는지에 대한 경험을 공유하고자 합니다.

# App Service 리소스 생성 및 구성
먼저 웹앱을 배포하려면 당연하게도 호스팅에 쓸 인프라 리소스가 필요합니다. 지난 번 ["ASP.NET앱 개발과 Azure 관리형 서비스로 배포하기 - 2. 관리형 서비스로 빠르게 구축하고 배포하기"](/2020/11/15/quick-aspnet-dev-azmanaged-deploy-part2)에서도 사용한 Azure App Service 를 앱 배포에 사용하기로 하였습니다. 이를 위해, 아래 사진과 같이, App Service 생성시 런타임 스택을 Python 3.8 로 지정 후 생성합니다. 
> 참고: [Application Insight 의 경우 .Net 기반 앱만 런타임에서 모니터링 하는것만 지원하므로.](https://docs.microsoft.com/ko-kr/azure/azure-monitor/app/app-insights-overview#get-started) Python 환경에서도 사용하려면 추후 별도 구성이 필요합니다.

![](images/appsvc.png)

웹앱 배포를 하기 전, DB 연결정보, 암호화 키 정보, SendGrid(이메일 발송) 토큰 등을, Django `settings` 모듈에서는 환경변수에서 불러와서 설정하도록 합니다. 그리고 App Service 의 *구성* 화면의 *애플리케이션 설정*에서 해당 환경변수를 설정해 줍니다. 

![](images/appconfig.png)

API 토큰, 암호화 키, DB 접속 암호 같이 평문으로 저장되면 안 되는 값은 Key Vault 에 저장한 후 이를 불러오도록 설정해야 합니다.
API 토큰이나 DB 접속 암호같이 문자열로 이뤄진 것은 *비밀* 화면에서 새로 생성하여 저장합니다.
![](images/keyvault1.png)

App Service 에서 Key Vault 에 접근할 수 있도록, 액세스 정책을 추가해 줍니다. 이를 위해 App Service 리소스의 ID 에서 *시스템 할당 항목* 을 켜서 Azure AD 에 등록되도록 하고, Key Vault 의 액세스 정책에서 찾아 추가합니다. 추가한 항목에는 *가져오기* 와 *나열* 권한을 부여합니다.
![](images/keyvault2.png)
![](images/keyvault3.png)

Key Vault 에 등록된 비밀의 상세 정보를 조회하면, `https://` 로 시작하는 비밀 식별자를 찾을 수 있습니다. 이 값을 이용해서 App Service 에서 비밀을 불러와서 사용할 수 있습니다. App Service 의 *애플리케이션 설정* 에 값 등록시 아래와 같은 형식의 값을 넣고 저장하면 됩니다.

```bash
@Microsoft.KeyVault(SecretUri=<비밀_식별자>)
# 예시
# @Microsoft.KeyVault(SecretUri=https://mysecret.vault.azure.net/secrets/DbPass/1234cff7f231420aa785e17a12345038)
```