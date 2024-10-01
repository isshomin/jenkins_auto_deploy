# Deployment Automation: Core Technologies for Servers 🧲

---
<br>

## 개요 ✒

이 프로젝트는 운영 서버에서의 **배포 자동화**를 위한 핵심 기술을 다룹니다. 현대적인 애플리케이션 개발 환경에서 **빠르고 안전한 배포 프로세스를 구축하는 것**이 매우 중요합니다. 그렇기 때문에 프로젝트 내에서 효율적인 배포 자동화를 위해 필수적인 도구와 기술을 사용하여 **운영 서버의 가용성을 유지하면서 배포 시간을 단축하는 방법**을 제공합니다.

---

<br>

## 실습환경 🏞

```ruby
virtualbox: version 7.0
ubuntu: version 22.04.4 LTS
docker: version 27.3.1
jenkins: version 2.462.2
ngrok: version 3.16.0
```

---

<br>

## 구축환경 설정 ⚙

#### 1️⃣) 컨테이너에서 /var/jenkins_home/appjar 경로에 호스트의 appjardir 디렉터리를 마운트하고, 호스트의 8888 포트로 jenkins의 웹 인터페이스 8080 포트에 접근할 수 있게 설정합니다.

```ruby
docker run --name myjenkins2 --privileged -p 8888:8080 -v $(pwd)/appjardir:/var/jenkins_home/appjar jenkins/jenkins:lts-jdk17
```

#### 2️⃣) ngrok을 실행 후 명령어를 입력하여 로컬환경을 외부에서 접속할 수 있게 터널링을 해줍니다.

```ruby
ngrok http http://localhost:8888
```
![image](https://github.com/user-attachments/assets/8ed9c254-8378-4bf2-96a9-3022aff6f492)
<br>

#### 3️⃣) 배포하고자 하는 github Repository(이하 배포코드)의 settings에 들어가서 ngrok의 URL을 이용하여 webhooks 설정을 해줍니다.

![image](https://github.com/user-attachments/assets/fbf018b3-e0e6-4176-bb76-4e4d89efadbc)
<br>

#### 4️⃣) jenkins에서 새로운 item을 생성하고 파이프라인 스크립트를 작성하고 저장해줍니다.

```ruby
pipeline {
    agent any

    stages {
        //github에서 소스 코드를 클론하는 단계
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/isshomin/jenkins-test.git'
            }
        }
        //프로젝트를 빌드하는 단계
        stage('Build') {
            steps {
                sh 'chmod +x gradlew'                    
                sh './gradlew clean build -x test'
            }
        }
        //빌드된 .jar 파일을 특정 디렉토리로 복사하는 단계
        stage('Copy jar') { 
            steps {
                script {
                    def jarFile = 'build/libs/SpringApp-0.0.1-SNAPSHOT.jar'                   
                    sh "cp ${jarFile} /var/jenkins_home/appjar/"
                }
            }
        }
    }
}
```

#### ❗Github hook trigger for GITScm polling"은 jenkins에서 github와의 연동을 설정할 때 사용하는 트리거 옵션 중 하나입니다. 이 옵션은 github에서의 코드 변경 사항을 jenkins가 감지할 수 있도록 합니다.

![image](https://github.com/user-attachments/assets/e03fc03f-e7db-4762-8732-1a793c0b05c4)

<br>

#### 5️⃣) 배포코드를 수정하여 jenkins에 repository가 제대로 hook 되어 자동으로 빌드되는지 확인합니다.

![image](https://github.com/user-attachments/assets/b11a713f-f2de-420a-b260-d1de0bb8ea9f)
<br>

#### 6️⃣) virtual box의 1번 머신 내 호스트와 컨테이너가 마운트된 디렉토리에 .jar 파일이 제대로 복사되었는지 확인합니다.  

![image](https://github.com/user-attachments/assets/c2a0c24e-e6e9-413d-a29a-3bdc447fd883)
<br>

![image](https://github.com/user-attachments/assets/32dd1dbe-d0b4-4230-b79b-0f04404b7aa3)
<br>

---

<br>

## 머신 간 통신설정 📱

#### 1️⃣) 1번 머신에서 SSH key 생성해줍니다.

```ruby
ssh-keygen -t rsa -b 4096
```

#### 2️⃣) 원격 서버에 SSH 공개 키를 추가해줍니다.

```ruby
ssh-copy-id username@10.0.2.11
```

#### 3️⃣) 1번 머신에서 2번 머신으로 접속을 확인합니다.

```ruby
ssh username@10.0.2.11
```

![image](https://github.com/user-attachments/assets/3d5e38d9-9a33-4a26-b087-debd2e318950)
<br>

---

<br>

## 자동화 스크립트 작성 ⌨

#### 1️⃣) 1번 머신 /appjardir 디렉토리 내에 스크립트를 생성합니다.

**change_code.sh**

```ruby

#!/bin/bash

# JAR 파일 경로 설정
JAR_FILE="./SpringApp-0.0.1-SNAPSHOT.jar"

# 2번 PC 정보
REMOTE_USER="username"       # 2번 PC 사용자 이름
REMOTE_IP="10.0.2.11"        # 2번 PC IP 주소
REMOTE_DIR="/home/username/appjardir2/"  # 2번 PC에서 파일을 복사할 디렉토리
AUTO_RUN_SCRIPT="auto_run.sh"  # 2번 PC에서 실행할 스크립트

# COOLDOWN 중복 실행 방지 대기 시간 (예: 10초)
COOLDOWN=10
LAST_RUN=0

# 파일 수정 감지 및 복사
inotifywait -m -e close_write "$JAR_FILE" |
while read -r directory events filename; do
    CURRENT_TIME=$(date +%s)

    # 마지막 실행 후 지정된 시간이 지났는지 확인
    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
        # JAR 파일을 2번 PC로 복사
        scp "$JAR_FILE" "$REMOTE_USER@$REMOTE_IP:$REMOTE_DIR"
        
        # 복사가 완료되었다는 메시지 출력
        echo "$(date): $filename 파일이 수정되어 $REMOTE_IP로 복사되었습니다."
        
        # 2번 PC에서 auto_run.sh 실행
        ssh "$REMOTE_USER@$REMOTE_IP" "bash $REMOTE_DIR/$AUTO_RUN_SCRIPT"
        echo "$(date): $AUTO_RUN_SCRIPT가 $REMOTE_IP에서 실행되었습니다."

        # 마지막 실행 시간 업데이트
        LAST_RUN=$CURRENT_TIME
    else
        echo "$(date): 쿨다운 기간 중입니다. ($((CURRENT_TIME - LAST_RUN)) 초 경과)"
    fi
done
```

#### ❗inotifywait를 사용하기 위해 inotify-tools을 설치해줍니다.
```ruby
sudo apt update
sudo apt install inotify-tools
```


#### 2️⃣) 2번 머신 /appjardir2 디렉토리 내에 스크립트를 생성합니다.

**auto_run.sh**

```ruby
#!/bin/bash

# Spring Boot 애플리케이션 재시작
# 기존 8080 포트 사용 중인 프로세스 종료
if  lsof -i :8999 > /dev/null; then
  # 8999 포트가 사용 중일 경우 이전 프로세스를 종료
  kill -9 $(lsof -t -i:8999)
  echo '정상적으로 종료되었습니다.'
fi

# 백그라운드에서 새로 실행
# > $DEPLOY_DIR/app.log : 애플리케이션 로그를 app.log 파일에 저장하도록 구성
nohup java -jar /home/username/appjardir2/SpringApp-0.0.1-SNAPSHOT.jar > app.log 2>&1 &

echo "배포완료 및 재 실행됩니다."
```

---

<br>

## 실행 테스트 🎬

#### 1️⃣) 2번 머신에서 실행되는 서버를 window환경에서도 접속할 수 있게 포트포워딩을 해줍니다.

![image](https://github.com/user-attachments/assets/f188eba5-46eb-4740-8935-fb9f39adba42)

#### 2️⃣) 각 스크립트에 실행권한을 부여합니다.

```ruby
#1번 머신
chmod +x change_code.sh

#2번 머신
chmod +x auto_run.sh
```

#### 3️⃣) 1번 머신에서 change_code.sh를 실행시켜줍니다.

![image](https://github.com/user-attachments/assets/c893c0f9-7528-48e7-a21b-4f32778cbcc8)

#### 4️⃣) 배포코드를 수정하여 정상적으로 작동하는지 확인합니다.

**1번 머신(배포 및 테스트서버)** 

![image](https://github.com/user-attachments/assets/9d91f4b8-3f84-4caf-b88f-e9066a65a3ae)



**2번 머신(운영 서버)**

![image](https://github.com/user-attachments/assets/e199304b-9ec8-4a46-8fb8-53929fb2cf82)

#### 5️⃣) linux와 window에서 각 환경에서 url을 확인해줍니다.

**linux 환경**

![image](https://github.com/user-attachments/assets/a39a18db-3979-4326-bd2a-decbe18bc272)

**window 환경**

![image](https://github.com/user-attachments/assets/13d36720-e237-4bdc-9535-30c5c9064402)


---

<br>

## 요약 📩

<br>

---

