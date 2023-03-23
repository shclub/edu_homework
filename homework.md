# 과제  
 
매 주 마다 과제는 아래와 같다.

1. 1주차
    -  Nexus private registry를 사용하여 Jenkins Pipeline 구성
    -  컨테이너 Mysql DB 활용
    -  Portainer 설치
    -  Harbor 구성해 보기 
2. 2주차 : CKA 문제 풀어 보기 [ 문제 ](./cka.md) 
3. 3주차 : kubernetes에 Jenkins Master/Slave 구성하기 (CI - Python )
4. 4주차 : kubernetes에 CI/CD 구성 (CI : Maven/Skaffold , CD : ArgoCD/kustomize )


<br/>

##  1주차

<br/>

### Nexus private registry를 사용하여 Jenkins Pipeline 구성

<br/>

> 문제 : nexus (또는 Harabor) 로 구성된 private docker registry를 사용하여 pipeline을 신규로 구성한다.  

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

- jenkins에서 본인 nexus (또는 Harbor) 의 docker ci를 신규 생성한다.
- jenkinsfile을 새로 만들고 수정한다.
  - dockerRepo 변경
  - dockerCredentials 은 신규 생성한 docker ci
- docker.withRegistry 의 공백에 본인이 nexus (또는 Harbor) 주소를 입력한다.
- jenkins pipeline 을 copy from 으로 신규 생성하고 Jenkinsfile 이름을 변경한다.

<br/>

```bash
properties([pipelineTriggers([githubPush()])])

pipeline {
    environment {
        // Global 변수 선언
        // Harbor 사용시 : 앞에 project 이름이 붙는다. 예) edu
        dockerRepo = "edu/edu1"
        // nexus
        //dockerRepo = "edu1"
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
                    // Harbor
                    docker.withRegistry('https://211.252.85.148:40002', dockerCredentials){
                    // nexus
                    //docker.withRegistry( 'http://211.252.85.148:40010', dockerCredentials){
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

컨테이너로 접속한다.  

<br/>

```bash
root@newedu:~/edu1 # docker exec -it ef9757c00f48 sh
```  

<br/>

컨테이너안에서 명령어로 mysql에 접속한다.  
id/패스워드는 docker-compose.yml에서 확인 가능  


<br/>

```bash
sh-4.2# mysql -u wordpress -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.41 MySQL Community Server (GPL)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use wordpress;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```  

<br/>

databases 목록을 확인하고 원하는 db를 선택하고   
create table 문을 실행한다.   

<br/>

```bash
mysql> create table customer ( customer_id varchar(10), customer_name varchar(20));
Query OK, 0 rows affected (0.01 sec)
```  

<br/>

테이블 목록을 확인하면 customer 테이블이 생성된 걸 확인 할수 있다.  

<br/>

```bash
mysql> show tables;
+-----------------------+
| Tables_in_wordpress   |
+-----------------------+
| customer              |
| test                  |
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+
14 rows in set (0.00 sec)

mysql> select * from customer;
Empty set (0.00 sec)
```  

<br/>


### Portainer 설치

<br/>

> 문제

<br/>

docker 컨테이너 GUI 관리 툴인 portainer를 설치하고 웹에서 접속하여
          모니터링한다.
  - url  참고 :  https://docs.portainer.io/start/install-ce/server/docker/linux
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


##  3주차

<br/>

### kubernetes에 Jenkins Master/Slave 구성하기

<br/>

참고
- Jenkins Slave 방식 : https://findmypiece.tistory.com/153
- https://akomljen.com/set-up-a-jenkins-ci-cd-pipeline-with-kubernetes/
- https://tech.osci.kr/2020/01/21/kubernetes%EC%83%81%EC%9D%98-jenkins-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EC%9E%91%EC%84%B1%EB%B0%A9%EB%B2%95/
- https://octopus.com/blog/jenkins-helm-install-guide
- https://velog.io/@aylee5/EKS-Helm%EC%9C%BC%EB%A1%9C-Jenkins-%EB%B0%B0%ED%8F%AC-MasterSlave-%EA%B5%AC%EC%A1%B0-with-Persistent-VolumeEBS
- https://ijnuemik.tistory.com/20
- https://www.skyer9.pe.kr/wordpress/?p=6777


<br/>

로그인 한 후에 jenkins 폴더를 생성한다.  

<br/>

yaml 화일 참고 : https://github.com/shclub/jenkins

<br/>

```bash
root@newedu:~# mkdir -p jenkins
root@newedu:~# cd jenkins
```

<br/>

#### 계정 생성과 RBAC 생성

<br/>

jenkins admin 계정의 secret를 생성하기 위해 계정과 비밀번호를 base64로 인코딩 한다.

<br/>

```bash
root@newedu:~/jenkins# echo -n 'admin' | base64
YWRtaW4=
root@newedu:~/jenkins# echo -n 'New1234!' | base64
TmV3MTIzNCE=
```  

<br/>

secret를 만들기 위해 yaml 화일을 생성한다.    

<br/>

```bash
root@newedu:~/jenkins# vi jenkins-edu-admin-secret.yaml
``` 

<br/>

data 부분에 base64 인코딩 된 값을 넣어준다.

<br/>

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: jenkins-admin-secret
data:
  jenkins-admin-user: YWRtaW4=
  jenkins-admin-password: TmV3MTIzNCE=
type: Opaque
```

<br/>

secret를 생성하고 확인한다.

<br/>

```bash
root@newedu:~/jenkins# kubectl apply -f jenkins-edu-admin-secret.yaml
secret/jenkins-admin-secret created
root@newedu:~/jenkins# kubectl get secret
NAME                                 TYPE                                  DATA   AGE
jenkins-admin-secret                 Opaque                                2      7s
my-service-account-dockercfg-4j5n7   kubernetes.io/dockercfg               1      169d
my-service-account-token-d67wk       kubernetes.io/service-account-token   4      169d
my-service-account-token-wxttf       kubernetes.io/service-account-token   4      169d
super-secret                         Opaque                                1      169d
```

<br/>

`jenkins-admin` service account 를 생성 하고 role 과 rolebinding 을 생성한다.  

<br/>


```bash
root@newedu:~/jenkins# vi sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
```

<br/>

```bash
root@newedu:~/jenkins# vi jenkins_rbac.yaml
```

<br/>

jenkins-admin 권한은 최소화 한다.

<br/>

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-admin
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
```

<br/>

본인의 namespace 에 적용한다.

<br/>

```bash
root@newedu:~/jenkins# kubectl apply -f jenkins_rbac.yaml
serviceaccount/jenkins-admin created
role.rbac.authorization.k8s.io/jenkins-admin created
rolebinding.rbac.authorization.k8s.io/jenkins-admin created
```  

<br/>

생성된 resource 들을 확인한다.  

<br/>

```bash
root@newedu:~/jenkins# kubectl get sa
NAME                 SECRETS   AGE
builder              2         247d
default              2         247d
deployer             2         247d
edu                  2         7d20h
jenkins-admin        2         10s
my-service-account   2         169d
root@newedu:~/jenkins# kubectl get role
NAME            CREATED AT
developer       2022-09-27T14:16:45Z
jenkins-admin   2023-03-16T02:03:11Z
pod-role        2022-09-27T14:04:40Z
root@newedu:~/jenkins# kubectl get rolebindings
NAME                              ROLE                                          AGE
admin                             ClusterRole/admin                             247d
developer-binding-myuser          Role/developer                                169d
edu30-admin                       ClusterRole/admin                             247d
jenkins-admin                     Role/jenkins-admin                            23s
pod-rolebinding                   Role/pod-role                                 169d
pod-rolebinding2                  Role/pod-role                                 7d19h
system:deployers                  ClusterRole/system:deployer                   247d
system:image-builders             ClusterRole/system:image-builder              247d
system:image-pullers              ClusterRole/system:image-puller               247d
system:openshift:scc:privileged   ClusterRole/system:openshift:scc:privileged   182d
```  

<br/>

jenkins-admin 으로 OKD에 접속하기 위해 anyuid, privileged 권한을 부여한다.    

privileged 권한은 podman 에서 cri-o 런타임을 연결하기 위해 필요하다.  

<br/>

```bash
root@newedu:~/jenkins# oc adm policy add-scc-to-user anyuid -z jenkins-admin 
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "jenkins-admin"
root@newedu:~/jenkins# oc adm policy add-scc-to-user privileged -z jenkins-admin
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "jenkins-admin"
```  

<br/>

#### Storage 설정

<br/>


Jenkins 가 사용하는 stroage를 위해 pv / pvc 를 생성해야 하며
사전에 NFS 에 접속하여 폴더를 생성한다. 

<br/>

```bash
[root@edu jenkins]# mkdir -p edu
[root@edu jenkins]# mkdir -p edu1
[root@edu jenkins]# ls
edu  edu1  edu10  edu11  edu12  edu13  edu14  edu15  edu16  edu17  edu18  edu19  edu2  edu20  edu21  edu3  edu4  edu5  edu6  edu7  edu8  edu9
```

<br/>

Jenkins slave 용 폴더도 생성한다.

```bash
[root@edu jenkins]# mkdir -p edu
[root@edu jenkins]# mkdir -p edu1_slave
...
```

<br/>

jenkins master / slave 용 해당 폴더의 권한을 설정한다.

<br/>

```bash
[root@edu jenkins]# chown -R nfsnobody:nfsnobody edu
[root@edu jenkins]# chmod 777 edu
[root@edu jenkins]# chown -R nfsnobody:nfsnobody edu_slave
[root@edu jenkins]# chmod 777 edu_slave
```  

<br/>

Master용 PV 를 생성한다. 사이즈는 5G로 설정한다.

<br/>

```bash  
root@newedu:~/jenkins#  vi jenkins_pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-edu-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  nfs:
    path: /share_8c0fade2_649f_4ca5_aeaa_8fd57904f8d5/jenkins/edu
    server: 172.25.1.162
  persistentVolumeReclaimPolicy: Retain
```

<br/>

PV를 생성하고 Status를 확인해보면 Available 로 되어 있는 것을 알 수 있습니다.  

<br/>

```bash
root@newedu:~/jenkins# kubectl apply -f  jenkins_pv.yaml
persistentvolume/jenkins-edu-pv created
root@newedu:~/jenkins# kubectl get pv jenkins-edu-pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
jenkins-edu-pv   5Gi        RWX            Retain           Available                                   10s
```

<br/>

Master용 pvc 를 생성합니다. pvc 이름을 기억합니다.

<br/>

```bash
root@newedu:~/jenkins# vi jenkins_pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-edu-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  volumeName: jenkins-edu-pv
```
<br/>


Slave 용 PV 를 생성한다. 사이즈는 5G로 설정한다.

<br/>

```bash  
root@newedu:~/jenkins#  vi jenkins_slave_pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-edu-slave-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  nfs:
    path: /share_8c0fade2_649f_4ca5_aeaa_8fd57904f8d5/jenkins/edu_slave
    server: 172.25.1.162
  persistentVolumeReclaimPolicy: Retain
```

<br/>

PV를 생성하고 Status를 확인해보면 Available 로 되어 있는 것을 알 수 있습니다.  

<br/>

```bash
root@newedu:~/jenkins# kubectl apply -f  jenkins_slave_pv.yaml
persistentvolume/jenkins-edu-slave-pv created
root@newedu:~/jenkins# kubectl get pv jenkins-edu-slave-pv
NAME                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                         STORAGECLASS   REASON   AGE
jenkins-edu-slave-pv   5Gi        RWX            Retain           Bound    edu30/jenkins-edu-slave-pvc                           100s
```

<br/>

Slave 용 pvc 를 생성합니다. pvc 이름을 기억합니다.

<br/>

```bash
root@newedu:~/jenkins# vi jenkins_slave_pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-edu-slave-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  volumeName: jenkins-edu-slave-pv
```

<br/>

PVC를 생성할 때는 namespace ( 본인의 namespace ) 를 명시해야 합니다.  
PVC 생성을 확인 해보고 다시 PV를 확인해 보면 Status가 Bound 로 되어 있는 것을 알 수 있습니다.  이제 PV 와 PVC가 연결이 되었습니다.

<br/>

```bash
root@newedu:~/jenkins# kubectl apply -f jenkins_slave_pvc.yaml
persistentvolumeclaim/jenkins-edu-slave-pvc created
root@newedu:~/jenkins# kubectl get pvc
NAME                    STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS   AGE
app-volume              Bound    app-config             1Gi        RWX            az-c           169d
jenkins-edu-pvc         Bound    jenkins-edu-pv         5Gi        RWX                           136m
jenkins-edu-slave-pvc   Bound    jenkins-edu-slave-pv   5Gi        RWX                           3m27s
root@newedu:~/jenkins# kubectl get pv jenkins-edu-slave-pv
NAME                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                         STORAGECLASS   REASON   AGE
jenkins-edu-slave-pv   5Gi        RWX            Retain           Bound    edu30/jenkins-edu-slave-pvc                           4m
```

<br/>

#### Helm  Jenkins  설정

<br/>

Jenkins 는 Helm Chart 를 이용하여 설치를 합니다.  

<br/>

현재 로컬의 helm repository 를 확인한다.   

<br/>

```bash
root@newedu:~/jenkins# helm repo list
NAME                           	URL
bitnami                        	https://charts.bitnami.com/bitnami
nfs-subdir-external-provisioner	https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```  

<br/>

jenkins helm repository를  아래와 같이 추가 한다.

<br/>

```bash
root@newedu:~/jenkins# helm repo add jenkins https://charts.jenkins.io --insecure-skip-tls-verify
"jenkins" has been added to your repositories
```

<br/>

helm repository를 update 한다.  

<br/>

```bash
root@newedu:~/jenkins# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "nfs-subdir-external-provisioner" chart repository
...Successfully got an update from the "jenkins" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```  

<br/>

jenkins helm reppository 에서 helm chart를 검색을 하고 jenkins chart를 선택합니다.  

<br/>

```bash
root@newedu:~/jenkins# helm search repo jenkins
NAME           	CHART VERSION	APP VERSION	DESCRIPTION
bitnami/jenkins	12.0.1       	2.387.1    	Jenkins is an open source Continuous Integratio...
jenkins/jenkins	4.3.8        	2.387.1    	Jenkins - Build great things at any scale! The ...
```

<br/>

jenkins/jenkins 차트에서 차트의 변수 값을 변경하기 위해 values.yaml 화일을 추출한다.

<br/>


```bash
root@newedu:~/jenkins#  helm show values jenkins/jenkins > jenkins_values.yaml
```


<br/>

vi 데이터에서 생성된 jenkins_values.yaml을 연다.  

<br/>

```bash
root@newedu:~# vi jenkins_values.yaml
```  

라인을 보기 위해 ESC 를 누른 후 `:set nu` 를 입력하면 왼쪽에 라인이 보인다.  

아래 라인을 찾아 값을 변경한다.  51번 라인에 앞에서 생성한 secret을 넣는다.  

<br/>

```bash
 50   admin:
 51     existingSecret: "jenkins-admin-secret" #
 52     userKey: jenkins-admin-user
 53     passwordKey: jenkins-admin-password
```  

<br/>

```bash
244   installPlugins:
245     - kubernetes:3842.v7ff395ed0cf3 #3734.v562b_b_a_627ea_c
246     - workflow-aggregator:590.v6a_d052e5a_a_b_5
247     - git:4.13.0
248     - configuration-as-code:1569.vb_72405b_80249
...
508   # Openshift route
509   route:
510     enabled: true  # true 로 변경
511     labels: {}
512     annotations: {}
...
617 agent:
618   enabled: true
619   defaultsProviderTemplate: ""
620   # URL for connecting to the Jenkins contoller
621   jenkinsUrl:
622   # connect to the specified host and port, instead of connecting directly to the Jenkins controller
623   jenkinsTunnel:
624   kubernetesConnectTimeout: 5
625   kubernetesReadTimeout: 15
626   maxRequestsPerHostStr: "32"
627   namespace:
628   image: "jenkins/jnlp-slave" #"jenkins/inbound-agent"
629   tag: "latest-jdk11"
...
662   volumes: # []
663   # - type: ConfigMap
664   #   configMapName: myconfigmap
665   #   mountPath: /var/myapp/myconfigmap
666   # - type: EmptyDir
667   #   mountPath: /var/myapp/myemptydir
668   #   memory: false
669   # - type: HostPath
670   #   hostPath: /var/lib/containers
671   #   mountPath: /var/myapp/myhostpath
672   # - type: Nfs
673   #   mountPath: /var/myapp/mynfs
674   #   readOnly: false
675   #   serverAddress: "192.0.2.0"
676   #   serverPath: /var/lib/containers
677    - type: PVC
678      claimName: jenkins-edu-slave-pvc
679      mountPath: /var/jenkins_home
680      readOnly: false
...
691   workspaceVolume: # {}
692   ## DynamicPVC example
693   # type: DynamicPVC
694   # configMapName: myconfigmap
695   ## EmptyDir example
696   # type: EmptyDir
697   # memory: false
698   ## HostPath example
699   # type: HostPath
700   # hostPath: /var/lib/containers
701   ## NFS example
702   # type: Nfs
703   # readOnly: false
704   # serverAddress: "192.0.2.0"
705   # serverPath: /var/lib/containers
706   ## PVC example
707    type: PVC
708    claimName: jenkins-edu-slave-pvc
709    readOnly: false
710   #
710   #
...

822 persistence:
823   enabled: true
824   ## A manually managed Persistent Volume and Claim
825   ## Requires persistence.enabled: true
826   ## If defined, PVC must be created manually before volume will be bound
827   existingClaim: "jenkins-edu-pvc" #
828   ## jenkins data Persistent Volume Storage Class
829   ## If defined, storageClassName: <storageClass>
830   ## If set to "-", storageClassName: "", which disables dynamic provisioning
831   ## If undefined (the default) or set to null, no storageClassName spec is
832   ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
833   ##   GKE, AWS & OpenStack)
834   ##
835   storageClass:
836   annotations: {}
837   labels: {}
838   accessMode: "ReadWriteOnce"
839   size: "5Gi"  # "8Gi"  
...

870 serviceAccount:
871   create: false #  이미 생성 했기 때문에 false 로 변경
872   # The name of the service account is autogenerated by default
873   name: "jenkins-admin" #
874   annotations: {}
875   extraLabels: {}
876   imagePullSecretName:

```  
<br/>

#### Helm 으로 Jenkins 설치

<br/>

jenkins_values.yaml 를 사용하여 설치 한다.

<br/>

```bash
root@newedu:~/jenkins# helm install jenkins  -f jenkins_values.yaml jenkins/jenkins
NAME: jenkins
LAST DEPLOYED: Thu Mar 16 16:33:52 2023
NAMESPACE: edu30
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace edu30 -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  echo http://127.0.0.1:8080
  kubectl --namespace edu30 port-forward svc/jenkins 8080:8080

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://127.0.0.1:8080/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
```   

<br/>

설치가 완료되면 pod를 조회하여 jenkins-0 pod가 있는지 확인한다.

<br/>

```bash
root@newedu:~/jenkins# kubectl get po
NAME                                        READY   STATUS    RESTARTS   AGE
jenkins-0                                   2/2     Running   0          18h
nfs-test-589c488d6f-8lk5p                   1/1     Running   0          40h
``` 

<br/>

#### Jenkins 설정

<br/>

웹 브라우저 에서 본인의 jenkins 로 접속한다.   
route 정보를 모르면 아래 명령어로 조회한다.

<br/>

```bash
root@newedu:~/jenkins# kubectl get route
NAME      HOST/PORT                                 PATH   SERVICES   PORT   TERMINATION     WILDCARD
jenkins   jenkins-edu30.apps.211-34-231-82.nip.io          jenkins    http   edge/Redirect   None
```  

<br/>

https://jenkins-edu30.apps.211-34-231-82.nip.io/


<br/>

아래 처럼 로그인 화면이 나오면 admin 계정으로 로그인한다.

<br/>

<img src="./assets/jenkins_login.png" style="width: 80%; height: auto;"/> 

<br/>

Manage Jenkins 메뉴를 클릭한다.

<br/>

<img src="./assets/manage_jenkins1.png" style="width: 80%; height: auto;"/> 

Manage Plugins 메뉴를 클릭한다.

<br/>

<img src="./assets/manage_plugins.png" style="width: 80%; height: auto;"/> 

<br/>

Plugin은 3 가지를 설치한다.  
- git parameter
- pipeline stage view
- docker pipeline  

<br/>

git parameter

<img src="./assets/git_parameter.png" style="width: 80%; height: auto;"/> 

<br/>

install without restart 로 설치한다.   

<br/>

pipeline stage view  

<img src="./assets/pipeline_stage_view.png" style="width: 80%; height: auto;"/> 


docker pipeline  

<img src="./assets/docker_pipeline.png" style="width: 80%; height: auto;"/> 


<br/>

plugin 설정을 완료 하면 Manage jenkins -> Manage Nodes and Clouds 메뉴로 이동하여 Jenkins 설정을 한다.  

<br/>

<img src="./assets/manage_clouds1.png" style="width: 80%; height: auto;"/> 

<br/>

Configure Clouds로 이동한다.  

<br/>

<img src="./assets/manage_clouds2.png" style="width: 80%; height: auto;"/>   

<br/>

Configure Clouds에 가면  kubernetes Cloud Details 가 Jenkins Master 설정이고 Pod Template 에 Slave 설정을 하면 된다.  

<br/>

<img src="./assets/manage_clouds3.png" style="width: 80%; height: auto;"/> 


<br/>

kubernetes Cloud Details 를 확장하여 자세히 보면 Jenkins Master POD 가 있는 namespace 를 확인 할 수 있다.  

<br/>

<img src="./assets/manage_clouds4.png" style="width: 80%; height: auto;"/> 

기본값으로 설정된 값을 유지한다.

<br/>

<img src="./assets/manage_clouds5.png" style="width: 80%; height: auto;"/> 

<br/>

POD Labels의 key 값만 변경한다. 안해도 상관 없음  

<br/>

<img src="./assets/manage_clouds6.png" style="width: 80%; height: auto;"/> 

<br/><br/>

Pod Template 을 확장하여 Slave 설정을 시작한다.    
우리는 OKD 에 Jenkins를 설치를  하였고 OKD는 docker runtime 대신 CRI-O runtime을 사용하기 때문에 podman 으로 docker 를 대신한다.   

<br/>

podman-agent라는 이름으로 설정을 하고 Pod template details 를 클릭한다.   

<br/>

<img src="./assets/manage_clouds7.png" style="width: 80%; height: auto;"/> 

<br/>

Pod Template Label은  podman-agent 로 설정한다.    

<br/>

하나의 POD에 2개의 컨테이너를 설정한다.
- JNLP 컨테이너 : jenkins slave 를 위한 컨테이너
- Podman 컨테이너 : Podman 프로그램이 설치된 컨테이너  
   - podman 컨테이너의 /etc/containers와 /var/lib/containers를 hostpath로 연결해 주어야 한다.
   - podman에서 worker node의 cri-o 런타임 엔진을 사용  
   
<br/>

먼저 jnlp 컨테이너 값을 설정한다.  
- docker image 를 변경한다 : jenkins/jnlp-slave:latest-jdk11  
- command to run 을 기존 sleep 값을 지운다.

<br/>

<img src="./assets/manage_clouds8.png" style="width: 80%; height: auto;"/> 

<br/>

해당 slave pod에 추가 컨테이너 값을 설정하기 위해 add container를 클릭하고 Container Template를 선택한다.  

<br/>

<img src="./assets/manage_clouds9.png" style="width: 40%; height: auto;"/> 

<br/>

podman 컨테이너를 설정한다.    
- 이름을 설정한다 : podman
- docker image 를 설정한다 : mattermost/podman:1.8.0  
- Advanced 메뉴를 클릭하고 Run in privileged mode를 체크한다. (hostpath)  

<br/>

<img src="./assets/manage_clouds10.png" style="width: 80%; height: auto;"/> 

<br/>

Add volume 버튼을 클릭하고 slave 용 pvc와 podman용 hostpath를 설정한다.   

slave용 pvc는 이미 설정이되어 있기 때문에 hostpath 만 설정해도 된다.  

<br/>

<img src="./assets/manage_clouds11.png" style="width: 80%; height: auto;"/> 

<br/>

아래로 이동하여 service account를 `jenkins-admin` 으로 변경한다.  

workspace volume 도 helm 에서 사전에 설정 했기 때문에 확인만 하고 save 버튼을 눌러 저장한다.  

<br/>

<img src="./assets/manage_clouds12.png" style="width: 80%; height: auto;"/> 

<br/>


#### Jenkins Pipeline 생성

<br/>

대쉬보드로 이동하여 신규 pipeline을 구성 한다.   

New Item 버튼을 클릭한다.

<img src="./assets/jenkins_pipeline1.png" style="width: 40%; height: auto;"/> 

<br/>

Item 이름을 설정하고 pipeline 을 선택한 후 ok 버튼을 클릭한다.

<br/>

<img src="./assets/jenkins_pipeline2.png" style="width: 60%; height: auto;"/> 

<br/>

Git Parameter 를 설정 한다. 
Git Parameter 가 안보이는 수강생은 plugins 에서 git parameter를 설치해야 한다.  

<br/>

<img src="./assets/jenkins_pipeline3.png" style="width: 60%; height: auto;"/> 

<br/>

git Repostory는 본인의 git 주소를 입력하고 script path를 입력한다.    

<br/>

강사 edu1 repository ( (https://github.com/shclub/edu1) )의 Jenkinsfile_okd 화일을 참고하여 신규 생성한다.    
( 현재 OKD 네트웍 이슈로 Harbor로 연결 안되어 docker hub 로 연결 하여 생성. Jenkinsfile_dockerhub 참고 )

<br/>

<img src="./assets/jenkins_pipeline4.png" style="width: 60%; height: auto;"/> 

<br/>

credential은 Add 버튼을 클릭하여 github_ci 와 harbor_ci 라는 이름 으로 생성한다.  

harbor 가 구성이 안된 사람은  id는 `edu` , 비밀번호는 `New1234!` 로 구성하여 생성한다.  

<br/>

<img src="./assets/jenkins_pipeline5.png" style="width: 60%; height: auto;"/> 

<br/>

github의 Jenkinsfile_okd 는 본인의 환경에 맞게 변경한다.    
- def docker_registry = "https://211.252.85.148:40002"  
  - 예) edu1 : https://211.43.12.162:40002"
  - 예) 본인 Harbor 가 없으면 수정 하지 말것 
- def imageName = "211.252.85.148:40002/edu/edu1"
  - 예) edu1 : 211.43.12.162:40002/<본인프로젝트>/edu1
  - 예) 본인 Harbor 가 없으면서 순번이 2번인 경우 : 211.252.85.148:40002/edu2/edu1
- podTemplate의 namespace는 본인의 namespace 로 반드시 변경
- persistentVolumeClaim 의 claim은 본인의 slave claim으로 변경
  - 예) edu1 : jenkins-edu1-slave-pvc
- git url은 본인의 github repository 로 변경
- harbor credential은 다른 이름으로 생성했으면 그것에 맞추어 변경

<br/>

<img src="./assets/jenkins_pipeline6.png" style="width: 60%; height: auto;"/> 

<br/>

대쉬보드로 돌아와서 본인의 파이프 라인 클릭

<br/>

<img src="./assets/jenkins_pipeline7.png" style="width: 60%; height: auto;"/> 

<br/>

Build with parameters 로 실행

<br/>

<img src="./assets/jenkins_pipeline8.png" style="width: 60%; height: auto;"/> 

<br/>

성공적으로 진행이 된 것을 확인한다.  

<br/>

<img src="./assets/jenkins_pipeline9.png" style="width: 60%; height: auto;"/> 

<br/>

Harbor로 이동하여 이미지가 정상적으로 Push 되었는지 확인한다.      

강사의 Harbor : https://211.252.85.148:40002/ 
- edu 계정으로 로그인
- 본인의 project로 이동하여 edu1 도커이미지 확인  

<br/>

<img src="./assets/jenkins_pipeline10.png" style="width: 60%; height: auto;"/> 

<br/>


강사의 Jenkjns : https://jenkins-edu30.apps.211-34-231-82.nip.io/   
- 계정 : admin
  