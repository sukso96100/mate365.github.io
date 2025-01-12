---
feature_image: images/2020-12-03_12-39-21.png
authors: 
- seongjin kim
date: "2020-12-11T00:00:00Z"
categories:
- Hands on

tags:
- AWS
- AWS Re:Invent
- AWS Jams
- Fortinet
- FortiGate
- Transit Gateway

title: "AWS Re:Invent 2020 JAM으로 알아보는 NGFW로 네트워크 보안 강화하기"
---

1. 개요  
    올해 AWS 리인벤트는 COVID-19으로 인해 결국 온라인으로 진행되고있어요.  
    대개 이런 큰 이벤트에서 저는 주로 Hands On이나 직접 참여하는 활동을 즐기는데요.  
    이번에 들어가 보니 Re-Invent 페이지에서 Hands On Content 메뉴가 떡하니 보이네요!  
    반가운 마음에 바로 접속해봤어요.  
    Jam Lounge에서는 AWS 인프라에서 발생한 문제를 해결하는 재미있는 이벤트가 있어요!  

    ![](images/2020-12-11_14-53-46.png)  

    등록을 하고 접속하면 아래와 같이 다양한 챌린지들이 보입니다.  

    ![](images/2020-12-11_14-55-55.png)  

    이번 시간에는 영국에서 발생한 Acme 사의 네트워크 보안 관련 문제를 해결하러 가봅시다.  

    ![](images/2020-12-11_14-57-32.png)  

    이런... Acme는 신입사원이 첫 입사 날부터 보안 문제를 해결해야 하는 무서운 회사네요.  
    (입사 첫날 저를 이렇게 안 굴려주신 팀장님 감사합니다......)  

1. 작업 1 : 라우팅 문제 해결
    해야 할 작업은 총 5개인데요.  
    아래 그림에서 우측 위를 보시면 'AWS 콘솔 열기'로 콘솔로 가서 CloudFormation을 보면 구성된 정보를 볼 수 있어요.  
    
    ![](images/2020-12-03_12-39-21.png)  

    웹서버 정보나 URL, 패스워드 정보가 있네요.  
    
    ![](images/2020-12-03_12-39-48.png)  

    일단 라우팅 문제를 해결하라고 하니 ping이 가는지부터 확인합니다.  
    JumpBox에서 server1, 2로 핑을 보내보면 server2로 핑이 가지 않습니다.  
    (완료하고 나서 캡처하려니 콘솔접근이 안 되네요. 이미지가 없어 죄송합니다.)  
    
    그럼 방화벽 구성을 한번 보겠습니다.  

    ![](images/2020-12-03_23-35-44.png)  
    
    포티넷이군요! 포티넷 방화벽을 무료로 사용해볼 기회라니!  
    위에서 확인한 URL로 ID, PW로 접속합니다.  

    ![](images/2020-12-03_23-50-29.png)  
    
    일단 급한 마음에 정적 라우팅을 확인해봤는데 무슨 문제인지 잘 모르겠네요.  
    그럼 문제에서 나온 '재고'에서 말하는 TGW 구성이 잘 되어있는지 확인해봐야겠네요.  
    ![](images/2020-12-11_15-44-43.png)  
    VPC는 3개이고 TGW는 한 개, Attach 된 VPC는 3개군요.  
    ![](images/2020-12-11_15-46-41.png)  
    ![](images/2020-12-11_15-46-51.png)  

    TGW의 Routing Table 설정을 살펴보니 SpokeRouteTable에 Spoke2 VPC가 Associate되지 않았네요. 바로 수정해서 추가합니다.  

    ![](images/2020-12-03_23-51-10.png)  
    
    한가지더!  
    아까 VPC들을 보니 처음에 확인했던 라우팅 테이블에서 10.2.0.0/16 대역이 없었던 것도 추가합니다.  

    ![](images/2020-12-05_0-05-22.png)  

    이제 핑이 잘 갑니다. (이것도 캡처가 없네요.ㅠㅠ)  
    지정된 S3 버킷에서 정답 Flag가 담진 파일을 열어서 입력합니다!  

    ![](images/2020-12-04_0-25-33.png)  
    ![](images/2020-12-04_0-25-57.png)  
    
    이번 작업 1에서는 TGW 구성에서 Routing Associate를 수정하여 VPC간 연결 문제를 해결했습니다.  
    네트워크 구성 문제를 해결했으니 다음 문제는 뭘까요?  
    궁금하네요.  

1. 작업 2 : 공격 차단
    
    ![](images/2020-12-04_0-00-09.png)  
    Fortigate를 사용해서 공격을 차단하라고 하네요.  
    Websrv1에 열린 URL로 접속하면 DVWA가 나옵니다.  

    ![](images/2020-12-04_0-19-38.png)  

    DVWA(Damn Vulnerable Web App)  는 웹 취약점 실습을 목적으로 웹 환경이 취약하게 구성되어 있습니다. 처음 웹 취약점 공부할때가 생각나네요.  
    (~~사실 학생시절 BeeBox로 공부했다는건 함정~~)  
       
    ![](images/2020-12-04_0-19-22.png)  
    
    취약성 테스트로는 Command Injection 구문을 줍니다. 글은 기계번역이 된것인지 고양이라고 나오네요 ㅎㅎ. 명령어를 입력했더니 Command Injection이 잘 수행되어 중요 정보인 passwd를 긁어오는 것을 확인할 수 있습니다.  
    * Command Injection 공격이란?  
        Web Application에 OS 명령어를 요청으로 보낼 경우 서버에서 해당 명령어가 동작하는 취약점을 말합니다.  
        아래 명령어의 경우 cat 명령어가 동작하여 passwd의 계정 정보를 출력합니다.  
    
        ```bash
        8.8.8.8 | cat /etc/passwd
        ```

    NGFW가 있는데도 불구하고 공격 구문이 수행되는 상황이 확인됩니다. 이번 작업은 AttackID를 확인하는 것 까지네요.  
    Log & Report에서 Intrusion Prevention 로그를 보면 공격이 왔다는 것을 확인 할 수 있습니다.  
    
    ![](images/2020-12-04_0-24-47.png)  

    위 Attack ID를 Answer에 제출합니다.  

1. 작업 3 : 멀웨어 다운로드 차단  

    ![](images/2020-12-04_0-00-20.png)  
    
    이번에는 악성코드를 아웃바운드로 다운로드 받는 것을 차단하라고 하네요.  
    
    ![](images/2020-12-04_14-08-10.png)  
    
    아래와 '시작하기'에서 알려준 공격 구문을 Injection하면 eicar 테스트 파일이 다운로드 됩니다.  
    * EICAR 테스트 파일은 Anti-Virus SW를 테스트하기 위해 제작된 파일 악성파일은 아니지만 기본적으로 백신 프로그램에서 악성으로 판단합니다.  

        ```bash
        4.4.2.2 | curl -k https://secure.eicar.org/eicarcom2.zip
        ```
    
    PK로 시작하는 파일 내용이 보이는 것으로 zip파일이라는 것을 알 수 있습니다.  
    (zip파일은 매직넘버가 ASCII로는 PK입니다.)  

    ![](images/2020-12-04_14-09-35.png)  

    그럼 차단을 수행해봅시다.  
    서버에서는 Outbound로 다운로드를 받는 것이니 Policy를 Outbound 정책으로 생성합니다.  
    (보니 원래 있던 걸 수정해도 되더군요.)  
    기존 정책에서 차단이 안되었던 이유가 아웃바운드 트래픽이 https Connection이어서 그랬던 것이었네요.  
    FortiGate에서 Policy & Objects > IPv4 Policy로 들어갑니다.  
    아래와 같이 Security Policy애서 SSL Inspection을 ssl-mitm-https로 설정해서 Antivirus랑 같이 켜서 Policy를 수정 또는 생성합니다.  

    ![](images/2020-12-04_14-28-29.png)  

    ![](images/2020-12-04_14-38-19.png)  

    다시 이전의 Exploit을 수행해봅니다.

    ![](images/2020-12-04_14-29-37.png)  

    Log & Report의 AntiVirus에서 악성 파일이 잡히는 것을 확인할 수 있습니다.  
    
    ![](images/2020-12-04_14-38-58.png)  

    해당 로그를 클릭하여 상세 정보를 확인해 VirusID를 확보하면 작업이 완료됩니다.  

1. 작업 4 : 확인되지 않은 동적 주소 객체 수정  

    ![](images/2020-12-04_0-00-37.png)  
    
    이번에는 뭔가 주소를 수정하라는 것 같군요.  
    일단 DevOps 팀에서 새 방화벽으로 통신이 안된다고하고 SDN 주소 개체를 수정하라고 하니 FortiGate에서 뭐가 안되는지 한번 봅시다.  

    ![](images/2020-12-04_14-44-11.png)  

    Addesses를 가보니 jumpbox1에 뭔가 좋지않은 느낌표가 붙어있네요.  
    마우스 커서를 위에 놓으면 해당 SDN주소와 연결된 IP가 표시됩니다.  
    이를 이용해서 IP를 확인해보겠습니다.  

    ![](images/2020-12-04_14-50-25.png)  

    ![](images/2020-12-04_14-50-41.png)  

    이런...jumpbox1에 대해 연결된 동적 주소가 없다고 나오네요.  
    jumpbox1에 대한 SDN Connector 설정을 확인해봅시다.  
    
    ![](images/2020-12-04_16-09-56.png)  

    SDN Connector의 Tag에는 AppId가 있는데 실제 Jumpbox1의 EC2는 AppId Tag가 없네요.  

    ![](images/2020-12-11_19-20-48.png)  

    그럼 SDN Connector에 적힌 설정대로 Tag를 달아줍시다.

    ![](images/2020-12-04_16-08-49.png)  

    태그를 수정하고 나서 잠시 후 Addesses를 다시 확인하면 동적 주소가 연결되는 걸 확인할 수 있어요.  

    ![](images/2020-12-04_16-09-37.png)  

    ![](images/2020-12-04_16-14-26.png)  

    SDN Connector를 사용해서 주소를 동적으로 얻어낼 수 있네요. 정말 유용한 기능입니다. 

1. 작업 5 : HTTP 액세스 문제 해결

    ![](images/2020-12-04_0-00-47.png)  

    이번에는 웹페이지 접속이 안된다고 하네요.  
    이 회사는 무슨 회사이길래 신입사원에서 이렇게 많은 것을 시킬까요...무섭네요.  
    (~~다 해내는 신입사원도 괴...ㅁㅜㄹ..~~)   
    일단 어떤 웹 서비스가 접속이 안되는지 접속을 시도해 봅시다.  

    ![](images/2020-12-04_16-18-20.png)  
    ![](images/2020-12-04_16-18-44.png)  
    ![](images/2020-12-04_16-19-12.png)  

    app2가 접속이 안되네요.

    ![](images/2020-12-04_16-17-43.png)  

    내부 정책을 확인해보니 app2는 vip_app2_8002 포트로 보내는 걸로 보입니다.
    접속하는 도메인은 서로 같고 URI에서 path만 다른걸 보니 ALB 규칙에서 뭔가 잘못된게 있을 수도 있겠네요.
    
    ![](images/2020-12-04_23-04-26.png)  
    ![](images/2020-12-04_23-04-38.png)  

    역시 ALB 규칙에서 Path를 이용한 조건 경로가 설정이 안되있어서 설정했습니다.  
    다시 접속해보도록 합시다.  

    ![](images/2020-12-04_23-05-05.png)  

    이번에는 504 Timeout 에러가 발생하네요.
    504 에러는 프록시와의 통신이 제한 시간을 초과했다는 것으로 해석할 수 있는데 이는 결국 '요청을 전송했음에도 응답을 받지 못하였다.' 라고 해석할 수 있을 거 같습니다.
    다시 돌아가서 app1과의 설정을 비교해보도록 하면 될거 같네요.

    ![](images/2020-12-04_23-05-21.png)  

    다시 app2에 대한 정책을 보니 NAT가 꺼져있었네요.

    ![](images/2020-12-04_16-17-43.png)  

    app2로 들어온 트래픽이 나갈때 내부 IP를 FortiGate IP로 변경해줘야 다음 응답을 받을 수 있겠지요. 이를 위해 SNAT를 설정합니다.  
    이전에 확인했던 IPv4 Policy에서 app2의 Policy를 수정합니다. NAT를 on하고 “Use outgoing interface address”를 사용합니다.  

    ![](images/2020-12-04_23-05-34.png)  
    
    이제 app2의 URL로 접속해봅시다!  

    ![](images/2020-12-05_0-05-36.png)  
    
    잘 되는군요!  

    이상 acme라는 엄청난 회사의 신입사원 도전기였습니다.  
    많은 것을 배우고 처리한 하루였네요.  
    긴글 읽어주셔서 감사합니다.  

1. 참고문헌 :  
    - AWS JAM 힌트
    - https://docs.aws.amazon.com/directconnect/latest/UserGuide/direct-connect-transit-gateways.html
    - https://docs.fortinet.com/document/fortigate/6.0.0/security-fabric-connector-integration-with-aws/151980/configuring-aws-sdn-connector-using-the-gui
    - https://docs.fortinet.com/document/fortigate/6.2.0/cookbook/898655/static-snat
