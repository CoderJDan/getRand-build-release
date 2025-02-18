![스크린샷 2025-02-14 180825](https://github.com/user-attachments/assets/924978a4-6ac5-4342-85aa-7e77a0011c61)# 📌 미미-노트북 대여 시스템: 빌드 및 배포 문서

![Mimi Laptop Rental](https://img.shields.io/badge/Mimi-Laptop%20Rental-blue.svg)

## 📝 개요

getRand() {}; 는 MSA를 적용한 데이터 조회 및 통계 플랫폼입니다. 해당 플랫폼의 CI/CD를 구현하여 실제 서비스 배포와 같은 환경을 구축하였습니다.

---

## 🚀 빌드 및 배포 개요

### 📌 사용 기술 및 도구

- **운영체제:** Ubuntu (가상 서버)
- **언어:** Java (Spring Boot)
- **빌드 도구:** Gradle
- **컨테이너화:** Docker
- **CI/CD 도구:** GitHub Actions, Jenkins, ArgoCD
- **배포 인프라:** Kubernetes
- **데이터베이스:** MySQL

### 🔄 CI/CD 개요

1. **GitHub**: 소스 코드 관리 및 CI/CD 트리거
2. **Jenkins**: 자동화된 빌드 및 Docker 이미지 생성, Docker Hub에 푸시
3. **Docker Hub**: 빌드된 이미지를 저장하고 Kubernetes에서 사용
4. **ArgoCD**: Kubernetes 클러스터에 자동 배포

---


## 🏗️ 빌드 프로세스
### Git Push
![스크린샷 2025-02-14 190013](https://github.com/build_release_screenshot/jenkins_github_push_trigger.png)


![스크린샷 2025-02-14 190131](https://github.com/user-attachments/assets/928b4f61-4284-4ee7-b839-9b6c99bfc794)


### 📂 Dockerfile을 이용한 빌드
![스크린샷 2025-02-14 182150](https://github.com/user-attachments/assets/c8cb543f-c897-46ed-8608-9926e9cc5e47)

```Dockerfile
# Build Stage
FROM gradle:8.11.1-jdk17 AS build

# 작업 디렉토리 생성
WORKDIR /myapp

# 프로젝트 전체 파일을 복사
COPY . /myapp

# Gradle 실행 권한 추가
RUN chmod +x /myapp/gradlew

# Gradle 빌드 실행 (테스트 제외)
RUN /myapp/gradlew clean build --no-daemon -x test

# Run Stage
FROM openjdk:17-alpine

# 작업 디렉토리 생성
WORKDIR /myapp

# 빌드된 JAR 파일 복사
COPY --from=build /myapp/build/libs/*SNAPSHOT.jar /myapp/getrand.jar

# 애플리케이션 실행 포트
EXPOSE 5679

# 애플리케이션 실행 명령어
ENTRYPOINT ["java", "-jar", "/myapp/getrand.jar"]
```

---

## ☸️ Kubernetes 배포

### 📜 배포 리소스 정의 (`mimi-app-service.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rental
  namespace: mimiproject
spec:
  replicas: 1
  selector:
    matchLabels:
      project: mimiuser
  template:
    metadata:
      labels:
        project: mimiuser
    spec:
      containers:
        - name: rental
          image: daul0519/mimiuser:v1.3
          ports:
            - containerPort: 5678
          env:
            - name: DB_HOST
              value: "mysql-service"
            - name: DB_NAME
              value: "mimi"
            - name: DB_USER
              value: "mytest"
            - name: DB_PASSWORD
              value: "1234"
            - name: SPRING_DATASOURCE_URL
              value: "jdbc:mysql://10.104.200.22:3306/mimi"
```

### 🌐 Ingress 설정 (`ingress-setting.yaml`)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-setting
  namespace: mimiproject
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - backend:
              service:
                name: mimi-user
                port:
                  number: 5678
            path: /rental
            pathType: Prefix
```

---

## ⚙️ Jenkins CI/CD 파이프라인 (`Jenkinsfile`)

```groovy
pipeline{
    agent any

    environment{
        APP_REPO_URL = 'https://github.com/CoderJDan/getRand_datacollectionservice.git'
        GITHUB_CREDENTIAL_ID = 'github_hook_id'
        DOCKERHUB_CREDENTIAL_ID = 'docker-hub-access'
        DOCKERHUB_REPOSITORY_IMAGE = 'jangdaniel/getrand-datacollection-service'
        DOCKERHUB_TAG = "v1.0.${env.BUILD_NUMBER}"
    }
    stages{
        stage("git clone"){
            steps{
                git branch: 'develop', credentialsId: 'github_hook_id', url: 'https://github.com/CoderJDan/getRand_datacollectionservice.git'
            }
        }
        stage("docker build"){
            steps{
                sh 'docker build -t $DOCKERHUB_REPOSITORY_IMAGE:$DOCKERHUB_TAG .'
            }
        }
        stage("docker push"){
            steps{
                script{
                    docker.withRegistry('https://index.docker.io/v1/', DOCKERHUB_CREDENTIAL_ID){
                        def myImage = docker.image("$DOCKERHUB_REPOSITORY_IMAGE:$DOCKERHUB_TAG")
                        myImage.push()
                    }
                }
            }
        }
    }
}
```

---
## 🔄ArgoCD : 배포 자동화 
![스크린샷 2025-02-14 180825](https://github.com/user-attachments/assets/03397350-fa81-4741-b460-5172f36a82d7)
![스크린샷 2025-02-14 184412](https://github.com/user-attachments/assets/c5e636de-6e44-44d5-8ebc-698370dfe73b)
![스크린샷 2025-02-14 183025](https://github.com/user-attachments/assets/5f2536fc-886b-408e-b440-9f9d4ce3fd52)


---

## 🔄 GitHub Actions: YAML 파일 동기화

### 📜 `push_yaml_to_repo.yml`
```yaml
name: Push YAML to Another Repo

on:
  push:
    branches:
      - Devops
    paths:
      - "argo/**/*.yaml"
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: 저장소 A 체크아웃
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Git 설정
        run: |
          git config --global user.name "github-actions"
          git config --global user.email "github-actions@github.com"

      - name: 저장소 B (`mimiyaml.git`) 클론
        run: |
          git clone https://x-access-token:${{ secrets.GH_PAT }}@github.com/05Daul/mimiyaml.git repo_b
          cd repo_b
          git checkout daul || git checkout -b daul
          git pull origin daul --rebase

      - name: YAML 파일 복사 및 푸시
        run: |
          mkdir -p repo_b/argo
          cp -r argo/*.yaml repo_b/argo/ || echo "No YAML files to copy"
          cd repo_b
          git add .
          if ! git diff --cached --exit-code; then
            git commit -m "자동 업데이트: 저장소 A에서 YAML 파일 변경됨"
            git push origin daul || (sleep 5 && git push origin daul)
          else
            echo "No changes detected, skipping push."
          fi
```

---

## ✅ 실행 및 검증 방법
```sh
# Kubernetes 배포
kubectl apply -f mimi-app-service.yaml
kubectl apply -f ingress-setting.yaml

# 배포 확인
kubectl get pods -n mimiproject
kubectl get svc -n mimiproject
```

---
## 💻 전체 흐름도

![단락 텍스트](https://github.com/user-attachments/assets/bd473892-2ddf-4b31-995a-7009f9a9ab1c)

---
---

## ✅ 실행 및 검증 방법
```sh
# Kubernetes 배포
kubectl apply -f mimi-app-service.yaml
kubectl apply -f ingress-setting.yaml

# 배포 확인
kubectl get pods -n mimiproject
kubectl get svc -n mimiproject
```

---

## 🔧 트러블슈팅
### 🚨 빌드 오류 해결
- `gradlew: Permission denied`: `chmod +x gradlew` 실행 후 다시 빌드
- `docker: command not found`: Docker 설치 및 실행 여부 확인

### ⚠️ 배포 오류 해결
- Pod CrashLoopBackOff 발생 시 `kubectl describe pod <pod-name>`로 로그 확인
- `kubectl logs <pod-name>` 명령어로 오류 메시지 분석

---

## 🎯 프로젝트 정보
📌 **Users레포지토리:** [GitHub - 미미팀](https://github.com/05Daul/mimiUsers)
📌 **Rental레포지토리:** [GitHub - 미미팀](https://github.com/05Daul/mimiRental)
📌 **Equipment레포지토리:** [GitHub - 미미팀](https://github.com/05Daul/mimiEquipment)
📌 **Menifest레포지토리:** [GitHub - 미미팀](https://github.com/05Daul/mimiyaml)  
📌 **Docker Hub:** [Mimi-User](https://hub.docker.com/r/daul0519/mimi-user)  
📌 **문의:** elre519@네이버  

