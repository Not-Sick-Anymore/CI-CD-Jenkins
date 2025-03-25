<img src="https://capsule-render.vercel.app/api?type=waving&color=00C3FF&height=150&section=header" width="1000" />

<div align="center">
<h1 style="font-size: 36px;">🚀 Jenkins를 활용한 CI/CD 자동화 프로젝트</h1>
</div>
</br>

## 목차
1. [🙆🏻‍♂️ 팀원](#%EF%B8%8F-팀원)
2. [📌 프로젝트 개요](#-프로젝트-개요)
3. [🚩프로젝트별 Architecture](#프로젝트별-architecture)
4. [🛠 프로젝트 수행 과정](#-프로젝트-수행-과정)
5. [🧨 Trouble-Shooting](#-trouble-shooting)

## 🙆🏻‍♂️ 팀원

#### 팀명 : 아프지말아조
우리FISA 4기 클라우드 엔지니어링 아프지말아조팀

|<img src="https://avatars.githubusercontent.com/u/150774446?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/55776421?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/179544856?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/129985846?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|김예진<br/>[@yeejkim](https://github.com/yeejkim)|이슬기<br/>[@seulg2027](https://github.com/seulg2027)|이은준<br/>[@2EunJun](https://github.com/2EunJun)|정파란<br/>[@BlueRedOrange](https://github.com/BlueRedOrange)|


## **📌 프로젝트 개요**

- **1단계:** 개인 로컬 시스템에서 Jenkins를 활용한 **CI/CD 자동화**
- **2단계:** 빌드된 애플리케이션을 **다른 로컬 시스템으로 이관 및 실행**
- **사용 도구:** `Jenkins`, `GitHub`, `Ngrok`, `Java`
- **배포 환경:** `Ubuntu 22.04`

## 🚩프로젝트별 Architecture

![image](https://github.com/user-attachments/assets/673e991f-4ceb-4efa-af7d-9108bdd572fa)
![image](https://github.com/user-attachments/assets/7cb26f0d-7350-4755-b7df-dc41303a8f29)


## **🛠 프로젝트 수행 과정**

### ✨ 사전 준비 사항

- **GitHub Webhook을 활용한 Jenkins 자동 빌드 설정**
    1. 로컬 개발 환경에서 Jenkins가 외부에서 접근 가능하도록 ngrok 설치 및 실행
    2. GitHub 리포지토리 설정 > Webhook 설정 값 입력 
    3. Jenkins에서 GitHub Webhook 플러그인 설치

![image](https://github.com/user-attachments/assets/39c4e082-604d-4829-a617-668889ba1ce8)

*🎆 **Webhook**이란?* 

- *특정 이벤트 발생 시 외부 URL로 자동으로 POST 요청을 보내는 기능*
- *이를 활용하여 **GitHub에서 코드 변경이 있을 때 자동으로 Jenkins 빌드를 트리거(trigger)**하도록 설정*
<br><br>

### **📍1단계: 개인 서버에서 CI/CD 실행**

✅ **CI 과정 (Continuous Integration)**

1. GitHub에서 최신 코드 Pull (`git pull`)
2. Gradle을 사용하여 JAR 빌드 (`./gradlew clean build -x test`)
3. 빌드된 JAR 파일을 **`bind` 폴더에 저장**

✅ **CD 과정 (Continuous Deployment)**

- Jenkins가 빌드된 JAR 파일을 실행

<details>
  <summary>📌 개인 로컬의 서로 다른 Ubuntu 서버로 jar 파일 전송하는 groovy 코드</summary>

  ```groovy
pipeline {
    agent any

    environment {
        GIT_REPO = <GITHUB ADDRESS>
        BRANCH = "main"

        // 빌드 결과를 저장할 로컬 경로
        BUILD_DIR = "step07_cicd/bind"
        JAR_NAME = "myapp.jar"

        // 배포 대상 서버 정보
        REMOTE_SERVER = "ubuntu@<IP ADDRESS>"
        REMOTE_DIR = "/home/ubuntu"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "📥 GitHub에서 코드 가져오기"
                git branch: "${BRANCH}", url: "${GIT_REPO}"
            }
        }

        stage('Build with Gradle') {
    steps {
        echo "🔧 Gradle로 JAR 빌드"
        sh '''
            cd step07_cicd
            chmod +x gradlew
            ./gradlew clean build -x test
            mkdir -p bind
            JAR_PATH=$(ls build/libs/*SNAPSHOT.jar | grep -v 'plain')
            cp "$JAR_PATH" bind/myapp.jar
        '''
    }
}

        stage('Deploy to Server') {
            steps {
                echo "📦 myserver02로 JAR 전송"
                sh '''
                    scp -o StrictHostKeyChecking=no step07_cicd/bind/${JAR_NAME} ${REMOTE_SERVER}:${REMOTE_DIR}/
                '''
            }
        }

        stage('Run on Server') {
    steps {
        echo "🚀 myserver02에서 애플리케이션 실행"
        sh '''
            ssh -o StrictHostKeyChecking=no ${REMOTE_SERVER} "pkill -f ${JAR_NAME} || true && nohup java -jar ${REMOTE_DIR}/${JAR_NAME} > ${REMOTE_DIR}/app.log 2>&1 &"
        '''
    }
}

    }

    post {
        success {
            echo "✅ 파이프라인 성공적으로 완료!"
        }
        failure {
            echo "❌ 파이프라인 실패!"
        }
    }
}
```
</details>         


### **📍2단계 : Build된 어플리케이션을 다른 로컬의 Ubuntu로 이관 및 실행**

네트워크 연결을 설정하여 다른 서버에게 파일을 보낸다. 이 때, Bridge로 연결하거나 NAT로 연결한 환경에서 실습을 진행한다.

✅ **Bridge로 연결**

1. Bridge로 연결 <br>
  ![image](https://github.com/user-attachments/assets/8c2af9d2-f0db-4f21-a066-f77b7ac07697)

2. 연결후 ip 확인 <br>
   ![image](https://github.com/user-attachments/assets/8e4fdf06-a741-49c3-aa88-bc1627f46362)

3. SSH RSA 키 추가해서 원격 서버에 비밀번호 없이 접속할 수 있도록 설정 <br>
  ![image](https://github.com/user-attachments/assets/7721ac4e-075e-4580-9cb1-b3d952aa59f9)

    <details>
      <summary>A 컴퓨터 → B 컴퓨터 Ubuntu에 jar 파일 전송하는 groovy 코드</summary>
      
      ```groovy
      pipeline {
          agent any
      
          environment {
              GIT_REPO = "https://github.com/Not-Sick-Anymore/CI-CD-Jenkins.git"
              BRANCH = "main"
      
              // 빌드 결과를 저장할 로컬 경로 (Jenkins 유저가 접근 가능한 위치로 설정)
              BUILD_DIR = "step07_cicd/bind"
              JAR_NAME = "myapp.jar"
      
              // 배포 대상 서버 정보
              REMOTE_SERVER = "ubuntu@192.168.0.20"
              REMOTE_DIR = "/home/ubuntu"
          }
      
          stages {
              stage('Checkout') {
                  steps {
                      echo "📥 GitHub에서 코드 가져오기"
                      git branch: "${BRANCH}", url: "${GIT_REPO}"
                  }
              }
      
              stage('Build with Gradle') {
          steps {
              echo "🔧 Gradle로 JAR 빌드"
              sh '''
                  cd step07_cicd
                  chmod +x gradlew
                  ./gradlew clean build -x test
                  mkdir -p bind
                  JAR_PATH=$(ls build/libs/*SNAPSHOT.jar | grep -v 'plain')
                  cp "$JAR_PATH" bind/myapp.jar
              '''
          }
      }
      
              stage('Deploy to Server') {
                  steps {
                      echo "📦 myserver02로 JAR 전송"
                      sh '''
                          scp -o StrictHostKeyChecking=no step07_cicd/bind/${JAR_NAME} ${REMOTE_SERVER}:${REMOTE_DIR}/
                      '''
                  }
              }
      
              stage('Run on Server') {
          steps {
              echo "🚀 myserver02에서 애플리케이션 실행"
              sh '''
                  ssh ${REMOTE_SERVER} "
          nohup /usr/bin/java -jar ${REMOTE_DIR}/${JAR_NAME} > ${REMOTE_DIR}/app.log 2>&1 & disown
      "
      
                  
              '''
          }
      }
      
          }
      
          post {
              success {
                  echo "✅ 파이프라인 성공적으로 완료!"
              }
              failure {
                  echo "❌ 파이프라인 실패!"
              }
          }
      }
      
      ```
      </details>




4. jar 파일 복사 여부 확인
   
```groovy
  ls -l /home/ubuntu/myapp.jar
```

5. jar 파일 실행 여부 확인

```Bash
  curl http://localhost:8081
```



✅ **NAT로 연결**

1. 포트 포워딩 추가
    
    A 컴퓨터에서 B컴퓨터(192.168.0.157)의 22번 포트와 연결 (30000번 포트)
    ![image](https://github.com/user-attachments/assets/6e272d17-38b3-4707-948f-2a8eb8aa4498)
    ![image](https://github.com/user-attachments/assets/a24dcb53-f524-4ca8-9a68-8c258f6f4713)

2. SSH RSA 키 추가해서 원격 서버에 비밀번호 없이 접속할 수 있도록 설정
    ![image](https://github.com/user-attachments/assets/0cf287df-fb06-4340-b8d6-0d84438ce996)
    ![image](https://github.com/user-attachments/assets/29e0ffec-2fad-4217-b903-bd3555482b27)


<details>
  <summary>A 컴퓨터 → B 컴퓨터 Ubuntu에 jar 파일 전송하는 groovy 코드</summary>

  ```groovy
  pipeline {
      agent any
  
      environment {
          GIT_REPO = "https://github.com/Not-Sick-Anymore/CI-CD-Jenkins.git"
          BRANCH = "main"
  
          // 빌드 결과를 저장할 로컬 경로 (Jenkins 유저가 접근 가능한 위치로 설정)
          BUILD_DIR = "step07_cicd/bind"
          JAR_NAME = "myapp.jar"
  
          // 배포 대상 서버 정보
          REMOTE_SERVER = "ubuntu@192.168.0.157"
          REMOTE_DIR = "/home/ubuntu"
      }
  
      stages {
          stage('Checkout') {
              steps {
                  echo "📥 GitHub에서 코드 가져오기"
                  git branch: "${BRANCH}", url: "${GIT_REPO}"
              }
          }
  
          stage('Build with Gradle') {
              steps {
                  echo "🔧 Gradle로 JAR 빌드"
                  sh '''
                      cd step07_cicd
                      chmod +x gradlew
                      ./gradlew clean build -x test
                      mkdir -p bind
                      JAR_PATH=$(ls build/libs/*SNAPSHOT.jar | grep -v 'plain')
                      cp "$JAR_PATH" bind/myapp.jar
                  '''
              }
          }
  
          stage('Deploy to Server') {
              steps {
                  echo "📦 예진이 서버로 JAR 전송"
                  sh '''
                      scp -P 30000 -o StrictHostKeyChecking=no step07_cicd/bind/${JAR_NAME} ${REMOTE_SERVER}:${REMOTE_DIR}/
                  '''
              }
          }
      }
  
      post {
          success {
              echo "✅ 파이프라인 성공적으로 완료!"
          }
          failure {
              echo "❌ 파이프라인 실패!"
          }
      }
  }
  ```
  </details>



### 🧨 Trouble-Shooting

#### 원격 서버 SSH로 접속 시 권한 문제

- 문제 상황
    - 1단계(개인 로컬에서 다른 서버에 배포) 과정에서 배포하는 과정에서, 다른 서버에 ‘Permission Denied’ 권한 문제 발생
    
- 해결 방법: **Jenkins 서버에서 SSH Key 생성 및 SSH 공개 키 추가**
  
  1. Jenkins 서버에서 SSH key 생성

  ```bash
    # Jenkins 서버 진입
    $docker exec -it <컨테이너이름> bash
    $cd /var/jenkins_home
      
    # SSH 키 생성
    $ssh-keygen -t rsa
  ```

  2. 원격 서버에 SSH 공개 키 추가

  ```bash
    $ssh-copy-id <USERNAME>@<IP ADDRESS>
  ```
     
        
#### VMWare에서 외부 네트워크로 연결이 되지 않는 문제

**◼️기기 환경 사양**
- OS : Ubuntu 24.02
- Virtual Machine : VMWare Workstation Pro 17

- 문제 상황
    - 내부 네트워크에서만 통신이 가능하여, Ubuntu Package 업데이트가 불가능하였다.
    - `ping 8.8.8.8` 을 날렸을 때 *unreachable 오류가 발생*했다.
      ![image](https://github.com/user-attachments/assets/ec2db2b8-2fc6-430f-a6f8-942314ce40b1)


- 해결 방법
    1. `/etc/netplan/01-netcfg.yaml` 파일 수정해서 기본 게이트웨이 주소 변경해주기 <br>
        → 게이트웨이가 고정 IP의 네트워크에 포함된 IP여야 함

    ![image](https://github.com/user-attachments/assets/bf376c91-17e7-47c9-a74a-5bc4ef3b5b2f)
    ![image](https://github.com/user-attachments/assets/31aee626-424c-442d-a1ed-ebb120104876)


    2. network netplan을 적용시키기

       
    ```bash
      $network netplan apply
    ```

    3. ip route 현황 확인, ping으로 8.8.8.8 외부 통신 확인
       ![image](https://github.com/user-attachments/assets/1b6e5f54-726a-418c-be93-57c7f016ae2b)
       ![image](https://github.com/user-attachments/assets/377d2219-3907-492a-a88d-fb87e695fd1b)











  
