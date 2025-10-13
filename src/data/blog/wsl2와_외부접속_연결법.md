---
  author: jowoosung
  pubDatetime: 2025-10-13T09:04:50.954Z
  modDatetime: 2025-10-13T09:04:51.350Z
  title: wsl2와 외부접속 연결법
  slug: wsl2와-외부접속-연결법
  featured: true
  draft: false
  tags:
    - infra
  description: wsl2의 네트워크적 특성과 외부접속을 위한 구성법 설명
---
## Table of contents

## 개요  
윈도우 환경에서 우분투를 이용하여 서버를 구성하려한다.  
그 중에 가장 간편한 방법은 wsl을 이용하여 구성하는 것으로 알고 있다.  
하지만 wsl2는 기본적으로 NAT 환경에서 동작하기 때문에, 서버를 단순히 띄운다고 해서 호스트 PC나 외부에서 접근할 수 있는것은 아니다.  
그래서 wsl2 네트워크 특성을 이해하고, 포트포워딩과 방화벽 설정을 통해 호스트와 외부에서도 접속 가능하게 만드는 방법을 소개한다.  

## wsl2 네트워크 특성  
wsl2는 가상화 기반으로 리눅스 커널을 돌리기 때문에, windows와는 별도의 가상 네트워크 인터페이스를 사용한다.  
- wsl 내부 IPsms hostname -I 명령으로 확인할 수 있다.  
- 서버를 127.0.0.1로만 바인딩하면 wsl 내부에서만 접근이 가능하다.  
- wsl2 ip는 재시작할 때마다 바귀어서 항상 같은 IP를 쓰는 것은 아니다.  

즉, wsl 안에서만 서버 테스트할거면 문제없지만, 호스트 PC나 외부에서 접속하려면 별도의 포트포워딩 설정이 필요하다.  

## 포트포워딩과 방화벽 설정  
호스트에서 wsl 서버에 접근하려면, windows에서 포트포워딩과 방화벽 설정을 해줘야 한다.  
이 설정을 안하면 호스트 pc는 wsl 내부 ip를 알 수 없기 때문에 외부 요청을 전달할 수 없다.  
### 방화벽 열기  
먼저 TCP 80(HTTP)과 443(HTTPS) 포트를 방화벽에서 허용해야 한다.  
이 작업을 해야 호스트로 들어오는 요청이 방화벽에 의해 막히지 않는다.  
```bash
netsh advfirewall firewall add rule name="Allow HTTP" dir=in action=allow protocol=TCP localport=80
netsh advfirewall firewall add rule name="Allow HTTPS" dir=in action=allow protocol=TCP localport=443
```
### WSL IP 확인  
WSL 내부 서버 IP는 다음 명령어로 확인할 수 있다.  
이 IP는 windows 호스트가 요청을 전달할 대상이 되기 때문에 알아둬야 한다.  

```bash
wsl hostname -I
```
### Windows -> WSL 포트포워딩  
`netsh interface portproxy` 명령어를 사용하면, 호스트 pc로 들어오는 요청을 WSL 내부 서버로 전달할 수 있다.  

```bash
netsh interface portproxy add v4tov4 listenport=80 listenaddress=0.0.0.0 connectport=80 connectaddress=172.31....
netsh interface portproxy add v4tov4 listenport=443 listenaddress=0.0.0.0 connectport=443 connectaddress=172.31....
```
- `listenaddress=0.0.0.0` -> 호스트의 모든 네트워크 인터페이스에서 요청을 받기 위해 설정한다.
- `connectaddress` -> WSL 내부 IP를 지정해서 요청을 WSL 서버로 전달
- `listenport`/`connectport` -> 각각 호스트와 WSL에서 쓸 포트  
설정 확인:
```bash
netsh interface portproxy show all
```
이제 호스트 PC IP 로 접속하면 WSL2서버에 연결할 수 있다.  

## 주의 사항  
1. WSL2를 재시작하면 내부 IP가 바뀌기 때문에, 포트포워딩 설정을 다시 해줘야 함
2. 방화벽 설정이 제대로 되어 있는지 항상 확인애야함
3. 외부에서 접속을 열 경우 traefik 인증이나 방화벽 규칙같은 추가 보안 조치가 필요함  