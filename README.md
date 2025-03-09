### [ 시나리오 ]

| A 회사는 건물 내에서 구성된 **인트라넷**을 통해 사내 업무를 진행하고 있습니다.
하지만 최근 **신사업 진출**로 인해 **출장 업무**가 급격히 증가하면서,
외부에서도 사내망에 안전하게 접속할 수 있는 환경이 필요하게 되었습니다.
이에 따라, **사내 인프라 운영팀**의 B는 팀장으로부터
**출장 직원들이 외부에서도 사내망에 접속할 수 있도록 VPN 서버를 구축하라**는 지시를 받았습니다.
B는 VPN 서버를 구현한 후, **통신이 안전하게 암호화되어 있는지 테스트**를 진행할 예정입니다. |
| --- |
|  |

### [ Architecture ]

<img width="885" alt="Image" src="https://github.com/user-attachments/assets/e1f34201-7b39-4865-9fd2-7e9da79607b6" />
**VPN 서버용 VM 설정**

- 네트워크 설정 제외, 강의 실습용 환경과 동일
1. **네트워크 및 VM 설정**
2. 어댑터 1에 [어댑터에 브리지] 및 노트북 WI-Fi 어댑터 선택(강의실 WIFI IP대역 사용)
    
    어댑터 2에 [내부네트워크] 및 intnet ip는 linux내에서 Manual을 사용하여 지정
    
![Image](https://github.com/user-attachments/assets/29fbbe51-ee74-4ad5-ad2a-a33aedba0009)
    
![Image](https://github.com/user-attachments/assets/a9ee4886-594c-47d6-b1d2-dd7a1676d458)
    
3. 기기별 사용 IP 설정
    
    
    | VPN Server	(VM) 	-external 192.168.0.141
    -internal 192.168.56.103
    Windows		-external 192.168.0.143
    Linux		(VM)	-external 192.168.0.142
    WebServer	(VM)	-internal 192.168.56.104 |
    | --- |

## **실습 가이드 라인**

## OpenVPN 및 easy-rsa 설치 및 인증서 발급 과정

### 1. 필수 패키지 설치

```bash

yum install -y epel* openvpn easy-rsa
```

- **EPEL 저장소** 설치 이유: 추가 패키지 지원
- **easy-rsa 설치 이유**: SSL/TLS 기반 인증서 관리를 용이하게 하기 위함

---

### 2. 키(key), 인증서(crt) 발급 과정

### 1) 작업 및 보관 공간 생성

```bash
mkdir /root/easy-rsa
ln -s /usr/share/easy-rsa/* /root/easy-rsa/
chmod 777 /root/easy-rsa
```

- `/root/easy-rsa` 디렉터리 생성 및 easy-rsa 실행 파일 링크
- 권한 부여하여 접근 가능하도록 설정

```bash

mkdir -p /root/client-configs/keys
mkdir -p /root/client-configs/files
chmod 777 /root/client-configs
```

- 클라이언트 인증서(.crt), 개인 키(.key), 설정 파일(.ovpn) 저장 디렉터리 생성 및 권한 부여

### 2) PKI (Public Key Infrastructure) 작업 시작

```bash
cd /root/easy-rsa/3.0.8
./easyrsa init-pki
```

- `init-pki` 실행: 인증서, 개인 키, 중간 인증 기관(CA) 인증서 및 관련 파일을 저장할 기본 디렉터리 구조 생성

![Image](https://github.com/user-attachments/assets/b19920a1-3b8f-45b2-b46a-54758e9392fd)

- 이 명령어를 실행하는 이유? :인증서, 개인 키, 중간 인증 기관(CA) 인증서, 그리고 기타 관련 파일을 저장할 기본 디렉토리 구조를 생성

3) ca.crt 및 ca.key 생성

> ./easyrsa build-ca nopass

Common Name : server 로 설정

![Image](https://github.com/user-attachments/assets/4011a67f-b3a7-40c6-a636-9e7df1135aeb)

- 이 명령어를 실행하는 이유? :서버에서 사용할수있는 CA의 기능 제공
- ca.crt와 ca.key란? crt는 발행된 인증서의 유효성을 검증하는데 사용하고 key는 인증서를 서명하는데 사용
- nopass 하는 이유? :CA개인키가 암호없이 생성

4) server.crt(서버 인증서) 및 server.key 생성

```coffeescript
방법1:  ./easyrsa gen-req server nopass | gen-sign server server

방법2:  ./easyrsa build-server-full server nopass
```

![Image](https://github.com/user-attachments/assets/bbea8ab5-3d39-4351-b5f2-e015d946573c)

- server.req란? ca로부터 인증서(server.crt) 발급받기위해 사용
- server.key란? 데이터의 암호화 및 디지털 서명에 사용됨
- server.crt란? 서버의 공개 키와 연결되어, 해당 서버가 신뢰할 수 있는지를 클라이언트가 검증

5) dh.pem 생성

```coffeescript
./easyrsa gen-dh
```

![Image](https://github.com/user-attachments/assets/404df6f1-45d2-4bb4-885b-cc6dfa2c11ca)

- Diffie-Hellman 키 생성 및 하는 이유: 안전한 키 교환을 통해 보안 통신 채널을 구축

6) ta.key 생성

```coffeescript
openvpn --genkey --secret ta.key
```

![Image](https://github.com/user-attachments/assets/5bd2fe58-9d25-4205-bdd2-f11243004113)

- ta.key가 필요한 이유? TLS 인증 키로 추가적인 보안 계층을 제공
- SSL/TLS란?: 인터넷 상에서 데이터를 안전하게 전송하기 위해 설계된 표준 보안 기술

7) (Client).key, crt 발급

```coffeescript
방법1: 
./easyrsa gen-req (Client) nopass | gen-sign (Client) client

(Client) = client로 이름 설정

방법2:
./easyrsa build-client-full (Client) nopass

(Client) = client로 이름 설정
```

![Image](https://github.com/user-attachments/assets/230e4430-a565-4d00-8d0c-4172d55e1e3b)

8) 만든 .key .crt를 필요 위치로 copy

```coffeescript
# 1. TLS 인증 키 (ta.key)
cp /root/easy-rsa/3.0.8/ta.key /etc/openvpn/server/  # 서버로 복사
cp /root/easy-rsa/3.0.8/ta.key /root/client-configs/keys/  # 클라이언트 설정 디렉터리로 복사

# 2. 클라이언트 개인 키 (Client.key)
cp /root/easy-rsa/3.0.8/pki/private/(Client).key /root/client-configs/keys/  # 클라이언트 키 복사

# 3. 서버 개인 키 (server.key)
cp /root/easy-rsa/3.0.8/pki/private/server.key /etc/openvpn/server/  # 서버 키 복사

# 4. Diffie-Hellman 파라미터 (dh.pem)
cp /root/easy-rsa/3.0.8/pki/dh.pem /etc/openvpn/server/  # 키 교환을 위한 DH 파라미터 복사

# 5. 인증 기관(CA) 인증서 (ca.crt)
cp /root/easy-rsa/3.0.8/pki/ca.crt /etc/openvpn/server/  # 서버로 복사
cp /root/easy-rsa/3.0.8/pki/ca.crt /root/client-configs/keys/  # 클라이언트 설정 디렉터리로 복사

# 6. 클라이언트 인증서 (Client.crt)
cp /root/easy-rsa/3.0.8/pki/issued/(Client).crt /root/client-configs/keys/  # 클라이언트 인증서 복사

# 7. 서버 인증서 (server.crt)
cp /root/easy-rsa/3.0.8/pki/issued/server.crt /etc/openvpn/server/  # 서버 인증서 복사

```

1. **VPN Server 설정**

1) server 구성 샘플 파일(.conf) 복사

```coffeescript
cp /usr/share/doc/openvpn-2.4.12/sample/sample-config-files/server.conf /etc/openvpn/
```

2) server.conf 내용 수정

```coffeescript
vi server.conf
```

찾기 명령어로 아래 요소들 찾아서 수정

```coffeescript
# OpenVPN 서버 설정 파일 구성 (server.conf)

port 1194                         # OpenVPN 포트 지정
proto tcp                         # TCP 프로토콜 사용 (proto udp는 주석 처리)
dev tun                           # TUN 디바이스 설정

# 인증서 및 키 파일 경로 설정
ca /etc/openvpn/server/ca.crt      # CA 인증서 파일
cert /etc/openvpn/server/server.crt # 서버 인증서 파일
key /etc/openvpn/server/server.key  # 서버 개인 키 파일
dh /etc/openvpn/server/dh.pem      # Diffie-Hellman 파라미터 파일 (dh2048.pem에서 이름 변경)

# 네트워크 설정
server 10.8.0.0 255.255.255.0       # VPN 네트워크 설정 (클라이언트에게 할당할 IP 범위)
push "route 192.168.56.0 255.255.255.0"  # 내부 네트워크 경로 설정 (선택 사항)

# TLS 인증
tls-auth /etc/openvpn/server/ta.key 0  # ta.key 파일 설정 (0은 서버 역할)

# 암호화 설정
cipher AES-256-CBC                    # 암호화 알고리즘 지정

# 클라이언트 관련 설정 (주석 해제 X)
;topology subnet       # 클라이언트 간의 라우팅을 서브넷 수준에서 수행
;explicit-exit-notify 1 # UDP 사용 시 연결 종료 메시지 전달 (UDP 사용 시 주석 해제)
;duplicate-cn          # 동일한 클라이언트 키로 동시 접속 가능하게 설정
;user nobody           # OpenVPN 실행 권한 제한
;group nobody          # OpenVPN 실행 권한 제한
;mute 20               # 같은 로그가 20번 이상 반복될 경우 무시

# 클라이언트 DNS 및 게이트웨이 설정 (주석 해제 X)
;push "dhcp-option DNS 8.8.8.8"   # 클라이언트가 사용할 DNS 주소 설정
;push "redirect-gateway def1 bypass-dhcp"  # 모든 패킷을 VPN 서버를 통해 전달 (필수 아님)

```

2) sysctl.conf 수정

```coffeescript
# 2) sysctl.conf 수정 (IP 포워딩 활성화)

vi /etc/sysctl.conf  # sysctl.conf 파일 편집

# IP 포워딩 활성화 (추가 작성)
net.ipv4.ip_forward=1  

# 설정 적용
sysctl -p  

# net.ipv4.ip_forward=1을 추가해야 하는 이유?
# -> IP 포워딩 또는 라우팅 기능을 수행하기 위해 필요 

```

:IP 포워딩 또는 라우팅 기능을 수행하기위해

3) 설정 적용

```coffeescript
sysctl -p

systemctl restart network
```

sysctl 적용 및 network 재시작

1. 방화벽 및 내부망 접속을 위한 iptables 설정
    
    ```coffeescript
    
    # 1. 방화벽 실행 및 상태 확인
    systemctl start firewalld          # 방화벽 실행
    systemctl status firewalld         # 방화벽 상태 확인
    
    # 2. OpenVPN 포트 및 서비스 활성화
    firewall-cmd --zone=public --add-port=1194/tcp --permanent  # OpenVPN TCP 포트 1194 활성화
    firewall-cmd --zone=public --add-service=openvpn --permanent  # OpenVPN 서비스 활성화
    
    # 3. NAT (IP 마스커레이드) 활성화
    firewall-cmd --add-masquerade --permanent  # 클라이언트가 인터넷 및 내부 네트워크에 접근 가능하도록 설정
    
    # 4. 방화벽 규칙 적용
    firewall-cmd --reload  # 변경 사항 적용
    
    # MASQUERADE란?
    # - 네트워크 내부 기기가 외부 네트워크에 접근할 때,
    #   내부 IP 주소를 방화벽 또는 라우터의 공개 IP 주소로 변환하는 기능 (부록 14번 참고)
    
    ```
    
![Image](https://github.com/user-attachments/assets/053cae32-e08b-401b-a641-414cbbe41b05)
    
2. **Windows Client 설정 파일 생성1) client 구성 샘플 파일 복사**
    
    ```coffeescript
    **cp /usr/share/doc/openvpn-2.4.12/sample/sample-config-files/client.conf /root/client-configs/client.conf**
    ```
    

2) client.conf 파일 수정

```coffeescript
vi /root/client-configs/client.conf
```

```coffeescript
# 4) 클라이언트 설정 파일 (client.conf)

# VPN 터널 디바이스 설정
dev tun

# 프로토콜 설정
proto tcp  # 사용하는 프로토콜 (서버 설정과 동일하게 맞춰야 함)

# OpenVPN 서버의 IP 및 포트 설정
remote 192.168.0.141 1194  # VPN 서버의 실제 IP 주소로 변경

# 실행 권한 관련 (주석 해제 X)
;user nobody
;group nogroup

# 인증서 및 키 파일 경로 설정
ca /root/client-configs/keys/ca.crt         # CA 인증서 파일 경로
cert /root/client-configs/keys/client.crt   # 클라이언트 인증서 파일 경로
key /root/client-configs/keys/client.key    # 클라이언트 개인 키 파일 경로
tls-auth /root/client-configs/keys/ta.key 1 # TLS 인증 키 (1 = 클라이언트)

# ------------------------------------------------------------------------------

# 5) .ovpn 생성 자동화 스크립트 작성 (make_config.sh)

vi /root/client-configs/make_config.sh  # 스크립트 생성

# ------------------------------------------------------------------------------

# make_config.sh 내용
#!/bin/bash

# 첫 번째 인자로 클라이언트 식별자 받기
KEY_DIR=/root/client-configs/keys
OUTPUT_DIR=/root/client-configs/files
BASE_CONFIG=/root/client-configs/client.conf

cat ${BASE_CONFIG} \
  <(echo -e '<ca>') \
  ${KEY_DIR}/ca.crt \
  <(echo -e '</ca>\n<cert>') \
  ${KEY_DIR}/${1}.crt \
  <(echo -e '</cert>\n<key>') \
  ${KEY_DIR}/${1}.key \
  <(echo -e '</key>\n<tls-auth>') \
  ${KEY_DIR}/ta.key \
  <(echo -e '</tls-auth>') \
  > ${OUTPUT_DIR}/${1}.ovpn

# ------------------------------------------------------------------------------

# 6) make_config.sh 실행 권한 부여 및 사용 방법

chmod +x /root/client-configs/make_config.sh  # 실행 권한 부여

# 클라이언트 OVPN 파일 생성 예제
/root/client-configs/make_config.sh client1  # client1.ovpn 생성

# 생성된 .ovpn 파일 확인
ls -l /root/client-configs/files/

```

4) shell script 권한 부여 및 실행

```coffeescript
chmod 777 /root/client-configs/make_config.sh

/root/client-configs/make_config.sh
```

(client).ovpn 파일 생성

![Image](https://github.com/user-attachments/assets/dcb5842b-754e-41e5-8b9c-686211c0a669)

5) Windows Client가 요청 시 위 과정으로 (client).ovpn 생성 후 전달

1. **Linux Client 설정 파일 생성1) Linux Client가 요청 시 clinet.conf에 필요한 파일 전달ca.crt, ta.ket, (Client).crt, (Client).key를 각 위치에서 전달Linux Client는 openvpn 설치 후 client.conf 파일 작성(이하 후술)**
2. **OpenVPN 실행1) OpenVPN Server 실행하기> sudo systemctl start openvpn@server.service**
    
![Image](https://github.com/user-attachments/assets/ec04bdf6-e0e2-48c9-b5dc-86fdb87fc9a5)
    
3. **키 생성과 흐름 정리 (부록 15번)**

**VPN 서버 완성 시 파일 구조**

**For Client**

```coffeescript
# 7) OpenVPN 관련 파일 정리 및 경로 구성

# 클라이언트 관련 파일 (총 4개)
CLIENT_KEYS_DIR=/root/client-configs/keys
CLIENT_FILES_DIR=/root/client-configs/files

# 클라이언트 키 및 인증서
ls -l ${CLIENT_KEYS_DIR}/ca.crt  # CA 인증서
ls -l ${CLIENT_KEYS_DIR}/ta.key  # TLS 인증 키
ls -l ${CLIENT_KEYS_DIR}/(client).crt  # 클라이언트 인증서
ls -l ${CLIENT_KEYS_DIR}/(client).key  # 클라이언트 개인 키

# 클라이언트 OVPN 설정 파일 (Windows 유저용)
ls -l ${CLIENT_FILES_DIR}/(client).ovpn

# 리눅스 클라이언트 설정 파일
ls -l /etc/openvpn/client.conf

# ------------------------------------------------------------------------------

# 서버 관련 파일 (총 5개)
SERVER_DIR=/etc/openvpn/server

# 서버 키 및 인증서
ls -l ${SERVER_DIR}/ca.crt  # CA 인증서
ls -l ${SERVER_DIR}/dh.pem  # Diffie-Hellman 파라미터
ls -l ${SERVER_DIR}/server.crt  # 서버 인증서
ls -l ${SERVER_DIR}/server.key  # 서버 개인 키
ls -l ${SERVER_DIR}/ta.key  # TLS 인증 키

# 서버 설정 파일
ls -l /etc/openvpn/server.conf  # OpenVPN 서버 설정 파일 (위 5개 파일의 경로 입력됨)
```

1. **윈도우에서 접근해보기**
2. **먼저 윈도우에 openvpn을 설치해 줍니다**

설치 사이트: [**https://openvpn.net/community-downloads/**](https://openvpn.net/community-downloads/)

1. **vpn에 접속하기 위해 필요한 인증키 client.ovpn을 가져옵니다.**

**C:\Program Files\OpenVPN\config**

위의 주소로 가상 머신에 있는 **ta.key**와 client**.ovpn**을 가져오면   됩니다.

가져올 때는 scp나 winSCP를 사용하여 가져옵니다.

- c**lient.ovpn 가져오기**

![Image](https://github.com/user-attachments/assets/73e1a5ad-0b01-4797-9a3c-c5209c184828)

- **가져온 결과**

![Image](https://github.com/user-attachments/assets/76180113-1568-407a-bf62-fddec87a9990)

1. **그 후 실행을 한 뒤 연결하면 접속된 것을 확인할 수 있습니다.**

![Image](https://github.com/user-attachments/assets/4f99484c-24d3-4216-aa73-d2b7b4732ad0)

- **VPN을 껐을 때**

![Image](https://github.com/user-attachments/assets/83540458-ac53-4065-97ca-f7cffe4b020a)

- **VPN을 켰을 때**

![Image](https://github.com/user-attachments/assets/a5026f46-0ade-4728-9d71-3bafc4a97a48)

**Linux 방면 접근**

1. **먼저 윈도우에 openvpn을 설치해 줍니다**

자신의 client의 ip먼저 확인. 위의 경우 192.168.0.142입니다.

client서버에 epel*,easy-rsa,openvpn설치

```coffeescript
[root@Personal_Computer ~]# yum -y install epel*

[root@Personal_Computer ~]# yum -y install easy-rsa  #easy-rsa설치

[root@Personal_Computer ~]# yum -y install openvpn   #Openvpn설치
```

리눅스 클라이언트 설정

vpnserver로부터 (Client).crt,ca.crt ,(Client).key,ta.key를 client로 전달

[root@Personal_Computer ~]# mkdir -p /root/client-configs/keys/ # key를 저장할 디렉토리 생성

| client에 넣어야하는것들 | Openvpnserver에서의 위치 |
| --- | --- |
| ca.crt | (/root/client-configs/keys/ca.crt) |
| ta.key | (/root/client-configs/keys/ta.key) |
| (client).crt | (/root/client-configs/keys/(Client).crt) |
| (client).key | (/root/client-configs/keys/(Client).key) |

위에 (Client)는 앞에서 ./easyrsa build-client-full (Client) nopass로 부터 만들어짐

OpenVpnserver에서 client의 /root/client-configs/keys/로

위의 4개의 값들을 보내주기(방화벽 ssh:22번 확인필요)

```coffeescript
[OpenVpn_Server ~]# scp /root/client-configs/keys/ca.crt root@192.168.0.142:/root/client-configs/keys/  #root앞에 띄어쓰기 필요

#client ip확인후 자신의 환경에 맞게 root@192.168.0.142 처럼 각자 설정
#OPENVPN Port 방화벽 설정 [root@Personal_Computer ~]에서 실행

firewall-cmd --zone=public --add-port=1194/tcp --permanent
firewall-cmd --zone=public --add-service openvpn --permanent
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
```

---

```coffeescript
# 8) 클라이언트 설정 파일 복사 및 편집

# 1. 기본 client.conf 파일을 /etc/openvpn/으로 복사
cp /usr/share/doc/openvpn-2.4.12/sample/sample-config-files/client.conf /etc/openvpn/

# 2. client.conf 편집 (VPN 사용 시 필요한 설정 적용)
vi /etc/openvpn/client.conf

# ------------------------------------------------------------------------------

# 9) /etc/openvpn/client.conf 설정 내용

client                               # 클라이언트 모드 설정
dev tun                              # TUN 디바이스 사용 (VPN 터널)
proto tcp                            # 사용하는 프로토콜 (서버와 동일해야 함)
remote 192.168.0.141 1194            # VPN 서버 IP 및 포트 설정

# 실행 권한 관련 (주석 해제 X)
;user nobody
;group nogroup

# 인증서 및 키 파일 경로 설정
ca /root/client-configs/keys/ca.crt         # CA 인증서 파일
cert /root/client-configs/keys/client.crt   # 클라이언트 인증서 파일
key /root/client-configs/keys/client.key    # 클라이언트 개인 키 파일
tls-auth /root/client-configs/keys/ta.key 1 # TLS 인증 키 (1 = 클라이언트)

```

---

```coffeescript
[root@Personal_Computer~]#cd /etc/openvpn

[root@Personal_Computer openvpn]#openvpn --config client.conf  #로 vpn 실행후 다른 터미널 창을 사용

root@Personal_Computer:/etc/openvpn
```

![Image](https://github.com/user-attachments/assets/22f50ee7-7365-48aa-8e74-f1f11500f9a3)
위와 같이 Initialization Sequence Completed가 나오면 clinet에서 openvpn 설정 완료 (쉘 종료시 연결 끊김)

**Office_web_server**

VPN 서버용 VM 설정

- 네트워크 설정 제외, 강의 실습용 환경과 동일

어댑터 2에 [내부네트워크] 및 intnet ip는 linux내에서 Manual을 사용하여 지정

![Image](https://github.com/user-attachments/assets/7e2c9710-9b24-44f2-a9ff-118f12a76e71)

OpenVpnserver의 내부네트워크설정과 같은 Gateway로 설정

![Image](https://github.com/user-attachments/assets/5cb61157-ac53-46a0-8020-6b29a67fc6a0)

Route 설정으로 양방향 통신 가능
