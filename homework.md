# 과제  
 
매 주 마다 과제는 아래와 같다.

1. 1주차
    -  Nexus private registry를 사용하여 Jenkins Pipeline 구성
    -  컨테이너 Mysql DB 활용
    -  Portainer 설치
    -  Harbor 구성해 보기 
2. 2주차 : CKA 문제 풀어 보기 [ 문제 ](./cka.md) 
3. 3주차 : kubernetes에 Jenkins Master/Slave 구성하기

<br/>

##  1주차

<br/>

### Nexus private registry를 사용하여 Jenkins Pipeline 구성

<br/>

> 문제 : nexus로 구성된 private docker registry를 사용하여 pipeline을 신규로 구성한다.  

<br/>

순서
- credential을 private docker registry 용으로 신규로 구성 한다.  
- New Item으로 신규 pipeline을 구성한다. ( copy from 이름 활용 )
- Jenkins 화일을 신규 생성하고 docker registry 관련 값을 수정한다.
- pipeline 을 실행시키고 본인의 nexus에 도커 이미지가 저장되었는지 확인한다.    
- 참고 :  Jenkins_private 화일  

<br/>

> 답안

<br/>

- jenkins에서 본인 nexus의 docker ci를 신규 생성한다.
- jenkinsfile을 새로 만들고 수정한다.
  - dockerRepo 변경
  - dockerCredentials 은 신규 생성한 docker ci
- docker.withRegistry 의 공백에 본인이 nexus 주소를 입력한다.
- jenkins pipeline 을 copy from 으로 신규 생성하고 Jenkinsfile 이름을 변경한다.

<br/>

```bash
properties([pipelineTriggers([githubPush()])])

pipeline {
    environment {
        // Global 변수 선언
        dockerRepo = "211.252.85.148:40010/edu1"
        dockerCredentials = 'edu_private_docker_ci'
        dockerImageVersioned = ""
        dockerImageLatest = ""
    }

    agent any

    stages {
        /* checkout repo */
        stage('Checkout SCM') {
            steps{
                script{
                    checkout scm
                 }
            }   
        }
        
        stage("Building docker image"){
            steps{
                script{
                    dockerImageVersioned = docker.build dockerRepo //+ ":$BUILD_NUMBER"
                    dockerImageLatest = docker.build dockerRepo + ":latest"
                }
            }
        }
        stage("Pushing image to registry"){
            steps{
                script{
                    // if you want to use custom registry, use the first argument, which is blank in this case
                    docker.withRegistry( '211.252.85.148:40010', dockerCredentials){
                        dockerImageVersioned.push()
                        dockerImageLatest.push()
                    }
                }
            }
        }
        stage('Cleaning up') {
            steps {
                sh "docker rmi $dockerRepo"//:$BUILD_NUMBER"
            }
        }
    }

    /* Cleanup workspace */
    post {
       always {
           deleteDir()
       }
   }
}
```


<br/>


### 컨테이너 Mysql DB 활용

<br/>

docker compose로 구성한 mysql container에 접속하여 로그인 한 후 wordpress db에 customer 테이블을 생성해 본다.  
 - docker-compose.yml 화일 참고
 - table 이름은 customer 이고 필드는 customer_id , customer_name 만 필요  

<br/>

> 답안

<br/>

컨테이너안에서 명령어로 mysql에 접속한다.  
id/패스워드는 docker-compose.yml에서 확인 가능  

<br/>

```bash
mysql -u wordpress -p
Enter password:
```  

<br/>

databases 목록을 확인하고 원하는 db를 선택하고   
source 명령어를 사용하여  create table 문을 실행한다.   

<br/>

```bash
show databases;
use wordpress;
create table customer (customer_id, customer_name);
```   

<br/>

테이블 목록을 확인하면 customer 테이블이 생성된 걸 확인 할수 있다.  

<br/>

```bash
show tables;
select * from customer;
```  

<br/>


### Portainer 설치

<br/>

> 문제

<br/>

docker 컨테이너 GUI 관리 툴인 portainer를 설치하고 웹에서 접속하여
          모니터링한다.
  - url  참고 :  https://docs.portainer.io/v/ce-2.11/start/install/server/docker/linux
  - 웹 포트는 40005로 expose 한다 ( https 9443 포트 변경 필요 ).
  - 웹브라우저 접속은 https://(본인VM Public IP):40005  
     admin 비밀번호 신규로 생성 (8자리 이상) 한다.


<br/>

> 답안

<br/>

데이터를 저장하기 위해 도커 볼륨을 생성한다. 향후에 컨테이너의 폴더와 연결한다.  

```bash
docker volume create portainer_data
```  

<br/>

로컬에 도커 볼륨이 생성되어 있는지 확인한다.  

```bash
docker volume ls
```  

<br/>

https 포트인 9443만 40005로 변경한다.  

<br/>

```bash
docker run -d -p 8000:8000 -p 40005:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:2.11.1
```  

<br/>

<img src="./assets/portainer_docker_volume.png" style="width: 60%; height: auto;">  

<br/>


<img src="./assets/portainer_admin.png" style="width: 60%; height: auto;">  

<br/>

Getting Start를 선택하고 로컬 도커를 클릭한다.  
     
<img src="./assets/portainer_local.png" style="width: 60%; height: auto;">

<br/>

도커에서 관리하는 리소스 대쉬보드를 볼수 있다.   
     
<img src="./assets/portainer_dashboard.png" style="width: 60%; height: auto;">  

<br/>

컨테이너 항목을 선택하면 자세한 컨테이너 현황을 볼수 있다.   
     
<img src="./assets/portainer_container.png" style="width: 60%; height: auto;">  

<br/>

도커 볼륨은 아래 명령어로 삭제 할 수 있다.   
 
```bash
docker volume prune
```

<br/>

###  Harbor 구성해 보기


<br/>

> 문제   

Harbor 설치 ( https가 힘든 사람은 http 로 구성 )
- Private Docker Registry 를 Harbor를 사용하여 구성 한다.
- Harbor 포트는 40002를 사용하며 https 연결하기 위한 인증서 설정을 한다.
- Harbor 에 edu 프로젝트를 생성하고 신규 계정을 생성하여 members에 추가한다.
- nginx 이미지를 본인의 Private Docker Registry에 Push 한다.
- Harbor에서 확인한다.
- clair 를 사용하여 도커 취약점을 분석한다.  

<br/>

샘플 : https://211.252.85.148:40002/  
계정 : admin/New1234!

<br/>

> 답안  

<br/>

#### 인증기관 인증서 생성

<br/>

참고:
- 출처 1 : https://kyeongseo.tistory.com/entry/harbor-%EC%84%A4%EC%B9%98
- 출처 2 : https://velog.io/@denver_almighty/Docker-Harbor-%EC%84%A4%EC%B9%98  

<br/>


먼저 certs 폴더를 생성합니다.  

<br/>

```bash
mkdir -p ~/certs
cd ~/certs
```
<br/>

CA 인증서를 생성한다. (private key를 생성)

<br/>

```bash
root@newedu:~/certs# openssl genrsa -out ca.key 4096
Generating RSA private key, 4096 bit long modulus
.....++
..................................++
e is 65537 (0x10001)
```  

<br/>

CA 인증서 private key에서 public key를 추출한다.  
private key 파일에는 private key와 public key가 모두 포함되어 있다.  

<br/>

```bash
Can't load /root/.rnd into RNG
140457750159808:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/root/.rnd
Generating a RSA private key
...............................................................................................................................................................................................................................................................................................................................++++
...++++
writing new private key to 'ca.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:KR
State or Province Name (full name) [Some-State]:KyunggiDo
Locality Name (eg, city) []:SeongNam
Organization Name (eg, company) [Internet Widgits Pty Ltd]:KT
Organizational Unit Name (eg, section) []:edu
Common Name (e.g. server FQDN or YOUR name) []:211.252.85.148
Email Address []:shclub@gmail.com
```   


<br/>

#### 서버 인증서 생성

<br/>

개인키를 생성합니다.

<br/>

```bash
root@newedu:~/certs# openssl genrsa -out harbor.key 4096
Generating RSA private key, 4096 bit long modulus (2 primes)
............................++++
....................................................................................................++++
e is 65537 (0x010001)
```

<br/>

서버 인증서 private key에서 public key를 추출한다.  

인증서 서명 요청(CSR)을 생성

<br/>

```bash
root@newedu:~/certs# openssl req -sha512 -new  -key harbor.key  -out harbor.csr
Can't load /root/.rnd into RNG
140445988479424:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/root/.rnd
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:KR
State or Province Name (full name) [Some-State]:KyunggiDo
Locality Name (eg, city) []:SeongNam
Organization Name (eg, company) [Internet Widgits Pty Ltd]:KT
Organizational Unit Name (eg, section) []:edu
Common Name (e.g. server FQDN or YOUR name) []:211.252.85.148
Email Address []:shclub@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:9302390
An optional company name []:KT
```

<br/>

x509 v3 extension file을 생성한다.

<br/>

```bash
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=211.252.85.148
DNS.2=harbor
EOF
```

<br/>

v3.ext 파일을 사용해 harbor 호스트에 대한 인증서를 생성한다.  

<br/>

```bash
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.csr \
    -out harbor.crt
```  

<br/>

아래와 같이 결과가 나오면 된다.  

<br/>

```bash
Signature ok
subject=C = KR, ST = KyunggiDo, L = SeongNam, O = KT, OU = edu, CN = 211.252.85.148, emailAddress = shclub@gmail.com
Getting CA Private Key
```  

<br/>

아래와 같이 화일이 생성이 된 것을 확인 할 수 있다. 

<br/>

```bash
root@newedu:~/certs# ll
total 36
drwxr-xr-x  2 root root 4096 Mar  7 00:32 ./
drwx------ 11 root root 4096 Mar  7 00:29 ../
-rw-r--r--  1 root root 2163 Mar  7 00:23 ca.crt
-rw-------  1 root root 3272 Mar  7 00:22 ca.key
-rw-r--r--  1 root root   41 Mar  7 00:32 ca.srl
-rw-r--r--  1 root root 2224 Mar  7 00:32 harbor.crt
-rw-r--r--  1 root root 1821 Mar  7 00:27 harbor.csr
-rw-------  1 root root 3243 Mar  7 00:26 harbor.key
-rw-r--r--  1 root root  259 Mar  7 00:31 v3.ext
```  

<br/>


#### Harbor 및 docker에 인증서를 제공

<br/>

서버 인증서와 키를 Harbor 호스트의 certficates 폴더에 복사한다.

<br/>

```bash
mkdir -p /data/cert
cp harbor.crt /data/cert/
cp harbor.key /data/cert/
```

<br/>

docker에서 사용하기 위해 crt 파일을 cert 파일로 변환한다.  
docker는 .crt 파일을 CA 인증서로 해석하고, .cert 파일을 클라이언트 인증서로 해석  

<br/>

```bash
# /root/certs 에서 진행한다.
openssl x509 -inform PEM -in harbor.crt -out harbor.cert
```

<br/>

적절한 폴더를 만든 후 서버 인증서, 키 및 CA 파일을 harbor 호스트의 Docker 인증서 폴더에 복사한다.  

만약 nginx 기본 포트 443을 이미 다른 서비스에서 사용하고 있는 경우 /etc/docker/certs.d/yourdomain.com:port를 생성하거나 /etc/docker/certs.d/harbor_IP:port 생성

<br/>

```bash
mkdir -p  /etc/docker/certs.d/211.252.85.148:40002  
cp harbor.cert /etc/docker/certs.d/211.252.85.148:40002/  
cp harbor.key /etc/docker/certs.d/211.252.85.148:40002/  
cp ca.crt /etc/docker/certs.d/211.252.85.148:40002/  
```

<br/>

#### Insecure Registry 설정

<br/>

우리가 구축한 docker registry는 http로 설정이 되어 있지만 docker client에서는  https로 연결을 시도한다.  

정상적으로 remote 에서 연결 하기 위해서는 insecure registry를 설정해야 한다.

<br/>

linux인 경우 `/etc/docker/daemon.json` 화일을 vi 에디터로 열고 아래와 같이 추가한다.    
docker registry의 ip와 포트를 입력한다.  

<br/>

```bash  
root@newedu:~# vi /etc/docker/daemon.json
{
  "insecure-registries": [
    "211.252.85.148:40002"
  ]
}
``` 

이제 도커를 재시작 합니다.

<br/>  

```bash  
systemctl restart docker
```  

<br/>


#### Harbor 설치 와 설정을 한다.

<br/>

harbor 설치 파일을 다운로드한다.  먼저 root 폴더로 이동한다.  

<br/>

```bash 
cd ~/ 
wget "https://github.com/goharbor/harbor/releases/download/v2.1.3/harbor-offline-installer-v2.1.3.tgz"
tar xfvz harbor-offline-installer-v2.1.3.tgz
cd ~/harbor
cp harbor.yml.tmpl harbor.yml
 ```  

<br/>

harbor.yml 파일을 수정

<br/>

```bash  
hostname: 211.252.85.148

https:
  # https port for harbor, default is 443
  port: 40002 
  # The path of cert and key files for nginx
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key
  ...
harbor_admin_password: New1234!

# Harbor DB configuration
database:
  # The password for the root user of Harbor DB. Change this before any production use.
  password: New1234!
...    
```  

<br/>

Harbor 설치 스크립트 실행

<br/>

```bash
root@newedu:~/harbor# ./prepare
prepare base dir is set to /root/harbor
Unable to find image 'goharbor/prepare:v2.1.3' locally
v2.1.3: Pulling from goharbor/prepare
522a057091dd: Pull complete
afa60fdd2728: Pull complete
bb4d1e42614d: Pull complete
738d7c5d7e5b: Pull complete
0f80235190d5: Pull complete
d807fa4eab15: Pull complete
302f62788a49: Pull complete
472c667560f3: Pull complete
Digest: sha256:0877d0b90addbe605d2dc1b870510c820b428d7fc1ae980d6ce899c4970e0b25
Status: Downloaded newer image for goharbor/prepare:v2.1.3
Generated configuration file: /config/portal/nginx.conf
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /data/secret/keys/secretkey
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir
```

<br/>

설치를 아래 옵션과 같이 진행힌다.   
- clair : 도커 이미지 취약점 분석도구
- chartmuseum : helm chart repository

<br/>

install 을 하면 에러가 발생한다.   
아래에서 에러 조치를 한후 다시 실행한다.  

<br/>


```bash
root@newedu:~/harbor# ./install.sh --with-clair --with-chartmuseum

[Step 0]: checking if docker is installed ...

Note: docker version: 23.0.1

[Step 1]: checking docker-compose is installed ...

Note: docker-compose version: 1.21.2

[Step 2]: loading Harbor images ...
72021dc640d8: Loading layer [==================================================>]  34.51MB/34.51MB
ac18f6a923ed: Loading layer [==================================================>]  6.237MB/6.237MB
2139b6d15c37: Loading layer [==================================================>]  13.83MB/13.83MB
Loaded image: goharbor/clair-adapter-photon:v2.1.3


[Step 3]: preparing environment ...

[Step 4]: preparing harbor configs ...
prepare base dir is set to /root/harbor
Clearing the configuration file: /config/registryctl/env
Clearing the configuration file: /config/registryctl/config.yml
Clearing the configuration file: /config/nginx/nginx.conf
loaded secret from file: /data/secret/keys/secretkey
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir


[Step 5]: starting Harbor ...
Creating network "harbor_harbor" with the default driver
Creating network "harbor_harbor-clair" with the default driver
Creating network "harbor_harbor-chartmuseum" with the default driver
Creating harbor-log ... done
Creating harbor-portal     ... done
Creating redis         ... done
Creating registryctl       ... done
Creating harbor-db     ... done
Creating registry      ... done
Creating harbor-core   ... done
Creating harbor-jobservice ... done
Creating nginx             ... done
✔ ----Harbor has been installed and started successfully.----
```  

<br/>

현재 설치된 docker-compose의 버전이 낮아 1.18+ 이상 버전이 설치가 되어야 한다.   

docker compose를 제거하고 아래와 같이 설치한다.  

<br/>

```bash
root@newedu:~/harbor# apt-get remove docker-compose
root@newedu:~/harbor# curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-`uname -s`-`uname -m` -o /usr/bin/docker-compose
root@newedu:~/harbor# chmod +x /usr/bin/docker-compose
```

<br/>

docker compose 버전을 확인하고 1.21.2 버전이 설치 된 것을 확인 할 수 있다.

<br/>


```bash
root@newedu:~/harbor# docker-compose version
docker-compose version 1.21.2, build a133471
docker-py version: 3.3.0
CPython version: 3.6.5
OpenSSL version: OpenSSL 1.0.1t  3 May 2016
```

<br/>

 설치가 완료 되면 아래 명령어로 컨테이너 상태를 확인한다.  

 <br/>

```bash
root@newedu:~/harbor# docker-compose ps
      Name                 Command             State               Ports
--------------------------------------------------------------------------------
chartmuseum         ./docker-               Up (healthy)
                    entrypoint.sh
clair               ./docker-               Up (healthy)
                    entrypoint.sh
clair-adapter       /home/clair-            Up (healthy)
                    adapter/entryp ...
harbor-core         /harbor/entrypoint.sh   Up (healthy)
harbor-db           /docker-entrypoint.sh   Up (healthy)
harbor-jobservice   /harbor/entrypoint.sh   Up (healthy)
harbor-log          /bin/sh -c              Up (healthy)   127.0.0.1:1514->10514
                    /usr/local/bin/ ...                    /tcp
harbor-portal       nginx -g daemon off;    Up (healthy)
nginx               nginx -g daemon off;    Up (healthy)   0.0.0.0:80->8080/tcp,
                                                           :::80->8080/tcp, 0.0.
                                                           0.0:40002->8443/tcp,:
                                                           ::40002->8443/tcp
redis               redis-server            Up (healthy)
                    /etc/redis.conf
registry            /home/harbor/entrypoi   Up (healthy)
                    nt.sh
registryctl         /home/harbor/start.sh   Up (healthy)
```  


<br/>


#### web ui 접속

<br/>

웹브라우저에서 https://211.252.85.148:40002/ 로 접속한다. 

<br/>

https로 만 연결이 가능하며 Not secure 라고 되어 있는 것은  공인된 CA가 아니기 때문이고 사용하는데는 문제가 없다.  

<br/>

<img src="./assets/harbor8.png" style="width: 80%; height: auto;"/> 

<br/>

정상적으로 로그인이 되면 아래와 같은 화면이 나온다.

<br>

<img src="./assets/harbor1.png" style="width: 80%; height: auto;"/> 

<br/>

harbor.yml 에서 설정한 admin 계정과 비밀번호로 로그인 한다.  

<br/>

<img src="./assets/harbor2.png" style="width: 80%; height: auto;"/> 

<br/>

project에 보면 library라는 기본 프로젝트가 있다.   
우리는 edu라는 이름의 신규 프로젝트를 구성한다.  

<br/>

<img src="./assets/harbor3.png" style="width: 60%; height: auto;"/> 

<br/>

docker registry에 접속한 User를 생성한다.  

<img src="./assets/harbor4.png" style="width: 60%; height: auto;"/> 

<br/>

앞에서 생성한 프로젝트인 edu를 선택하고 members tab으로 이동한다.  
member 를 추가하고 권한을 부여한다.  

<img src="./assets/harbor5.png" style="width: 60%; height: auto;"/> 


<br/>

이제 Harbor 설정은 완료 되었다.  

<br/>

#### remote 에서 이미지 push 하기.

<br/>

push 할 서버의 docker 에서  해당 ip와 포트를 insecure registry 에 추가하고 docker를 재기동 한다.    

<br/>

먼저 private docker registry에 로그인 한다.  

<br/>

```bash
jakelee@jake-MacBookAir Downloads % docker login 211.252.85.148:40002
Username: edu
Password:
Login Succeeded
```  

<br/>

tagging을 한다.   

<br/>


```bash
jakelee@jake-MacBookAir Downloads % docker tag nginx 211.252.85.148:40002/edu/nginx
```

<br/>

이미지를 push 한다.  

<br/>

```bash
jakelee@jake-MacBookAir Downloads % docker push  211.252.85.148:40002/edu/nginx
Using default tag: latest
The push refers to repository [211.252.85.148:40002/edu/nginx]
2221dc0783f7: Pushed
9b5201f20365: Pushed
8382eef9f1dd: Pushed
4b88900a24d4: Pushed
aae0e4f98e7e: Pushed
53f02138e713: Pushed
latest: digest: sha256:e62e73dd7f24578c82f40f15b3e6c40b49e33ccb86188f56472fd27b0621ec37 size: 1570
```  

<br/>

정상적으로 push 가 되면 Harbor web 에서 확인한다.  

<img src="./assets/harbor6.png" style="width: 60%; height: auto;"/> 

<br/>

clair 설치를 하면 SCAN 버튼이 활성화 되고 docker image의 취약점을 inspect 할 수 있다.

<br/>

<img src="./assets/harbor7.png" style="width: 60%; height: auto;"/> 

<br/>

#### Harbor 종료 및 재기동

<br/>

harbor 는 docker-compose 로 구성되어 있다.    

종료를 위해서는 harbor 폴더로 이동하고 아래 명령어를 사용한다.

<br/>

```bash
root@newedu:~# cd ~/harbor
root@newedu:~/harbor# docker-compose down -v
Stopping harbor-jobservice ... done
Stopping nginx             ... done
Stopping harbor-core       ... done
Stopping clair-adapter     ... done
Stopping clair             ... done
Stopping chartmuseum       ... done
Stopping registry          ... done
Stopping harbor-db         ... done
Stopping redis             ... done
Stopping registryctl       ... done
Stopping harbor-portal     ... done
Stopping harbor-log        ... done
Removing harbor-jobservice ... done
Removing nginx             ... done
Removing harbor-core       ... done
Removing clair-adapter     ... done
Removing clair             ... done
Removing chartmuseum       ... done
Removing registry          ... done
Removing harbor-db         ... done
Removing redis             ... done
Removing registryctl       ... done
Removing harbor-portal     ... done
Removing harbor-log        ... done
Removing network harbor_harbor
Removing network harbor_harbor-clair
Removing network harbor_harbor-chartmuseum
```  

<br/>

재기동 하기 위해서는 아래와 같이 수행한다.

<br/>

```bash
root@newedu:~/harbor# docker-compose up -d
Creating network "harbor_harbor" with the default driver
Creating network "harbor_harbor-clair" with the default driver
Creating network "harbor_harbor-chartmuseum" with the default driver
Creating harbor-log ... done
Creating registry      ... done
Creating harbor-portal ... done
Creating chartmuseum   ... done
Creating redis         ... done
Creating registryctl   ... done
Creating harbor-db     ... done
Creating harbor-core   ... done
Creating clair         ... done
Creating clair-adapter ... done
Creating nginx             ... done
Creating harbor-jobservice ... done
```  
<br/>

status 를 확인한다.  

<br/>

```bash
root@newedu:~/harbor# docker-compose ps
      Name                     Command                  State                                   Ports
---------------------------------------------------------------------------------------------------------------------------------
chartmuseum         ./docker-entrypoint.sh           Up (healthy)
clair               ./docker-entrypoint.sh           Up (healthy)
clair-adapter       /home/clair-adapter/entryp ...   Up (healthy)
harbor-core         /harbor/entrypoint.sh            Up (healthy)
harbor-db           /docker-entrypoint.sh            Up (healthy)
harbor-jobservice   /harbor/entrypoint.sh            Up (healthy)
harbor-log          /bin/sh -c /usr/local/bin/ ...   Up (healthy)   127.0.0.1:1514->10514/tcp
harbor-portal       nginx -g daemon off;             Up (healthy)
nginx               nginx -g daemon off;             Up (healthy)   0.0.0.0:80->8080/tcp,:::80->8080/tcp,
                                                                    0.0.0.0:40002->8443/tcp,:::40002->8443/tcp
redis               redis-server /etc/redis.conf     Up (healthy)
registry            /home/harbor/entrypoint.sh       Up (healthy)
registryctl         /home/harbor/start.sh            Up (healthy)
```

<br/>

##  2주차

<br/>

### CKA 문제 풀어보기

<br/>