# Vid2Aud
mp4 비디오를 mps로 변환하는 마이크로서비스 프로젝트.

## Architecture

<p align="center">
  <img src="./Project documentation/ProjectArchitecture.png" width="600" title="Architecture" alt="Architecture">
  </p>


## Python 기반의 마이크로서비스 어플리케이션을 AWS EKS로 배포.

### 소개

이 문서는 AWS Elastic Kubernetes Service(EKS)에서 Python 기반 마이크로서비스 애플리케이션을 배포하는 단계별 가이드를 제공합니다. 애플리케이션은 다음과 같은 네 가지 주요 마이크로서비스로 구성됩니다: `auth-server`, `converter-module`, `database-server`(PostgreSQL 및 MongoDB), `notification-server`.

---

### 사전 준비 사항

시작하기 전에 다음 사전 조건을 충족했는지 확인하세요:

1. **AWS 계정 생성:** AWS 계정이 없다면 [여기](https://docs.aws.amazon.com/streams/latest/dev/setting-up.html)를 따라 계정을 생성하세요.

2. **Helm 설치:** Helm은 Kubernetes 패키지 관리 도구입니다. [여기](https://helm.sh/docs/intro/install/)를 참고하여 Helm을 설치하세요.

3. **Python:** Python이 시스템에 설치되어 있는지 확인하세요. [Python 공식 웹사이트](https://www.python.org/downloads/)에서 다운로드할 수 있습니다.

4. **AWS CLI:** AWS 명령줄 인터페이스(CLI)를 설치하세요. 공식 [설치 가이드](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)를 참고하세요.

5. **kubectl 설치:** 최신 안정 버전의 `kubectl`을 시스템에 설치하세요. 설치 방법은 [여기](https://kubernetes.io/docs/tasks/tools/)에서 확인할 수 있습니다.

6. **데이터베이스:** 애플리케이션에 필요한 PostgreSQL과 MongoDB를 설정하세요.

---

### 애플리케이션 배포의 주요 흐름

다음 단계에 따라 마이크로서비스 애플리케이션을 배포:

1. **MongoDB 및 PostgreSQL 설정:** 데이터베이스를 생성하고 자동 연결을 활성화.

2. **RabbitMQ 배포:** 메시지 큐잉을 위해 RabbitMQ를 배포합니다. 이는 `converter-module`에서 필요.

3. **RabbitMQ 큐 생성:** `converter-module`을 배포하기 전에 RabbitMQ에 두 개의 큐를 생성: `mp3`와 `video`.

4. **마이크로서비스 배포:**
   - **auth-server:** `auth-server` 매니페스트 폴더로 이동하여 구성 파일을 적용.
   - **gateway-server:** `gateway-server`를 배포합니다.
   - **converter-module:** `converter-module`을 배포합니다. `converter/manifest/secret.yaml` 파일에 이메일과 비밀번호를 제공해야함.
   - **notification-server:** 알림 및 2단계 인증(2FA)을 위해 이메일을 구성.

5. **애플리케이션 유효성 검사:** 모든 구성 요소의 상태를 다음 명령어로 확인:
   ```bash
   kubectl get all
   ```

6. **인프라 파기:** 설정된 모든 인프라를 삭제.

### Low Level Steps

#### Cluster Creation




<!-- 1. **Log in to AWS Console:**
   - Access the AWS Management Console with your AWS account credentials.

2. **Create eksCluster IAM Role**
   - Follow the steps mentioned in [this](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html) documentation using root user
   - After creating it will look like this:

   <p align="center">
  <img src="./Project documentation/ekscluster_role.png" width="600" title="ekscluster_role" alt="ekscluster_role">
  </p>

   - Please attach `AmazonEKS_CNI_Policy` explicitly if it is not attached by default

3. **Create Node Role - AmazonEKSNodeRole**
   - Follow the steps mentioned in [this](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role) documentation using root user
   - Please note that you do NOT need to configure any VPC CNI policy mentioned after step 5.e under Creating the Amazon EKS node IAM role
   - Simply attach the following policies to your role once you have created `AmazonEKS_CNI_Policy` , `AmazonEBSCSIDriverPolicy` , `AmazonEC2ContainerRegistryReadOnly`
     incase it is not attached by default
   - Your AmazonEKSNodeRole will look like this: 

<p align="center">
  <img src="./Project documentation/node_iam.png" width="600" title="Node_IAM" alt="Node_IAM">
  </p>

4. **Open EKS Dashboard:**
   - Navigate to the Amazon EKS service from the AWS Console dashboard.

5. **Create EKS Cluster:**
   - Click "Create cluster."
   - Choose a name for your cluster.
   - Configure networking settings (VPC, subnets).
   - Choose the `eksCluster` IAM role that was created above
   - Review and create the cluster.

6. **Cluster Creation:**
   - Wait for the cluster to provision, which may take several minutes.

7. **Cluster Ready:**
   - Once the cluster status shows as "Active," you can now create node groups. -->

#### 노드 그룹 생성

1. **Compute** 섹션에서 **"Add node group"** 버튼클릭.

2. AMI(기본값), 인스턴스 유형(e.g., `t3.medium`), 노드 수를 선택.

3. **"Create node group"** 버튼을 클릭하여 노드 그룹을 생성.

---

#### 노드 보안 그룹의 인바운드 규칙 추가

**주의:** 노드 보안 그룹에서 필요한 모든 포트가 열려 있는지 확인.

<p align="center">
  <img src="./Project documentation/inbound_rules_sg.png" width="600" title="Inbound_rules_sg" alt="Inbound_rules_sg">
</p>

---

#### EBS CSI 애드온 활성화

1. `ebs csi` 애드온을 활성화합니다. 클러스터가 생성된 후 PVCs(Persistent Volume Claims)를 활성화하기 위해 필요합니다.

<p align="center">
  <img src="./Project documentation/ebs_addon.png" width="600" title="ebs_addon" alt="ebs_addon">
</p>

#### EKS 클러스터에 애플리케이션 배포하기

1. 코드 리포지토리를 클론  
2. 클러스터 컨텍스트를 설정  
   ```bash
   aws eks update-kubeconfig --name <cluster_name> --region <aws_region>
   ```

### Commands

Here are some essential Kubernetes commands for managing your deployment:


### MongoDB

MongoDB 설치하려면 `values.yaml` 파일에 데이터베이스 사용자 이름과 비밀번호를 설정한 다음, MongoDB Helm 차트 폴더로 이동해서 다음 명령어 실행

```
cd Helm_charts/MongoDB
helm install mongo .
```

MongoDB 인스턴스에 접속하려면 다음 명령어 사용

```
mongosh mongodb://<username>:<pwd>@<nodeip>:30005/mp3s?authSource=admin
```

### PostgreSQL

`values.yaml` 파일에 데이터베이스 사용자 이름과 비밀번호 설정. PostgreSQL Helm 차트 폴더에서 PostgreSQL 설치하고 `init.sql` 파일의 쿼리로 초기화. PowerShell 사용자라면 아래 명령어 실행

```
cd ..
cd Postgres
helm install postgres .
```

Postgres 데이터베이스에 접속하고 "init.sql" 파일의 모든 쿼리 복사해서 실행

```
psql 'postgres://<username>:<pwd>@<nodeip>:30003/authdb'
```


### RabbitMQ

RabbitMQ 배포하려면 다음 명령어 실행

```
helm install rabbitmq .
```

`mp3`와 `video`라는 두 개의 큐가 RabbitMQ에 생성되어 있는지 확인. 큐를 만들려면 `<nodeIp>:30004>`에 접속하고 기본 사용자 이름 `guest`, 비밀번호 `guest`로 로그인.
**NOTE:** Ensure that all the necessary ports are open in the node security group.

### 마이크로서비스 매니페스트 파일 적용

- **Auth Service:**
  ```bash
  cd auth-service/manifest
  kubectl apply -f .
  ```

- **Gateway Service:**
  ```bash
  cd gateway-service/manifest
  kubectl apply -f .
  ```

- **Converter Service:**
  ```bash
  cd converter-service/manifest
  kubectl apply -f .
  ```

- **Notification Service:**
  ```bash
  cd notification-service/manifest
  kubectl apply -f .
  ```

---

### 애플리케이션 유효성 검사

마이크로서비스 배포 후 모든 구성 요소 상태를 확인하려면 다음 명령어 실행

```bash
kubectl get all
```

---

### 알림 구성

이메일 알림과 2단계 인증(2FA)을 구성하려면 다음 단계 따라

1. Gmail 계정으로 이동해서 프로필 클릭  
2. **"Google 계정 관리"** 클릭  
3. 왼쪽 메뉴에서 **"보안"** 탭 선택  
4. **"2단계 인증"** 활성화  
5. 애플리케이션 전용 비밀번호를 찾아서 설정에서 확인  
6. **"기타"** 클릭하고 이름 입력  
7. **"생성"** 클릭해서 생성된 비밀번호 복사  
8. 이 비밀번호를 `notification-service/manifest/secret.yaml` 파일에 이메일과 함께 붙여넣기  

---

### 애플리케이션 실행

다음 API 호출로 애플리케이션 실행

---

#### API 정의

- **로그인 엔드포인트**
  ```http
  POST http://nodeIP:30002/login
  ```
  ```bash
  curl -X POST http://nodeIP:30002/login -u <email>:<password>
  ```
  **예상 결과:** 성공!

---

- **업로드 엔드포인트**
  ```http
  POST http://nodeIP:30002/upload
  ```
  ```bash
  curl -X POST -F 'file=@./video.mp4' -H 'Authorization: Bearer <JWT Token>' http://nodeIP:30002/upload
  ```
  **확인:** 이메일로 ID 수신 여부 확인  

---

- **다운로드 엔드포인트**
  ```http
  GET http://nodeIP:30002/download?fid=<Generated file identifier>
  ```
  ```bash
  curl --output video.mp3 -X GET -H 'Authorization: Bearer <JWT Token>' "http://nodeIP:30002/download?fid=<Generated fid>"
  ```  
  **확인:** 변환된 파일(mp3)이 성공적으로 다운로드되는지 확인

## 인프라 삭제

인프라 정리하려면 다음 단계 따라

1. **노드 그룹 삭제:** EKS 클러스터와 연결된 노드 그룹 삭제  

2. **EKS 클러스터 삭제:** 노드가 모두 삭제된 후 EKS 클러스터 자체를 삭제  
