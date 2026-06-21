---
title: HTB-DevArea (Linux)
author: suh0rang
date: 2026-06-20 00:00:00 +0900
categories: [WriteUp, HackTheBox]
tags: [Linux, hoverfly, OS Command Injection]
---

### Reconnaissance (정찰)

![](/assets/img/blog/lab/devarea/1.png)

간단하게 포트 스캔한 결과 21, 22, 80, 8080, 8500, 888 포트가 오픈되어있음을 알 수 있다.

![](/assets/img/blog/lab/devarea/images.png)

기본 스크립트를 이용하여 포트 스캔해보면 알 수 있는 결과는 다음과 같다.

- 21/tcp(FTP) : anonymous 접근 허용
- 22/tcp(ssh) : 해당 서버의 OS는 Ubuntu Linux
- 80/tcp(http) : Apache 웹 서버, 접근 시 http://devarea.htb/ 로 리다이렉트
- 8080/tcp(http) : Java 기반의 오픈소스 HTTP 서버
- 8500/tcp(fmtp) : HTTP 프록시 서버
- 8888/tcp(http) : Hoverfly Dashboard

![](/assets/img/blog/lab/devarea/image (1).png)

80 port로 운영 중인 웹 서버 접근 시 devarea.htb 사이트로 리다이렉트 되며, 해당 도메인이 hosts에 등록되지 않아 정상적으로 접근 불가하다.

![](/assets/img/blog/lab/devarea/image (2).png)

![](/assets/img/blog/lab/devarea/image (3).png)

/etc/hosts 파일에 IP와 도메인을 등록하면 정상적으로 접근 가능하다.

기업 채용 서비스 사이트로 추정되며, 접근 가능한 메뉴나 기능이 없다.

![](/assets/img/blog/lab/devarea/image (4).png)

8080 port에 접근했을 때, 도메인 뒤 페이지 경로를 정확하게 입력하지 않아서 404 Error 가 발생한다.

![](/assets/img/blog/lab/devarea/image (5).png)

![](/assets/img/blog/lab/devarea/image (6).png)

8888 port에 접근해본 결과, hoverfly dashboard 페이지를 확인할 수 있다.

hoverfly 서비스는 API 요청을 기록하고 가상 서비스를 생성하는 경량 프록시 도구로, 성능 테스트나 개발 단계에서 외부 API 없이 테스트 환경을 구축할 때 사용한다.

공개된 Default Password 같은 정보는 없어서 5개 정도의 추측 가능한 패스워드로 로그인 시도 했을 때 모두 일치하지 않았다. 

### **Initial Access (초기 침투) - FTP 익명 접근**

![](/assets/img/blog/lab/devarea/image (7).png)

![](/assets/img/blog/lab/devarea/image (8).png)

FTP 서비스에 anonymous 접근이 가능하기 때문에 해당 서버와 관련된 파일 탐색을 진행한다.

/pub 디렉터리에 employee-service.jar 파일이 존재하며, 해당 파일을 나의 Local로 전송한다.

![](/assets/img/blog/lab/devarea/image (9).png)

![](/assets/img/blog/lab/devarea/image (10).png)

jar 파일의 압축을 해제하면 Apache CXF 프레임워크 기반의 웹 서비스 파일 및 디렉터리가 존재하는 것을 알 수 있다. 

![](/assets/img/blog/lab/devarea/image (11).png)

/META-INF 디렉터리의 MANIFEST.MF 파일을 통해 서비스 운영에 대한 정보를 확인했다. 

웹 서비스 시작 시 참조하는 메인 클래스명은 `htb.devarea.ServerStarter` 이다.

![](/assets/img/blog/lab/devarea/image (12).png)

![](/assets/img/blog/lab/devarea/image (13).png)

`htb.devarea.ServerStarter`클래스 파일을 디컴파일한다.

8080 port에서 Apache cxf 프레임워크를 통한 SOAP 통신을 사용하며 /employeeservice 로 임직원 서비스가 구동 중인 것을 알 수 있다.

![](/assets/img/blog/lab/devarea/image (14).png)

![](/assets/img/blog/lab/devarea/image (15).png)

실제 엔드포인트 접근 시 404 에러가 발생하지 않으며, 적절한 요청을 전송하지 않은 데에 대한 에러만 발생한다.

![](/assets/img/blog/lab/devarea/image (16).png)

`htb.devarea.ServerStarter`클래스 파일에서 확인한 WSDL(Web Services Description Language) 파일에 접근하여 통신 구조를 파악한다.

message → 웹 서비스와 클라이언트 간에 교환되는 데이터에 대한 정의

EmployeeServiceService 서비스에 필요한 값은 다음과 같다.

- submitReport : 요청 데이터에 대한 정의
    - arg0 : report 타입의 객체를 전달
        - employeeName : String
        - department : String
        - content : String
        - confidential : Blooean
- submitReportResponse : 응답 데이터에 대한 정의
    - return : String

![](/assets/img/blog/lab/devarea/image (17).png)

매개변수 `confidential`, `content`, `department`, `employeeName` 에 데이터 타입에 맞춰 값을 입력한 뒤 전송했을 때 `Report marked confidential. Thank you, {employeeName}. Department : {department}. Content : {content}` 응답이 돌아온다.

XXE Injection을 시도했을 때 서버에서 XML 파서 설정이 DTD 선언을 허용하지 않도록 설정되어있어서 불가능 할 것으로 판단하였다.

- 헤맸던 거
    
    ![](/assets/img/blog/lab/devarea/image (18).png)
    
    여기서부터 에러는 안 났지만  파일 Include에 실패했다. 에러가 안 나고 Include 부분에서만 공백이 출력되는 것을 통해 전송되는 타입이나 필터링이 존재할 것이라고 추측하였지만 쉽게 해결하지 못했다.
    
    이유는 SOAP의 경우 정해진 데이터 형식으로 요청을 전송해야 되는데, 내가 다른 타입으로 전송해서 서버가 이를 해석하지 못하고 공백으로 처리하여 반환했던 것이다.
    
    Content-Type: Multipart/Related , body 값은 웹 폼 형식으로 전송하면 정상 응답이 돌아올 것이다.
    

### Credential Access (자격 증명 획득)

![](/assets/img/blog/lab/devarea/image (19).png)

SOAP 표준 방식 MTOM을 맞춰서 `Content-Type: multipart/related` 으로 요청을 전송한다. /etc/hosts 파일을 요청한 결과 base64로 인코딩된 파일 내용이 응답에 포함되어 온다.

![](/assets/img/blog/lab/devarea/image (20).png)

여러 경우의 수를 시도한 결과 /etc/systemd/system/hoverfly.service 파일이 존재하여 해당 파일 내에서 관리자 계정을 획득할 수 있었다.

관리자 → admin/O7IJ27MyyXiU

![](/assets/img/blog/lab/devarea/image (21).png)

![](/assets/img/blog/lab/devarea/image (22).png)

위에서 획득한 계정 정보로 hoverfly 서비스에 로그인 시도한 결과 관리자 계정으로 로그인에 성공했다.

### Execution(Unix Shell 실행)

NVD - CVE-2025-54123

hoverfly와 관련된 CVE-2025-54132가 존재하였고, 해당 CVE 내용은 /api/v2/hoverfly/middleware 에서 OSCommand Injection이 가능하다는 것이다.

입력값이 적절한 검사 없이 시스템 명령에 직접 전달되므로 공격자는 hoverfly 서비스(웹)에서  hoverfly 프로세스(시스템)의 권한으로 호스트 서버에서 임의의 명령을 직접 실행할 수 있게 된다.

![](/assets/img/blog/lab/devarea/image (23).png)

![](/assets/img/blog/lab/devarea/image (24).png)

가장 먼저 정상 작동 여부를 확인하기 위하여 애플리케이션에서 API 호출을 시도한 결과, `Connected to 10.129.244.208 (10.129.244.208) port 8888 (#0)` 를 통해 정상 작동하고 있음을 알 수 있다.

![](/assets/img/blog/lab/devarea/image (25).png)

https://github.com/advisories/GHSA-r4h8-hfp2-ggmf

실질적으로 취약한 Endpoint 인 middleware에 `/bin/bash`  shell을 사용하여 시스템 명령어를 입력하면 실행된 명령어에 대한 결과가 출력된다.

![](/assets/img/blog/lab/devarea/image (26).png)

![](/assets/img/blog/lab/devarea/image (27).png)

해당 서비스 내 python3을 실행하였을 때  NameError 가 발생하는 것을 통해 python3이  설치되어있음을 확인하였다.

![](/assets/img/blog/lab/devarea/image (28).png)

![](/assets/img/blog/lab/devarea/image (29).png)

nc를 이용해 내 서버의 4444 port 를 열고 대기하고, python3 기반으로 내 서버의 4444 port에 접속하는 reverse shell 코드를 전송한 결과 정상적으로 연결이 맺어졌다.

![](/assets/img/blog/lab/devarea/image (30).png)

현재 hoverfly를 실행중인 dev_ryan 사용자의 홈 디렉터리로 이동하면 user.txt 파일이 존재하고, 해당 파일 내용을 통해 user 권한의 flag를 획득할 수 있다.

### Privilege Escalation (권한상승)

![](/assets/img/blog/lab/devarea/image (31).png)

`sudo -l` 을 통하여 현재 사용자가 sudo로 실행할 수 있는 명령어 목록을 확인한 결과, `/opt/syswatch/syswatch.sh` 이 존재하는 것을 알 수 있었다.

![](/assets/img/blog/lab/devarea/image (32).png)

`/opt/syswatch/syswatch.sh` 실행에 필요한 옵션을 확인해본다.

해당 값을 통해 직접적으로 권한 상승은 불가능할 것으로 판단된다.

![](/assets/img/blog/lab/devarea/image (33).png)

suid가 설정되어있거나 777로 설정되어있는 파일 중 권한 상승에 사용될 수 있는 파일을 탐색하다가, `/usr/bin/bash` 파일의 권한이 777 으로 설정되어있는 것을 발견하였다.
`/usr/bin/bash`는 root 소유이며 실행 권한만 있어야 하지만, 777인 경우 일반 유저가 이 파일을 권한 상승 기능을 가진 파일로 바꿔치기 할 수 있다.

- 시행착오
    
    ![](/assets/img/blog/lab/devarea/image (34).png)
    
    환경변수 등록에 대한 제재가 없는 것 같아서 환경변수 등록을 통한 권한 상승 시도를 하였으나 실패하였다.
    

![](/assets/img/blog/lab/devarea/image (35).png)

![](/assets/img/blog/lab/devarea/image (36).png)

우선 sticky bit가 설정된 디렉터리인 tmp에 `/usr/bin/bash` 파일의 백업본을 생성해둔다.

![](/assets/img/blog/lab/devarea/image (37).png)

현재 로그인 되어있는 쉘이 `/bin/bash` 이기 때문에 `sh` 로 변경한 뒤 bash를 종료한다.

![](/assets/img/blog/lab/devarea/image (38).png)

이 스크립트가 실행될 때, 미리 복사해둔 원본 bash를 사용하여 실행하라고 쉬뱅으로 지정한 뒤, 해당 파일을 다시 `/tmp/bash` 로 생성한다.

`/tmp/bash` 에는 추가로 suid 권한까지 추가된 4755 를 부여하여 실행 시 root 권한으로 실행될 수 있게 `/usr/bin/bash` 파일의 내용을 덮어쓴다.

![](/assets/img/blog/lab/devarea/image (39).png)

sudo 명령어로 실행할 수 있는 `/opt/syswatch/syswatch.sh` 파일 실행 시 해당 파일에서 `/usr/bin/bash` 를 호출하게 된다. 

이 때 호출된 bash는 `/tmp/bash` 파일이며, suid 가 정상적으로 부여되어있다.

![](/assets/img/blog/lab/devarea/image (40).png)

`/tmp/bash -p` 를 통해 suid 가 부여된 파일에 Privileged 모드로 bash shell 이 실행되게 되어 root 권한으로 shell 이 실행될 수 있다.

root 홈 디렉터리로 이동하면 root.txt 파일이 존재하고, 해당 파일 내용을 통해 root 권한의 flag를 획득할 수 있다.

---

- [권한상승 참고]
    
    https://payatu.com/blog/a-guide-to-linux-privilege-escalation/
    
    https://medium.com/@ibm_ptc_security/linux-based-privilege-escalation-techniques-1e9b83269a75
    
    https://morgan-bin-bash.gitbook.io/linux-privilege-escalation/linux-privilege-escalation