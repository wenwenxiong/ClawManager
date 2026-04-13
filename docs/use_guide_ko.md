[<- README 홈으로 돌아가기](../README.ko.md)

# ClawManager 배포 및 빠른 시작 가이드

## 목차
- [1. 환경과 목표](#sec-01)
- [2. 배포 방식 개요](#sec-02)
- [3. 방식 A: k3s를 사용한 배포](#sec-03)
- [4. 방식 B: 표준 Kubernetes를 사용한 배포](#sec-04)
- [5. 중국 내 네트워크에서의 이미지 풀링 권장 사항(선택 사항)](#sec-05)
- [6. ClawManager 배포](#sec-06)
- [7. 웹 페이지 시작](#sec-08)
- [8. 빠른 시작 가이드(로그인 후 초기화 및 OpenClaw 인스턴스 생성)](#sec-09)
- [9. 콘솔 및 AI Gateway 기타 기능 설명](#sec-12)
- [10. 워크스페이스 모듈 설명](#sec-13)
- [11. 문제와 대응 빠른 참조](#sec-14)
- [12. 권장 최종 점검 순서(자가 점검용)](#sec-15)

<a id="sec-01"></a>
## 1. 환경과 목표
- **시스템 가정**: `x86_64` 아키텍처 Linux 서버.
- **배포 목표**: **ClawManager**를 배포하고 Web 페이지에서 보안 모델 구성을 완료한 뒤, **OpenClaw Desktop** 인스턴스를 생성하고 시작합니다.
- **적용 시나리오**:
  - **방식 A: k3s 단일 노드/경량 클러스터 배포**
  - **방식 B: 표준 Kubernetes 클러스터 배포**(예: kubeadm 클러스터, 기업용 K8s 클러스터, 클라우드 K8s 클러스터)


---

<a id="sec-02"></a>
## 2. 배포 방식 개요
다음 두 가지 방식 중 하나로 배포할 수 있습니다:

### 방식 A: k3s 배포
단일 노드, 테스트 환경 또는 경량 프로덕션 환경에 적합합니다.

### 방식 B: 표준 Kubernetes 배포
이미 표준 Kubernetes 클러스터를 갖춘 서버 환경에 적합합니다.

어떤 방식을 사용하든 최종적으로 동일한 ClawManager 매니페스트를 적용합니다:

```bash
kubectl apply -f deployments/k8s/clawmanager.yaml
```

---

<a id="sec-03"></a>
## 3. 방식 A: k3s를 사용한 배포

### 3.1 k3s 설치
```bash
curl -sfL https://get.k3s.io | sh -
```

중국 내 네트워크에서는 미러 소스를 사용하여 설치할 수 있습니다:

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh |   INSTALL_K3S_MIRROR=cn sh -
```

### 3.2 서비스 상태 확인
```bash
sudo systemctl status k3s --no-pager
sudo systemctl enable k3s
```

### 3.3 kubectl 구성
현재 사용자가 `kubectl`을 직접 사용할 수 없다면 다음을 실행합니다:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER:$USER" ~/.kube/config
```

또는 임시로 지정합니다:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### 3.4 클러스터 검증
```bash
kubectl get nodes
```

정상이라면 노드가 `Ready` 상태로 표시됩니다.

---

<a id="sec-04"></a>
## 4. 방식 B: 표준 Kubernetes를 사용한 배포

> 사용 가능한 Kubernetes 클러스터가 이미 있는 x86 서버 환경에 적용됩니다.

### 4.1 사전 점검
현재 `kubectl`이 대상 클러스터에 연결되어 있는지 확인합니다:

```bash
kubectl get nodes
kubectl get ns
```

정상이라면 최소 1개의 `Ready` 노드가 보여야 합니다.

### 4.2 기본 StorageClass 확인
ClawManager의 MySQL과 MinIO는 영구 스토리지가 필요합니다. 먼저 클러스터에 기본 `StorageClass`가 있는지 확인하는 것을 권장합니다:

```bash
kubectl get storageclass
```

클러스터에 기본 스토리지 클래스가 이미 있다면 바로 배포를 계속할 수 있습니다.

**기본 StorageClass가 없는 경우**, 사용 가능한 PV / PVC를 미리 준비하거나 로컬 경로 스토리지 방식을 사용하는 것을 권장합니다. 그렇지 않으면 이후 다음과 같은 문제가 발생할 수 있습니다:

```text
pod has unbound immediate PersistentVolumeClaims
```

---

<a id="sec-05"></a>
## 5. 중국 내 네트워크에서의 이미지 풀링 권장 사항(선택 사항)
서버가 Docker Hub 또는 기타 공개 레지스트리에 느리게 접근하는 경우 이미지 가속을 구성할 수 있습니다.

### 5.1 k3s 시나리오: `/etc/rancher/k3s/registries.yaml` 구성
```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"
      - "https://docker.nju.edu.cn"
      - "https://docker.1ms.run"
  quay.io:
    endpoint:
      - "https://quay.mirrors.ustc.edu.cn"
  gcr.io:
    endpoint:
      - "https://gcr.mirrors.ustc.edu.cn"
  k8s.gcr.io:
    endpoint:
      - "https://registry.aliyuncs.com/google_containers"
```

수정 후 다음을 실행합니다:

```bash
sudo systemctl restart k3s
```

### 5.2 이미지 풀링 검증
```bash
sudo k3s crictl pull docker.io/rancher/mirrored-pause:3.6
```

---

<a id="sec-06"></a>
## 6. ClawManager 배포

### 6.1 프로젝트 코드 가져오기
```bash
git clone https://github.com/Yuan-lab-LLM/ClawManager.git
cd ClawManager
```

### 6.2 배포 매니페스트 적용
저장소 루트 디렉터리에서 실행합니다:

```bash
kubectl apply -f deployments/k8s/clawmanager.yaml
```

### 6.3 기본 리소스 확인
```bash
kubectl get ns
kubectl get pods -n clawmanager-system
kubectl get svc -n clawmanager-system
```

정상적인 경우 다음 구성 요소가 표시됩니다:
- `clawmanager-app`
- `mysql`
- `minio`
- `skill-scanner`

다음 오류가 보이면:

```text
0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims
```

이는 클러스터 스토리지에서 MySQL / MinIO가 PVC 미바인드로 인해 시작되지 못한다는 의미입니다. 문서 끝의 다음 항목으로 바로 이동하세요:

- [11.1 스토리지 문제 전용 처리(PV/PVC)](#sec-14-storage)

---

<a id="sec-08"></a>
## 7. 웹 페이지 시작

### 7.1 NodePort로 접근
ClawManager의 프런트엔드 Service는 기본적으로 HTTPS NodePort를 사용합니다. 먼저 확인합니다:

```bash
kubectl get svc -n clawmanager-system
```

프런트엔드 포트가 다음과 같다면:

```text
443:30443/TCP
```

브라우저에서 직접 다음으로 접근할 수 있습니다:

```text
https://<서버IP>:30443
```


### 7.2 최초 HTTPS 접근 안내
일반적으로 자체 서명 인증서를 사용하므로 브라우저가 “안전하지 않음” 또는 인증서 경고를 표시할 수 있습니다. 다음을 클릭합니다:

```text
고급 → 계속 방문
```

그러면 페이지에 들어갈 수 있습니다.

---

<a id="sec-09"></a>
## 8. 빠른 시작 가이드(로그인 후 초기화 및 OpenClaw 인스턴스 생성)

위 배포를 완료하고 관리 페이지를 성공적으로 연 후에도, 실제로 **OpenClaw** 인스턴스를 생성하고 시작하려면 다음 초기화 단계를 완료해야 합니다.

### 8.1 시스템 로그인
1. 배포 완료 후 페이지를 엽니다. 예: `https://<노드IP>:30443`.
2. 기본 관리자 계정으로 로그인합니다:
   - **사용자 이름**: `admin`
   - **비밀번호**: `admin123`
3. 처음 로그인한 후에는 필요에 따라 기본 비밀번호를 변경하는 것을 권장합니다.


### 8.2 보안 모델 구성(AI Gateway)

![그림 1: AI Gateway 구성](./main/1.png)
로그인 후 먼저 사용 가능한 **보안 모델**을 구성해야 하며, 이는 플랫폼과 이후 인스턴스에서 공통으로 사용됩니다.

1. 왼쪽 메뉴에서 **AI Gateway** → **모델**을 클릭합니다.
2. 새 모델을 추가하거나 기존 모델을 편집하고, 연결하는 모델 서비스에 따라 다음 정보를 입력합니다:

   * **표시 이름**: 식별하기 쉬운 이름을 입력합니다.
   * **벤더 템플릿**: 모델 서비스 유형에 따라 해당 템플릿을 선택합니다. 사용자 정의 또는 호환 인터페이스를 사용하는 경우 **Local / Internal**을 선택할 수 있습니다.
   * **프로토콜**: 인터페이스 프로토콜에 따라 **OpenAI Compatible** 또는 실제 사용하는 다른 프로토콜을 선택합니다.
   * **Base URL**: 모델 서비스가 제공하는 인터페이스 주소를 입력합니다.
   * **API Key**: 해당 모델 서비스의 유효한 키를 입력합니다.
   * **Provider Model**: 실제 호출할 모델 이름을 입력합니다.
   * **통화**: 실제 상황에 맞게 입력합니다. 비용 표시가 필요 없다면 기본값을 유지할 수 있습니다.
   * **입력 가격 / 출력 가격**: 비용 통계를 하지 않을 경우 `0`을 입력할 수 있습니다.
3. 제출 전에 반드시 다음 항목을 체크합니다:

   * **보안 모델**
   * **사용**
4. **저장**을 클릭합니다。

> 참고: 페이지의 이미지는 입력 위치와 예시 형식을 보여주기 위한 것입니다. 실제 내용은 사용 중인 모델 서비스 구성에 따라 입력하세요。


### 8.3 OpenClaw 인스턴스 생성
모델 구성이 완료되면 **OpenClaw Desktop** 인스턴스를 생성합니다.

1. 왼쪽 아래의 **ADMIN**을 클릭하여 **워크스페이스**로 전환합니다.
2. **인스턴스 생성**을 클릭합니다。

![](./main/2.png)
#### 1단계: 기본 정보
- **인스턴스 이름**을 입력합니다(최소 3자).
- 설명은 선택 사항이며 비워 둘 수 있습니다.
- **다음**을 클릭합니다.

![](./main/3.png)
#### 2단계: 유형 선택
- **OpenClaw Desktop**을 선택합니다.
- **다음**을 클릭합니다。


![](./main/4.png)
#### 3단계: 구성
- **Small** 사양을 바로 선택할 수 있습니다:
  - `2 CPU`
  - `4 GB RAM`
  - `20 GB Disk`
- 아래 사용자 정의 구성 영역에서 필요에 따라 수정할 수도 있습니다。
- OpenClaw 리소스 주입 부분에서는 필요에 따라 다음을 선택할 수 있습니다:
  - **수동 리소스**
  - **리소스 패키지**
  - **아카이브 가져오기**
- 처음 사용하는 경우 기본값을 유지하거나 **수동 리소스**를 선택해도 됩니다。
- 마지막으로 **생성**을 클릭합니다。

### 8.4 첫 생성 안내
- **OpenClaw** 인스턴스를 처음 생성할 때는 필요한 이미지를 다운로드하고 환경을 초기화해야 하므로 시간이 더 오래 걸립니다。
- 네트워크가 느리거나 처음 이미지 풀링을 수행하는 경우, 인스턴스 상태가 오랫동안 **생성 중**으로 표시될 수 있습니다. 잠시 기다려 주세요。
- 오랜 시간이 지나도 시작되지 않으면 Kubernetes / Docker 로그로 돌아가 이미지, PVC, 게이트웨이 모델 등의 문제를 점검하세요。

---

<a id="sec-12"></a>
## 9. 콘솔 및 AI Gateway 기타 기능 설명

모델 구성 외에도 플랫폼 홈의 콘솔과 AI Gateway는 감사, 비용, 규칙 거버넌스 등의 기능을 제공하여 관리자가 클러스터 상태, 모델 호출 기록, 보안 정책 실행 상태를 중앙에서 쉽게 확인할 수 있도록 합니다。

### 9.1 콘솔 개요

![](./main/5.png)

콘솔 홈은 현재 클러스터와 플랫폼의 전체 운영 상태를 보여주며, 관리자가 리소스 사용량과 시스템 상태를 빠르게 파악할 수 있도록 합니다。

주요 내용은 다음과 같습니다：

- **클러스터 기본 정보 개요**: 현재 플랫폼의 총 사용자 수, 총 인스턴스 수, 실행 중 인스턴스 수, 총 스토리지 사용량을 표시합니다。
- **노드 개요**: 현재 사용 가능한 노드 수와 현재 클러스터의 주요 스케줄링 노드 정보를 표시합니다。
- **리소스 신청 현황**: 현재 플랫폼이 신청한 CPU, 메모리, 디스크 리소스 총량을 표시합니다。
- **용량 대시보드**: 노드, CPU, 메모리, 디스크 등 차원별로 전체 리소스 용량과 현재 사용률을 표시하여 클러스터에 사용 가능한 여유가 있는지 판단하기 쉽게 합니다。
- **기반 시설 표**: 현재 노드, 리소스 및 기본 런타임 환경의 상태 정보를 확인하는 데 사용됩니다。

> 참고: 콘솔은 주로 플랫폼 전체 리소스, 노드, 인스턴스 운영 개요를 보는 데 사용되며, 특정 인스턴스 내부의 OpenClaw 작업에 직접 사용되지는 않습니다。

### 9.2 AI Gateway 기능 개요

AI Gateway에는 **모델** 구성 외에도 다음 모듈이 있습니다：

- **AI 감사**: 모델 호출 Trace, 요청 및 응답 페이로드, 위험 히트, 라우팅 결정 및 호출 상세를 확인합니다。
- **비용**: Token 사용량, 예상 비용, 내부 비용 및 추세 통계를 확인합니다。
- **위험 제어 규칙**: 민감 콘텐츠 감지 규칙을 구성하고, 일치 시 통과시킬지 보안 모델로 라우팅할지 제어합니다。

### 9.3 비용 모듈

비용 페이지는 플랫폼 모델 호출의 비용과 Token 사용량을 집계하여 관리자가 전체 소비 상황을 이해하는 데 도움을 줍니다。

![](./main/6.png)

페이지에는 주로 다음 내용이 포함됩니다：

- **입력 Token**: 입력 프롬프트 총량을 집계합니다。
- **출력 Token**: 모델이 생성한 콘텐츠 총량을 집계합니다。
- **예상 비용**: Provider 단가에 따라 추정한 비용입니다。
- **내부 비용**: 보안 모델과 관련된 내부 회계 비용입니다。
- **일일 비용 추세**: 최근 7일 동안 현재 창 내 예상 비용과 Token 변화를 확인합니다。
- **사용자 요약**: 사용자별 사용량과 비용을 집계합니다。
- **인스턴스 요약**: 인스턴스별 사용량과 비용을 집계합니다。
- **최근 비용 기록**: Trace, 사용자, 모델 등의 조건으로 비용 기록을 검색하고 페이지 단위로 확인할 수 있으며, 감사 상세로 이동할 수 있습니다。

> 참고: 아직 모델 호출 기록이 생성되지 않았다면 입력 Token, 출력 Token, 비용 및 추세 차트가 모두 0으로 표시될 수 있으며 이는 정상입니다。

### 9.4 AI 감사 모듈

AI 감사 페이지는 최근 관리 대상 모델 호출 기록을 확인하는 데 사용되며, 관리자가 모델 호출, Token 사용량 및 라우팅 결과를 조사하는 데 도움을 줍니다。

![](./main/7.png)

주요 기능은 다음과 같습니다：

- **최근 AI Trace**: 최근 모델 호출 경로를 확인합니다。
- **Trace 목록**: 통합 테이블에서 최근 관리 대상 Trace를 확인합니다。
- **검색 및 필터**: Trace, 요청 내용, 사용자, 모델 등의 조건으로 검색할 수 있습니다。
- **상태 필터**: 상태별로 서로 다른 호출 결과를 볼 수 있습니다。
- **모델 필터**: 모델별로 해당 호출 기록을 필터링할 수 있습니다。
- **페이지 새로고침**: 페이지 단위 조회 및 최신 감사 결과 수동 새로고침을 지원합니다。

> 참고: 페이지에 “AI 감사 기록이 없습니다”라고 표시되면 아직 실제 모델 호출 요청이 발생하지 않았다는 의미입니다。

### 9.5 위험 제어 규칙 모듈

위험 제어 규칙 페이지는 민감 콘텐츠 감지 규칙을 구성하고, 규칙 일치 후 처리 동작을 결정하는 데 사용됩니다。

![](./main/8.png)

이 모듈은 주로 다음을 지원합니다：

- **규칙 목록 관리**: 모든 규칙과 활성화 상태를 확인합니다。
- **규칙 분류 조회**: 개인정보, 회사 정보, 고객 업무, 보안 자격 증명, 재무/법무, 정치적 민감, 사용자 정의 등의 분류별로 규칙을 확인할 수 있습니다。
- **규칙 필드 구성**: 규칙 ID, 표시 이름, 심각도, 동작, 정렬 순서, 정규식 Pattern, 설명을 설정할 수 있습니다。
- **규칙 동작 제어**: 규칙 일치 시 통과시키거나 보안 모델로 라우팅하도록 선택할 수 있습니다。
- **일괄 활성화 / 비활성화**: 규칙 상태를 일괄 조정할 수 있습니다。
- **규칙 테스트 콘솔**: 샘플 텍스트를 붙여넣어 활성 규칙 또는 초안 규칙 중 어떤 내용이 일치하는지 테스트할 수 있습니다。

현재 내장 규칙 예시는 다음을 포함하되 이에 국한되지 않습니다：

- 개인정보: 이메일 주소, 휴대전화 번호, 신분증 번호, 여권 번호, 은행 카드 컨텍스트, 주소, 이력서 내용 등。
- 회사 정보: 내부망 IP, 내부 도메인명, 호스트 명명, Kubernetes Service DNS, 프로젝트 코드명, 조직 구조, 급여 / HR 정보 등。
- 고객 업무: 고객 목록, 계약 / 견적서, 송장 세금 번호, CRM / 티켓 데이터 등。
- 보안 자격 증명: 개인 키, API Key, Token, JWT, Cookie / Session, 데이터베이스 연결 문자열, Kubeconfig, 환경 변수 비밀 값 등。
- 재무/법무: 예산, 이익, 매출, 법무 의견, 소송, NDA 등。
- 정치적 민감: 정치 기관, 군사/국가 안보, 극단적 폭력 관련 표현 등。

> 참고: 기본 규칙은 이미 많은 일반적인 민감 정보 탐지 시나리오를 포괄하고 있습니다. 실제 사용 시 비즈니스 요구에 따라 일부 규칙을 추가, 조정 또는 비활성화할 수 있습니다。
---

<a id="sec-13"></a>
## 10. 워크스페이스 모듈 설명

워크스페이스는 일반 사용자가 플랫폼에 들어온 후 사용하는 주요 작업 영역입니다. 개인 리소스 할당량 조회, 인스턴스 생성, 인스턴스 관리, OpenClaw 관련 리소스 유지에 사용됩니다. 이 모듈은 관리자 측의 “콘솔 개요”와 달리 일상 사용 및 운영 작업에 더 초점이 맞춰져 있습니다。

### 10.1 워크스페이스 홈
![](./main/9.png)
워크스페이스 홈은 현재 계정의 인스턴스 및 리소스 사용 현황을 표시하는 데 사용되며, 주로 다음 내용을 포함합니다：

- **내 인스턴스**: 현재 계정에서 생성한 인스턴스 수를 표시합니다。
- **실행 중**: 현재 실행 중인 인스턴스 수를 표시합니다。
- **사용된 스토리지**: 현재 계정이 사용 중인 스토리지 공간을 표시합니다。
- **내 리소스 할당량**: 현재 계정에서 사용 가능한 할당량 정보(인스턴스 수, 최대 CPU 코어 수, 최대 메모리, 최대 스토리지, 최대 GPU 수)를 표시합니다。
- **빠른 작업**: **새 인스턴스 생성** 및 **모든 인스턴스 보기** 두 개의 진입점을 제공하여 플랫폼을 빠르게 사용할 수 있게 합니다。

> 참고: 페이지에 “아직 인스턴스가 없습니다”가 표시되면, 바로 **새 인스턴스 생성**을 클릭하여 첫 번째 OpenClaw Desktop 인스턴스 생성을 시작할 수 있습니다。

### 10.2 내 인스턴스

**내 인스턴스** 페이지는 현재 계정에서 생성된 인스턴스를 통합 조회 및 관리하기 위한 페이지입니다. 이 페이지는 주로 인스턴스 관리 기능을 담당합니다。
![](./main/10.png)
일반적으로 지원되는 작업은 다음과 같습니다：

- **인스턴스 상태 보기**: 인스턴스가 생성 중, 실행 중, 중지됨 또는 비정상 상태인지 확인합니다。
- **인스턴스 상세 진입**: 인스턴스의 기본 정보, 리소스 구성 및 실행 상태를 확인합니다。
- **인스턴스 중지**: 인스턴스가 비정상이거나 환경을 다시 로드해야 하는 경우 중지 작업을 수행할 수 있습니다。
- **인스턴스 삭제**: 인스턴스가 더 이상 필요하지 않을 때 CPU, 메모리, 스토리지 등의 리소스를 해제하기 위해 직접 삭제할 수 있습니다。

> 참고: 인스턴스를 삭제하면 관련 리소스도 함께 정리됩니다. 실행 전에 내부 데이터와 구성이 백업되었는지 확인하세요。

### 10.3 리소스 관리

**리소스 관리** 페이지는 사용 가능한 OpenClaw 리소스 내용을 유지하여, 인스턴스 시작 후 주입하고 사용할 수 있도록 하는 데 사용됩니다。
![](./main/11.png)
페이지에는 주로 다음 부분이 있습니다：

- **리소스**: 사용 가능한 리소스 항목을 조회하고 유지합니다。
- **리소스 패키지**: 여러 리소스를 재사용 가능한 패키지로 묶어 일괄 주입을 쉽게 합니다。
- **주입 기록**: 리소스 주입 이력과 실행 상태를 확인합니다。

리소스 관리 페이지 왼쪽에서는 리소스 유형별로 구분 관리할 수도 있으며, 현재 페이지에 표시되는 유형은 다음과 같습니다：

- **채널**
- **스킬**
- **에이전트(출시 예정)**
- **예약 작업(출시 예정)**

페이지 오른쪽 상단에서는 다음을 지원합니다：

- **새로고침**: 현재 리소스 목록을 다시 불러옵니다。
- **새로 만들기**: 새로운 리소스 항목을 생성합니다。

> 참고: 리소스 관리는 주로 인스턴스 시작 후 사용할 수 있는 OpenClaw 리소스 내용을 준비하는 데 사용되며, 인스턴스 생성 과정을 직접 대체하지는 않습니다. 인스턴스 생성 시 **수동 리소스**, **리소스 패키지**, **아카이브 가져오기** 등의 방식과 함께 리소스를 주입할 수 있습니다。

---

<a id="sec-14"></a>
## 11. 문제와 대응 빠른 참조

<a id="sec-14-storage"></a>
### 11.1 스토리지 문제 전용 처리(PV/PVC)

다음 오류가 보이는 경우：

```text
0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims
```

클러스터 스토리지가 자동으로 바인딩되지 않았음을 의미합니다. 이 경우 x86 단일 노드 서버 방식으로 로컬 `hostPath` PV/PVC를 수동 생성할 수 있습니다。

> 이 방식은 단일 노드 서버 테스트 또는 경량 환경에 적합합니다. 프로덕션 환경에서는 NFS, Ceph, 클라우드 디스크 등 정식 스토리지를 사용하는 것이 좋습니다。

#### 11.1.1 PV 생성
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-local
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: /tmp/mysql-data
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: minio-pv-local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: /tmp/minio-data
EOF
```

#### 11.1.2 PVC 생성
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: clawmanager-system
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeName: mysql-pv-local
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-data
  namespace: clawmanager-system
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: minio-pv-local
EOF
```

#### 11.1.3 Pod 재생성
```bash
kubectl delete pod --all -n clawmanager-system
```

#### 11.1.4 상태 다시 확인
```bash
kubectl get pvc -n clawmanager-system
kubectl get pods -n clawmanager-system -w
```

예상 결과：
- `mysql-data` / `minio-data`가 `Bound`
- `mysql` / `minio` / `skill-scanner` / `clawmanager-app`가 최종적으로 `Running`

---

| 현상 | 원인 | 처리 |
| :--- | :--- | :--- |
| `kubectl`의 `localhost:8080` 연결이 거부됨 | kubeconfig가 구성되지 않음 | `KUBECONFIG`를 설정하거나 `~/.kube/config`에 복사 |
| Pod 이미지 풀링 타임아웃 | Docker Hub / GHCR 네트워크 불안정 | 이미지 가속 또는 프록시 구성 |
| MySQL / MinIO가 계속 `Pending` | PVC가 바인딩되지 않음 | `StorageClass`를 확인하거나 PV/PVC를 수동 생성 |
| 브라우저에서 페이지를 열 수 없음 | NodePort가 열려 있지 않음 / `port-forward` 프로세스가 유지되지 않음 | 포트를 열거나 포워딩 터미널을 유지 |
| 페이지는 열리지만 OpenClaw 인스턴스를 생성할 수 없음 | 보안 모델이 구성되지 않음 | 먼저 **AI Gateway → 모델**에서 보안 모델을 구성하고 활성화 |
| 인스턴스가 오랫동안 “생성 중” 상태로 남음 | 첫 이미지 풀링에 시간이 오래 걸림 / 스토리지 또는 네트워크 문제 | 잠시 기다리고, 필요 시 Pod와 이벤트 확인 |

---

<a id="sec-15"></a>
## 12. 권장 최종 점검 순서(자가 점검용)
1. `kubectl get nodes`
2. `kubectl get storageclass`
3. `kubectl get pods -n clawmanager-system`
4. `kubectl get pvc -n clawmanager-system`
5. `kubectl get svc -n clawmanager-system`
6. 브라우저에서 `https://<IP>:30443` 열기
7. 백엔드에 로그인하여 **보안 모델 구성** 완료
8. 워크스페이스에서 **OpenClaw Desktop** 인스턴스 생성
