# 과제  
 
매 주 마다 과제는 아래와 같다.

1. 1주차
    -  Nexus private registry를 사용하여 Jenkins Pipeline 구성
    -  컨테이너 Mysql DB 활용
    -  Portainer 설치
    -  Harbor 구성해 보기 
2. 2주차 : CKA 문제 풀어 보기 [ 문제 ](./cka.md) 
3. 3주차 : kubernetes에 Jenkins Master/Slave 구성하기 (CI - Python )
4. 4주차 : kubernetes에 CI/CD 구성 (CI : Maven/Skaffold/SonarQube , CD : ArgoCD/kustomize )


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

> 폴더는 사전에 강사가 생성하여 별도 생성 불필요

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

pod 내에서 nfs 연결해서 권한을 줄때는   

`chown -R nfsnobody:nfsnobody edu`  

대신 아래처럼 nobody:nogroup 으로 준다.

`chown -R nobody:nogroup edu`

<br/>


```bash
[root@edu jenkins]# chown -R nfsnobody:nfsnobody edu
[root@edu jenkins]# chmod 777 edu
[root@edu jenkins]# chown -R nfsnobody:nfsnobody edu_slave
[root@edu jenkins]# chmod 777 edu_slave
```  

<br/>

> 여기부터는 생성해야 함.  

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

jenkins/jenkins 차트에서 차트의 변수 값을 변경하기 위해 jenkins_values.yaml 화일을 추출한다.

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

kubernetes plugin은 반드시 아래와 같이 수정 필요.  
현재 jenkins과 호환되는 버전

<br/>

```bash
 91   javaOpts: "-Duser.timezone=Asia/Seoul"
 ...
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
처음에는 plugin 이 설치가 됨으로 빠를 때는 1분 이내 늦을때는 8~10분 정도 시간이 필요하다.  

<br/>

```bash
root@newedu:~/jenkins# kubectl get po
NAME                                        READY   STATUS    RESTARTS   AGE
jenkins-0                                   2/2     Running   0          18h
nfs-test-589c488d6f-8lk5p                   1/1     Running   0          40h
``` 
<br/>

아래 명령어로 plugin 설치가 되는 과정을 볼 수 있다.

<br/>

```bash
root@newedu:~/jenkins# kubectl logs jenkins1-0 -c init 
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


 <br/>


##  4주차

<br/>

### kubernetes에 CI/CD 구성 (CI : Maven/Skaffold/SonarQube , CD : ArgoCD/kustomize )

<br/>

참고
- https://metleeha.tistory.com/m/entry/Kubernetes-Cluster%EC%97%90-Helm%EC%9C%BC%EB%A1%9C-Sonarqube-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0
 

<br/>

VM에 로그인 한 후에 sonar 폴더를 생성한다.  

<br/>

yaml 화일 참고 : https://github.com/shclub/sonarqube

<br/>

```bash
root@newedu:~# mkdir -p sonar
root@newedu:~# cd sonar
``` 

<br/>

#### Storage 설정

<br/>


PostgreSQL 과 SonarQube 가 사용하는 stroage를 위해 pv / pvc 를 생성해야 하며
사전에 NFS 에 접속하여 폴더를 생성한다. 


<br/>

postgresql 은 아래 폴더에 생성되어 있고 수강생은 본인의 폴더 직접 생성.

<br/>

```bash
[root@edu postgre]# pwd
/mnt/postgre
[root@edu postgre]# mkdir -p edu
```

<br/>

SonarQube 용 폴더도 생성한다.

```bash
[root@edu sonar]# pwd
/mnt/sonar
[root@edu sonar]# mkdir -p edu
...
```

<br/>

postgresql / SonarQube 용 해당 폴더의 권한을 설정한다.

<br/>

pod 내에서 nfs 연결해서 권한을 줄때는   

`chown -R nfsnobody:nfsnobody edu`  

대신 아래처럼 nobody:nogroup 으로 준다.

`chown -R nobody:nogroup edu`

<br/>


```bash
[root@edu postgre]# chown -R nfsnobody:nfsnobody edu
[root@edu postgre]# chmod 777 edu
[root@edu sonar]# chown -R nfsnobody:nfsnobody edu
[root@edu sonar]# chmod 777 edu
```  


<br/>

postgresql 용 PV 를 생성한다. 사이즈는 5G로 설정한다.

<br/>

```bash  
root@newedu:~/sonar#  vi postgre_pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgre-edu-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  nfs:
    path: /share_8c0fade2_649f_4ca5_aeaa_8fd57904f8d5/postgre/edu
    server: 172.25.1.162
  persistentVolumeReclaimPolicy: Retain
```

<br/>

PV를 생성하고 Status를 확인해보면 Available 로 되어 있는 것을 알 수 있습니다.  

<br/>

```bash
root@newedu:~/sonar# kubectl apply -f  postgre_pv.yaml
```

<br/>

postgresql용  pvc 를 생성합니다. pvc 이름을 기억합니다.

<br/>

```bash
root@newedu:~/sonar# vi postgre_pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgre-edu-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  volumeName: postgre-edu-pv
```
<br/>


SonarQube 용 PV 를 생성한다. 사이즈는 5G로 설정한다.

<br/>

```bash  
root@newedu:~/sonar#  vi sonar_pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sonar-edu-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  nfs:
    path: /share_8c0fade2_649f_4ca5_aeaa_8fd57904f8d5/sonar/edu
    server: 172.25.1.162
  persistentVolumeReclaimPolicy: Retain
```

<br/>

PV를 생성하고 Status를 확인해보면 Available 로 되어 있는 것을 알 수 있습니다.  

<br/>

```bash
root@newedu:~/sonar# kubectl apply -f  sonar_pvc.yaml
```

<br/>

SonarQube 용 pvc 를 생성합니다. pvc 이름을 기억합니다.

<br/>

```bash
root@newedu:~/sonar# vi sonar_pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonar-edu-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  volumeName: sonar-edu-pv
```

<br/>

PVC를 생성할 때는 namespace ( 본인의 namespace ) 를 명시해야 합니다.  
PVC 생성을 확인 해보고 다시 PV를 확인해 보면 Status가 Bound 로 되어 있는 것을 알 수 있습니다.  이제 PV 와 PVC가 연결이 되었습니다.

<br/>

```bash
root@newedu:~/sonar# kubectl apply -f sonar_pvc.yaml
```

<br/>

#### Helm 으로 PostgreSQL 설치

<br/>

helm repo 업데이트를 합니다.  

```bash
root@newedu:~/sonar# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jenkins" chart repository
...Successfully got an update from the "nfs-subdir-external-provisioner" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```  

<br/>


helm 에서 postgreSQL 를 검색합니다.

```bash
root@newedu:~/sonar# helm search repo postgresql
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION
bitnami/postgresql   	12.2.6       	15.2.0     	PostgreSQL (Postgres) is an open source object-...
bitnami/postgresql-ha	11.2.0       	15.2.0     	This PostgreSQL cluster solution includes the P...
bitnami/supabase     	0.1.4        	0.23.2     	Supabase is an open source Firebase alternative...
```  

<br/>

bitnami/postgresql 차트에서 차트의 변수 값을 변경하기 위해 postgre_values.yaml 화일을 추출한다.

<br/>


```bash
root@newedu:~/sonar#  helm show values bitnami/postgresql  > postgre_values.yaml
```

<br/>

postgre_values.yaml 를 수정한다.  

- 28 ,29,30,31 : 본인이 DB 계정 설정
- 646 : 본인의 pvc로 변경
- 669 : 5G로 사이즈 변경
- 694 : read replica 0

<br/>

```bash
  27     auth:
  28       postgresPassword: "edu1234"
  29       username: "edu"
  30       password: "edu1234"
  31       database: "edu"
  32       existingSecret: ""
 ... 
 640   persistence:
 641     ## @param primary.persistence.enabled Enable PostgreSQL Primary data persistence using PVC
 642     ##
 643     enabled: true
 644     ## @param primary.persistence.existingClaim Name of an existing PVC to use
 645     ##
 646     existingClaim: "postgre-edu-pvc"
 647     ## @param primary.persistence.mountPath The path the volume will be mounted at
 648     ## Note: useful when using custom PostgreSQL images
 649     ##
 650     mountPath: /bitnami/postgresql
 651     ## @param primary.persistence.subPath The subdirectory of the volume to mount to
 652     ## Useful in dev environments and one PV for multiple services
 653     ##
 654     subPath: ""
 655     ## @param primary.persistence.storageClass PVC Storage Class for PostgreSQL Primary data volume
 656     ## If defined, storageClassName: <storageClass>
 657     ## If set to "-", storageClassName: "", which disables dynamic provisioning
 658     ## If undefined (the default) or set to null, no storageClassName spec is
 659     ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
 660     ##   GKE, AWS & OpenStack)
 661     ##
 662     storageClass: ""
 663     ## @param primary.persistence.accessModes PVC Access Mode for PostgreSQL volume
 664     ##
 665     accessModes:
 666       - ReadWriteOnce
 667     ## @param primary.persistence.size PVC Storage Request for PostgreSQL volume
 668     ##
 669     size: 5Gi
...
 688 readReplicas:
 689   ## @param readReplicas.name Name of the read replicas database (eg secondary, slave, ...)
 690   ##
 691   name: read
 692   ## @param readReplicas.replicaCount Number of PostgreSQL read only replicas
 693   ##
 694   replicaCount: 0
 ```

<br/>

이제 postgreSQL DB를 설치 합니다.

<br/>

```bash
root@newedu:~/sonar# helm install sonar-postgre bitnami/postgresql -f postgre_values.yaml
NAME: sonar-postgre
LAST DEPLOYED: Mon Mar 27 10:12:46 2023
NAMESPACE: edu30
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 12.2.6
APP VERSION: 15.2.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    sonar-postgre-postgresql.edu30.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_ADMIN_PASSWORD=$(kubectl get secret --namespace edu30 sonar-postgre-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To get the password for "edu" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace edu30 sonar-postgre-postgresql -o jsonpath="{.data.password}" | base64 -d)

To connect to your database run the following command:

    kubectl run sonar-postgre-postgresql-client --rm --tty -i --restart='Never' --namespace edu30 --image docker.io/bitnami/postgresql:15.2.0-debian-11-r14 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host sonar-postgre-postgresql -U edu -d edu -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace edu30 svc/sonar-postgre-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U edu -d edu -p 5432

WARNING: The configured password will be ignored on new installation in case when previous Posgresql release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.
```

<br/>

pod를 확인한다.

```bash
root@newedu:~/sonar# kubectl get po
NAME                                        READY   STATUS    RESTARTS   AGE
sonar-postgre-postgresql-0                  1/1     Running   0          82s
```

<br/>

NFS 서버에 접속하여 data 폴더가 생성 되었는지 확인한다.  

```bash
[root@edu edu]# pwd
/mnt/postgre/edu
[root@edu edu]# ls data
PG_VERSION  pg_commit_ts   pg_logical    pg_replslot   pg_stat      pg_tblspc    pg_xact               postmaster.pid
base        pg_dynshmem    pg_multixact  pg_serial     pg_stat_tmp  pg_twophase  postgresql.auto.conf
global      pg_ident.conf  pg_notify     pg_snapshots  pg_subtrans  pg_wal       postmaster.opts
```  

<br/>


#### Helm 으로 SonarQube 설치


<br/>


helm repo 업데이트를 합니다.  

```bash
root@newedu:~/sonar# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jenkins" chart repository
...Successfully got an update from the "nfs-subdir-external-provisioner" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```  


<br/>

helm 에서 sonarqube 를 검색합니다.

```bash
root@newedu:~/sonar# helm search repo sonarqube
NAME                 	CHART VERSION	APP VERSION	DESCRIPTION
NAME             	CHART VERSION	APP VERSION    	DESCRIPTION
bitnami/sonarqube	2.1.4        	9.9.0          	SonarQube(TM) is an open source quality managem...
```  

<br/>

bitnami/sonarqube 차트에서 차트의 변수 값을 변경하기 위해 sonarqube_values.yaml 화일을 추출한다.

<br/>


```bash
root@newedu:~/sonar#  helm show values bitnami/sonarqube  > sonarqube_values.yaml
```

<br/>

먼저 postgreSQL DB의 서비스 이름을 확인합니다.  


```bash
root@newedu:~/sonar# kubectl get svc
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
jenkins                       NodePort    172.30.247.178   <none>        8080:30332/TCP   10d
jenkins-agent                 ClusterIP   172.30.140.40    <none>        50000/TCP        10d
sonar-postgre-postgresql      ClusterIP   172.30.225.163   <none>        5432/TCP         19m
```  


<br/>

sonarqube_values.yaml 를 수정한다.  

- 708 : pvc 사용으로 true
- 722 : 5G로 사이즈 변경
- 728 : 본인의 pvc로 변경
- 1004 : 이미 postgresql db 설치 했기 때문에 false
- 1065 : postgresql db 서비스 이름 ( 위에서 조회 )

<br/>

```bash
 706 persistence:
 707   ## @param persistence.enabled Enable persistence using Persistent Volume Claims
 708   ##
 709   enabled: true
 710   ## @param persistence.storageClass Persistent Volume storage class
 711   ## If defined, storageClassName: <storageClass>
 712   ## If set to "-", storageClassName: "", which disables dynamic provisioning
 713   ## If undefined (the default) or set to null, no storageClassName spec is set, choosing the default provisioner
 714   ##
 715   storageClass: ""
 716   ## @param persistence.accessModes [array] Persistent Volume access modes
 717   ##
 718   accessModes:
 719     - ReadWriteOnce
 720   ## @param persistence.size Persistent Volume size
 721   ##
 722   size: 5Gi
 723   ## @param persistence.dataSource Custom PVC data source
 724   ##
 725   dataSource: {}
 726   ## @param persistence.existingClaim The name of an existing PVC to use for persistence
 727   ##
 728   existingClaim: "sonar-edu-pvc"
...
1001 postgresql:
1002   ## @param postgresql.enabled Deploy PostgreSQL subchart
1003   ##
1004   enabled: false
1005   ## @param postgresql.nameOverride Override name of the PostgreSQL chart
...
1062 externalDatabase:
1063   ## @param externalDatabase.host Host of an external PostgreSQL instance to connect (only if postgresql.enabled=false)
1064   ##
1065   host: "sonar-postgre-postgresql"
1066   ## @param externalDatabase.user User of an external PostgreSQL instance to connect (only if postgresql.enabled=false)
1067   ##
1068   user: edu
1069   ## @param externalDatabase.password Password of an external PostgreSQL instance to connect (only if postgresql.enabled=fa     lse)
1070   ##
1071   password: "edu1234"
1072   ## @param externalDatabase.existingSecret Secret containing the password of an external PostgreSQL instance to connect (o     nly if postgresql.enabled=false)
1073   ## Name of an existing secret resource containing the DB password in a 'password' key
1074   ##
1075   existingSecret: ""
1076   ## @param externalDatabase.database Database inside an external PostgreSQL to connect (only if postgresql.enabled=false)
1077   ##
1078   database: edu
 ```

<br/>

sonarqube를 설치 하기 전에 sonarqube service account 에게 권한을 부여합니다.  

```bash
root@newedu:~/sonar# oc adm policy add-scc-to-user anyuid -z sonarqube
root@newedu:~/sonar# oc adm policy add-scc-to-user privileged -z sonarqube
```  

<br/>

이제 sonarqube 를 설치 합니다.


```bash
root@newedu:~/sonar# helm install sonarqube bitnami/sonarqube -f sonarqube_values.yaml
NAME: sonarqube
LAST DEPLOYED: Mon Mar 27 10:51:23 2023
NAMESPACE: edu30
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

Your SonarQube(TM) site can be accessed through the following DNS name from within your cluster:

    sonarqube.edu30.svc.cluster.local (port 80)

To access your SonarQube(TM) site from outside the cluster follow the steps below:

1. Get the SonarQube(TM) URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace edu30 -w sonarqube'

   export SERVICE_IP=$(kubectl get svc --namespace edu30 sonarqube --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
   echo "SonarQube(TM) URL: http://$SERVICE_IP/"

2. Open a browser and access SonarQube(TM) using the obtained URL.

3. Login with the following credentials below:

  echo Username: user
  echo Password: $(kubectl get secret --namespace edu30 sonarqube -o jsonpath="{.data.sonarqube-password}" | base64 -d)
  ```

<br/>

pod를 확인한다.

```bash
root@newedu:~/sonar# kubectl get po
NAME                                        READY   STATUS    RESTARTS   AGE
jenkins-0                                   2/2     Running   26         9d
nfs-test-589c488d6f-8lk5p                   1/1     Running   2          10d
sonar-postgre-postgresql-0                  1/1     Running   0          54m
sonarqube-5d48b66455-5sgzh                  1/1     Running   0          3m7s
```

<br/>

NFS 서버에 접속하여 data 폴더가 생성 되었는지 확인한다.  

```bash
[root@edu edu]# pwd
/mnt/sonar/edu
[root@edu edu]# ls
data  extensions
```  

<br/>

서비스를 조회해서 LoadBalancer Type을 NodePort로 변경합니다.  

```bash
root@newedu:~/sonar# kubectl get svc
NAME                          TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
jenkins                       NodePort       172.30.247.178   <none>        8080:30332/TCP                10d
jenkins-agent                 ClusterIP      172.30.140.40    <none>        50000/TCP                     10d
sonar-postgre-postgresql      ClusterIP      172.30.225.163   <none>        5432/TCP                      64m
sonar-postgre-postgresql-hl   ClusterIP      None             <none>        5432/TCP                      64m
sonarqube                     LoadBalancer   172.30.154.233   <pending>     80:30262/TCP,9001:30118/TCP   13m
``` 

<br/>

```bash
root@newedu:~/sonar# kubectl edit svc sonarqube
service/sonarqube edited
``` 

<br/>

```bash
root@newedu:~/sonar# kubectl get svc
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
jenkins                       NodePort    172.30.247.178   <none>        8080:30332/TCP                10d
jenkins-agent                 ClusterIP   172.30.140.40    <none>        50000/TCP                     10d
sonar-postgre-postgresql      ClusterIP   172.30.225.163   <none>        5432/TCP                      64m
sonar-postgre-postgresql-hl   ClusterIP   None             <none>        5432/TCP                      64m
sonarqube                     NodePort    172.30.154.233   <none>        80:30262/TCP,9001:30118/TCP   13m
```  

<br/>

웹브라우저에서 http://211.34.231.84:30262/로 접속하여 로그인합니다.   

- id 는 user
- 비밀번호는 아래와 같이 추출합니다.
  ```bash
  root@newedu:~/sonar# kubectl get secret  sonarqube -o jsonpath="{.data.sonarqube-password}" | base64 -d
  agLMkOCVqv
  ```  

<br/>


<img src="./assets/sonarqube1.png" style="width: 80%; height: auto;"> 

<br/>


####  SonarQube 로 프로젝트 구성


<br/>

참고
- https://docs.sonarqube.org/9.8/analyzing-source-code/scanners/sonarscanner-for-maven/
- https://twofootdog.tistory.com/15
- https://happy-jjang-a.tistory.com/26

<br/>

SonarQube에 로그인 한 후에 제일 먼저 Admin 비밀빌번호를 변경한다.  
Account ->  My Account 클릭한 후 security tab으로 이동한다.  

<img src="./assets/sonarqube_token1.png" style="width: 60%; height: auto;"> 

<br/>

비밀번호를 변경한다.  

<img src="./assets/sonarqube3.png" style="width: 80%; height: auto;"> 

<br/>

프로젝트를 생성한다.  manual 로 만든다.

<img src="./assets/sonarqube4.png" style="width: 80%; height: auto;"> 


<br/>

원하는 이름을 넣고 branch에는 master를 입력한다.

<img src="./assets/sonarqube5.png" style="width: 80%; height: auto;"> 


<br/>

그 다음 with Jenkins를 클릭하여 필요한 정보를 확인한다.  설정은 별도로 안해도 된다.  

<img src="./assets/sonarqube6.png" style="width: 80%; height: auto;"> 


<br/><br/>

Jenkins와 연동하기 위해서는 Token 값을 생성해야 합니다.  
계정으로 이동합니다.

<img src="./assets/sonarqube_token1.png" style="width: 80%; height: auto;"> 

<br/>

security 탭으로 이동합니다.  

<img src="./assets/sonarqube_token2.png" style="width: 80%; height: auto;"> 

<br/>

Type은 Project Analysys Token 으로 설정하고 Generate 버튼을 클릭합니다. 

<img src="./assets/sonarqube_token3.png" style="width: 80%; height: auto;"> 


<br/>

생성된 Token 값을 COPY 버튼을 클릭하여 복사하여 적당하 곳에 저장합니다.  

<img src="./assets/sonarqube_token4.png" style="width: 80%; height: auto;"> 

<br/>


SonarQube 프로젝트를 구성하기 위해서는 backend 인 SpringBoot의 pom.xml 화일에 dependency를 추가 한다.  


```xml
		<plugin>
				<groupId>org.sonarsource.scanner.maven</groupId>
				<artifactId>sonar-maven-plugin</artifactId>
				<version>3.7.0.1746</version>
		</plugin>
``` 

<br/>

이제 jenkins를 설정합니다.  

Jenkins는 먼저  SonarQube plugins 을 설치합니다.   
- Sonarqube Scanner
- Sonar Quality Gates

<img src="./assets/sonarqube_jenkins1.png" style="width: 80%; height: auto;"> 


<br/>

Manage Jenkins -> Configure System 으로 이동하여 SonarQube 서버 설정을 합니다.  

<img src="./assets/sonarqube_jenkins2.png" style="width: 80%; height: auto;"> 


<br/>

Add Credential 을 추가 하는데 Secret Text로 설정하여 SonarQube Token 값을 설정 합니다.  

<img src="./assets/sonarqube_jenkins3.png" style="width: 80%; height: auto;"> 


<br/>

설정한 정보가 맞는지 확인 하고 save 합니다. 

<img src="./assets/sonarqube_jenkins4.png" style="width: 80%; height: auto;">

<br/>

Manage Jenkins -> Global Tool Configuration 이동하여 Sonar Scanner 설정을 합니다.  

<img src="./assets/sonarqube_jenkins7.png" style="width: 80%; height: auto;"> 

<br/>

이제 설정이 완료 되었음으로 Dashboard -> New Item 으로 이동하여 새로운 Pipeline을 생성합니다.  edu-backend pipeline을 복사하여 생성하며 jenkins 화일 이름만 변경 합니다.  

<br/>
 
Build with paramameter를 클릭하여 실행을 하고 console Output 을 확인합니다.   

아래와 같은 메시지가 나오면 연동이 성공 한 것입니다.  


```bash
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonarsource/scanner/api/sonar-scanner-api/2.14.0.2002/sonar-scanner-api-2.14.0.2002.jar (625 kB at 42 MB/s)
Downloaded from central: https://repo.maven.apache.org/maven2/org/sonatype/plexus/plexus-sec-dispatcher/1.4/plexus-sec-dispatcher-1.4.jar (28 kB at 792 kB/s)
[INFO] User cache: /root/.sonar/cache
[INFO] SonarQube version: 9.9.0.65466
[INFO] Default locale: "en_US", source code encoding: "UTF-8"
[INFO] Load global settings
[INFO] Load global settings (done) | time=126ms
[INFO] Server id: 296AEF55-AYcgz2EaLi3MhYbotBq5
[INFO] User cache: /root/.sonar/cache
[INFO] Load/download plugins
[INFO] Load plugins index
[INFO] Load plugins index (done) | time=67ms
[INFO] Load/download plugins (done) | time=1142ms
[INFO] Process project properties
[INFO] Process project properties (done) | time=10ms
[INFO] Execute project builders
[INFO] Execute project builders (done) | time=2ms
[INFO] Project key: edu
[INFO] Base dir: /home/jenkins/agent/workspace/edu_backend_sonar
[INFO] Working dir: /home/jenkins/agent/workspace/edu_backend_sonar/target/sonar
[INFO] Load project settings for component key: 'edu'
[INFO] Load project settings for component key: 'edu' (done) | time=26ms
[INFO] Auto-configuring with CI 'Jenkins'
[INFO] Load quality profiles
[INFO] Load quality profiles (done) | time=69ms
[INFO] Load active rules
[INFO] Load active rules (done) | time=1556ms
[INFO] Load analysis cache
[INFO] Load analysis cache (404) | time=9ms
[INFO] Load project repositories
[INFO] Load project repositories (done) | time=22ms
[INFO] Indexing files...
[INFO] Project configuration:
[INFO] 25 files indexed
[INFO] 0 files ignored because of scm ignore settings
[INFO] Quality profile for java: Sonar way
[INFO] Quality profile for xml: Sonar way
[INFO] ------------- Run sensors on module thirdproject
[INFO] Load metrics repository
[INFO] Load metrics repository (done) | time=20ms
[INFO] Sensor JavaSensor [java]
[INFO] Configured Java source version (sonar.java.source): 17
[INFO] JavaClasspath initialization
[INFO] JavaClasspath initialization (done) | time=6ms
[INFO] JavaTestClasspath initialization
[INFO] JavaTestClasspath initialization (done) | time=3ms
[INFO] Server-side caching is enabled. The Java analyzer will not try to leverage data from a previous analysis.
[INFO] Using ECJ batch to parse 22 Main java source files with batch size 189 KB.
[INFO] Starting batch processing.
[INFO] The Java analyzer cannot skip unchanged files in this context. A full analysis is performed for all files.
[INFO] 100% analyzed
[INFO] Batch processing: Done.
[INFO] Did not optimize analysis for any files, performed a full analysis for all 22 files.
[WARNING] Use of preview features have been detected during analysis. Enable DEBUG mode to see them.
[INFO] Using ECJ batch to parse 2 Test java source files with batch size 189 KB.
[INFO] Starting batch processing.
[INFO] 100% analyzed
[INFO] Batch processing: Done.
[INFO] Did not optimize analysis for any files, performed a full analysis for all 2 files.
[INFO] No "Generated" source files to scan.
[INFO] Sensor JavaSensor [java] (done) | time=2181ms
[INFO] Sensor JaCoCo XML Report Importer [jacoco]
[INFO] 'sonar.coverage.jacoco.xmlReportPaths' is not defined. Using default locations: target/site/jacoco/jacoco.xml,target/site/jacoco-it/jacoco.xml,build/reports/jacoco/test/jacocoTestReport.xml
[INFO] No report imported, no coverage information will be imported by JaCoCo XML Report Importer
[INFO] Sensor JaCoCo XML Report Importer [jacoco] (done) | time=2ms
[INFO] Sensor CSS Rules [javascript]
[INFO] No CSS, PHP, HTML or VueJS files are found in the project. CSS analysis is skipped.
[INFO] Sensor CSS Rules [javascript] (done) | time=0ms
[INFO] Sensor C# Project Type Information [csharp]
[INFO] Sensor C# Project Type Information [csharp] (done) | time=1ms
[INFO] Sensor C# Analysis Log [csharp]
[INFO] Sensor C# Analysis Log [csharp] (done) | time=11ms
[INFO] Sensor C# Properties [csharp]
[INFO] Sensor C# Properties [csharp] (done) | time=0ms
[INFO] Sensor SurefireSensor [java]
[INFO] parsing [/home/jenkins/agent/workspace/edu_backend_sonar/target/surefire-reports]
[INFO] Sensor SurefireSensor [java] (done) | time=55ms
[INFO] Sensor HTML [web]
[INFO] Sensor HTML [web] (done) | time=2ms
[INFO] Sensor XML Sensor [xml]
[INFO] 1 source file to be analyzed
[INFO] 1/1 source file has been analyzed
[INFO] Sensor XML Sensor [xml] (done) | time=217ms
[INFO] Sensor TextAndSecretsSensor [text]
[INFO] 25 source files to be analyzed
[INFO] 25/25 source files have been analyzed
[INFO] Sensor TextAndSecretsSensor [text] (done) | time=42ms
[INFO] Sensor VB.NET Project Type Information [vbnet]
[INFO] Sensor VB.NET Project Type Information [vbnet] (done) | time=1ms
[INFO] Sensor VB.NET Analysis Log [vbnet]
[INFO] Sensor VB.NET Analysis Log [vbnet] (done) | time=11ms
[INFO] Sensor VB.NET Properties [vbnet]
[INFO] Sensor VB.NET Properties [vbnet] (done) | time=0ms
[INFO] Sensor IaC Docker Sensor [iac]
[INFO] 0 source files to be analyzed
[INFO] 0/0 source files have been analyzed
[INFO] Sensor IaC Docker Sensor [iac] (done) | time=47ms
[INFO] ------------- Run sensors on project
[INFO] Sensor Analysis Warnings import [csharp]
[INFO] Sensor Analysis Warnings import [csharp] (done) | time=0ms
[INFO] Sensor Zero Coverage Sensor
[INFO] Sensor Zero Coverage Sensor (done) | time=9ms
[INFO] Sensor Java CPD Block Indexer
[INFO] Sensor Java CPD Block Indexer (done) | time=46ms
[INFO] SCM Publisher SCM provider for this project is: git
[INFO] SCM Publisher 25 source files to be analyzed
[INFO] SCM Publisher 25/25 source files have been analyzed (done) | time=322ms
[INFO] CPD Executor 10 files had no CPD blocks
[INFO] CPD Executor Calculating CPD for 12 files
[INFO] CPD Executor CPD calculation finished (done) | time=7ms
[INFO] Analysis report generated in 50ms, dir size=217.6 kB
[INFO] Analysis report compressed in 34ms, zip size=79.5 kB
[INFO] Analysis report uploaded in 53ms
[INFO] ANALYSIS SUCCESSFUL, you can find the results at: http://211.34.231.84:30262/dashboard?id=edu
[INFO] Note that you will be able to access the updated dashboard once the server has processed the submitted analysis report
[INFO] More about the report processing at http://211.34.231.84:30262/api/ce/task?id=AYchJdLGw_Vp_k2kP2ZW
[INFO] Analysis total time: 7.425 s
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  51.927 s
[INFO] Finished at: 2023-03-27T12:38:59+09:00
[INFO] ------------------------------------------------------------------------
```  


<br/>

SonarQube로 이동하여 데이터를 확인해 봅니다.    
Thirdproject 라는 이름으로 하나가 생성된 것을 볼수 있습니다.  


<img src="./assets/sonarqube_jenkins6.png" style="width: 80%; height: auto;"> 

<br/>

프로젝트를 클릭하여 좀 더 자세한 정보를 볼 수 있습니다.  

<img src="./assets/sonarqube_jenkins5.png" style="width: 80%; height: auto;">  


<br/>

Jenkins Pipeline은 아래와 같습니다.   
기존 backend pipeline 에 SonarQube가 추가가 되었습니다. 

<br/>

```bash
        ...
        stage('SonarQube Analysis') {
           container('build-tools') {
               withSonarQubeEnv('sonarqube'){ // 시스템설정 값
                 sh "./mvnw clean verify sonar:sonar -Dsonar.projectKey=edu"
             }
           }  
        }
        ...
```  

<br/>

전체 Pipeline 입니다.

<br/>

```bash
def label = "agent-${UUID.randomUUID().toString()}"
def gitBranch = 'master'
def docker_registry = "ghcr.io"  
def imageName = "ghcr.io/shclub/edu12-backend"
def fromImage = "ghcr.io/shclub/jre17-runtime:v1.0.0"
def git_ops_name = "edu12-backend-gitops"
def P_NAMESPACE = "edu30"

def TAG = getTag(gitBranch)

podTemplate(label: label, serviceAccount: 'jenkins-admin', namespace: P_NAMESPACE,
    containers: [
        containerTemplate(name: 'build-tools', image: 'ghcr.io/shclub/build-tool:v1.0.0', ttyEnabled: true, command: 'cat', privileged: true, alwaysPullImage: true)
        ,containerTemplate(name: 'jnlp', image: 'ghcr.io/shclub/jenkins/jnlp-slave:latest-jdk11', args: '${computer.jnlpmac} ${computer.name}')
    ],
    volumes: [
        hostPathVolume(hostPath: '/etc/containers' , mountPath: '/var/lib/containers' ),
        persistentVolumeClaim(mountPath: '/var/jenkins_home', claimName: 'jenkins-edu-slave-pvc',readOnly: false)
        ]){    
  
    
    node(label) { 
        stage('SCM') {
           checkout scm
       }  
       
        stage('SonarQube Analysis') {
           container('build-tools') {
               withSonarQubeEnv('sonarqube'){ // 시스템설정 값
                 sh "./mvnw clean verify sonar:sonar -Dsonar.projectKey=edu"
             }
           }  
        }
        
       stage('Maven Build & Image Push ') {
            container('build-tools') {
               withCredentials([usernamePassword(credentialsId: 'github_ci',usernameVariable: 'USERNAME',passwordVariable: 'PASSWORD')]) {
                    sh  """
                         ./mvnw clean package jib:build  -Dmaven.test.skip=true  \
                         -Djib.from.image=${fromImage} \
                         -Djib.from.auth.username=${USERNAME} \
                         -Djib.from.auth.password=${PASSWORD} \
                         -Djib.to.image=${imageName} \
                         -Djib.to.tags=${TAG}  \
                         -Djib.to.auth.username=${USERNAME} \
                         -Djib.to.auth.password=${PASSWORD}
                         echo 'TAG ==========> ' ${TAG}
                   """
              }
            }
        }

        stage('GitOps update') {
            container('build-tools') {
               withCredentials([usernamePassword(credentialsId: 'github_ci',usernameVariable: 'USERNAME',passwordVariable: 'PASSWORD')]) {
                    sh """  
                        cd ~
                        git clone https://${USERNAME}:${PASSWORD}@github.com/${USERNAME}/${git_ops_name}
                        cd ${git_ops_name}
                        git checkout HEAD
                        kustomize edit set image ${imageName}:${TAG}
                        git config --global user.email "shclub@gmail.com"
                        git config --global user.name ${USERNAME}
                        git add .
                        git commit -am 'update image tag  ${TAG} from My_Jenkins'
                        cat kustomization.yaml
                        git push origin HEAD
                    """
               } 
            }
        }
        
        
    }
}

def getTag(branchName){     
    def TAG
    def DATETIME_TAG = new Date().format('yyyyMMddHHmmss')
    TAG = "${DATETIME_TAG}"
    return TAG
}  
```