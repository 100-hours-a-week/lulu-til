# 12주차 위클리챌린지

## 커뮤니티 프로젝트 Helm · ArgoCD · Ingress · 모니터링 구성
### Helm Chart 전환, GitOps 자동 배포, Ingress/TLS 적용 및 PLG 스택 구축

---

## 1. Helm Chart

### 1-1. backend-app/Chart.yaml

```yaml
apiVersion: v2
name: backend-app
description: A Helm chart for deploying backend Application
type: application
version: 0.1.0
appVersion: "0.1.0"
```

### 1-2. backend-app/values.yaml

```yaml
replicaCount: 2

rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0

image:
  repository: 536697237610.dkr.ecr.ap-northeast-2.amazonaws.com/lulu-backend
  tag: "d75601198a7a20d3ce23b8a0b65ae296eca45e88"

containerPort: 8080

readinessProbe:
  path: /api/health-check
  initialDelaySeconds: 15
  periodSeconds: 10

livenessProbe:
  path: /api/health-check
  initialDelaySeconds: 20
  periodSeconds: 20
  failureThreshold: 3

springProfilesActive: "prod"

dbPassword: ..비밀
jwtSecret: ..비밀
awsAccessKey: ""
awsSecretKey: ""

service:
  protocol: TCP
  port: 8080
```

### 1-3. backend-app/templates/backend-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  SPRING_PROFILES_ACTIVE: {{ .Values.springProfilesActive }}
```

### 1-4. backend-app/templates/backend-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: {{ .Values.rollingUpdate.maxSurge }}
      maxUnavailable: {{ .Values.rollingUpdate.maxUnavailable }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: backend-container
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.containerPort }}
          envFrom:
            - configMapRef:
                name: {{ .Release.Name }}-configmap
            - secretRef:
                name: {{ .Release.Name }}-secret
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.path }}
              port: {{ .Values.containerPort }}
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.path }}
              port: {{ .Values.containerPort }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
      imagePullSecrets:
        - name: ecr-secret
```

### 1-5. backend-app/templates/backend-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
stringData: # stringData를 사용하면 적용 시 자동으로 Base64 인코딩 처리
  DB_PASSWORD: {{ .Values.dbPassword }}
  JWT_SECRET: {{ .Values.jwtSecret }}
  CLOUD_AWS_CREDENTIALS_ACCESS_KEY: {{ .Values.awsAccessKey }}
  CLOUD_AWS_CREDENTIALS_SECRET_KEY: {{ .Values.awsSecretKey }}
```

### 1-6. backend-app/templates/backend-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
spec:
  type: ClusterIP
  selector:
    app: {{ .Release.Name }}
  ports:
    - protocol: {{ .Values.service.protocol }}
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
```

### 1-7. frontend-app/Chart.yaml

```yaml
apiVersion: v2
name: frontend-app
description: A Helm chart for deploying frontend Application
type: application
version: 0.1.0
appVersion: "0.1.0"
```

### 1-8. frontend-app/values.yaml

```yaml
replicaCount: 2

rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0

image:
  repository: 536697237610.dkr.ecr.ap-northeast-2.amazonaws.com/lulu-frontend
  tag: "73f326805c72fd12359e8ca06dc67687582b7a5e"

containerPort: 3000

readinessProbe:
  path: /html/index.html
  initialDelaySeconds: 5
  periodSeconds: 20

livenessProbe:
  path: /html/index.html
  initialDelaySeconds: 5
  periodSeconds: 20
  failureThreshold: 5
  timeoutSeconds: 5

apiBaseUrl: "https://lulu-roh.xyz"

service:
  protocol: TCP
  port: 3000
```

### 1-9. frontend-app/templates/frontend-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  API_BASE_URL: {{ .Values.apiBaseUrl }}
```

### 1-10. frontend-app/templates/frontend-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: {{ .Values.rollingUpdate.maxSurge }}
      maxUnavailable: {{ .Values.rollingUpdate.maxUnavailable }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: frontend-container
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.containerPort }}
          envFrom:
            - configMapRef:
                name: {{ .Release.Name }}-configmap
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.path }}
              port: {{ .Values.containerPort }}
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.path }}
              port: {{ .Values.containerPort }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
            timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
      imagePullSecrets:
        - name: ecr-secret
```

### 1-11. frontend-app/templates/front-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
spec:
  type: ClusterIP
  selector:
    app: {{ .Release.Name }}
  ports:
    - protocol: {{ .Values.service.protocol }}
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
```

---

## 2. ArgoCD YML / CI 워크플로우

### 2-1. root-app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/nominsol/Community-GitOps.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 2-2. apps/backend-app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/nominsol/Community-GitOps.git
    targetRevision: main
    path: charts/backend-app
    helm:
      valueFiles: [values.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 2-3. apps/front-app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/nominsol/Community-GitOps.git
    targetRevision: main
    path: charts/frontend-app
    helm:
      valueFiles: [values.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 2-4. fe-cicd.yml (수정)

```yaml
name: Frontend CI/CD

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: lulu-frontend
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image_tag=$IMAGE_TAG" >> "$GITHUB_OUTPUT"

      - name: Checkout GitOps repo
        uses: actions/checkout@v3
        with:
          repository: nominsol/Community-GitOps
          token: ${{ secrets.GITOPS_PAT }}
          path: gitops

      - name: Update image tag in values.yaml
        run: |
          cd gitops
          sed -i "s/^  tag:.*/  tag: \"${{ steps.build-image.outputs.image_tag }}\"/" charts/frontend-app/values.yaml
          echo "----- 변경된 values.yaml -----"
          cat charts/frontend-app/values.yaml

      - name: Commit and push
        run: |
          cd gitops
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add charts/frontend-app/values.yaml
          git commit -m "frontend: update image tag to ${{ steps.build-image.outputs.image_tag }}"
          git push
```

### 2-5. be-cicd.yml (수정)

```yaml
name: Backend CI/CD

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: lulu-backend
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image_tag=$IMAGE_TAG" >> "$GITHUB_OUTPUT"

      - name: Checkout GitOps repo
        uses: actions/checkout@v3
        with:
          repository: nominsol/Community-GitOps
          token: ${{ secrets.GITOPS_PAT }}
          path: gitops

      - name: Update image tag in values.yaml
        run: |
          cd gitops
          sed -i "s/^  tag:.*/  tag: \"${{ steps.build-image.outputs.image_tag }}\"/" charts/backend-app/values.yaml
          echo "----- 변경된 values.yaml -----"
          cat charts/backend-app/values.yaml

      - name: Commit and push
        run: |
          cd gitops
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add charts/backend-app/values.yaml
          git commit -m "backend: update image tag to ${{ steps.build-image.outputs.image_tag }}"
          git push
```

### 2-6. GitOps 배포 테스트 결과

1. 테스트 주석 제거 후 push

   ![테스트 주석 제거 후 push](image.png)

2. GitHub Actions 실행 성공

   ![GitHub Actions](image.png)

3. Community-GitOps 저장소의 values.yaml 이미지 태그 커밋

   ![values.yaml 커밋](image.png)

4. ArgoCD Sync 상태 확인

   ![ArgoCD Sync](image.png)

- 코드 push → Actions 빌드/푸시 → GitOps 저장소 태그 커밋 → ArgoCD 자동 sync → 롤링 업데이트까지 전체 사이클이 사람 개입 없이 동작함을 확인

---

## 3. Ingress YML

### 3-1. app-ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - lulu-roh.xyz
        - origin.lulu-roh.xyz
      secretName: app-tls-secret
  rules:
    - host: lulu-roh.xyz
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-app-service
                port:
                  number: 8080
          - path: / # 프론트로 가는 길
            pathType: Prefix
            backend:
              service:
                name: frontend-app-service
                port:
                  number: 3000
```

### 3-2. clusterIssuer.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: xmxmxm3577@naver.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

---

## 4. 모니터링 / 로그 확인 구성 YML

### 4-1. monitoring-values.yaml (kube-prometheus-stack)

```yaml
grafana:
  adminPassword: ..비밀
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    hosts:
      - grafana.lulu-roh.xyz
    tls:
      - secretName: grafana-tls-secret
        hosts:
          - grafana.lulu-roh.xyz
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki.logging.svc.cluster.local:3100
      access: proxy
      isDefault: false

prometheus:
  prometheusSpec:
    retention: 7d
    serviceMonitorSelectorNilUsesHelmValues: false
```

### 4-2. loki-values.yaml (loki-stack)

```yaml
grafana:
  enabled: false   # grafana 를 이미 설치했어서 여기에서는 끈다

promtail:
  enabled: true

loki:
  persistence:
    enabled: true
    size: 10Gi
```

### 4-3. 모니터링 / 로그 조회 결과

![Grafana - Prometheus 메트릭 조회](image.png)

![Grafana - Loki 로그 조회](image.png)

- Grafana `Explore`에서 Prometheus(`up` 쿼리)와 Loki(`{namespace="default"}` 쿼리) 양쪽 모두 정상 조회 확인

---

## 5. 회고록

# 12주차 위클리 챌린지 - 과제 수행 과정 및 회고

## 0. 전체 설계 방향 잡기

이번 과제의 요구사항은 개인 프로젝트를 **Helm**과 **ArgoCD** 기반으로 배포하고, **Ingress**, **모니터링/로그** 확인 구성을 적용하는 것이었다. 각 단계가 다음 단계의 전제가 되기 때문에 아래 순서로 진행했다.

1. **Helm** — 배포 단위를 Chart로 묶어서, 이후 단계(ArgoCD)가 참조할 배포 가능한 하나의 원본을 먼저 만든다.
2. **Ingress** — 외부에서 클러스터로 들어오는 경로를 먼저 뚫어야, 나중에 ArgoCD/Grafana UI를 브라우저로 확인할 수 있다.
3. **ArgoCD** — Helm Chart를 GitOps 방식으로 자동 배포하는 파이프라인을 완성한다.
4. **모니터링/로그** — 배포 파이프라인이 안정되고 나서, 그 위에서 도는 것들을 관측하는 계층을 쌓는다.

이 순서로 진행하면서 각 단계마다 왜 이 기술/설정을 골랐는지를 먼저 비교해보고 결정한 뒤 구현했다.

---

## 1. Helm

### 1-1. 왜 Helm인가

지금은 백엔드/프론트 두 개뿐이지만, 실제 서비스라면 Namespace, ServiceAccount, Role, PVC, Ingress까지 더해져 관리할 YAML이 훨씬 많아지고, 여기에 dev/stage/prod 같은 환경까지 겹치면 리소스 구조는 같은데 값만 다른 상황이 반복된다. 예를 들어 dev의 API URL을 바꾸려면 configmap.yaml을, replicas를 바꾸려면 deployment.yaml을 각각 찾아 고쳐야 하는데, 이걸 정확히 찾아 고치기 어렵고 실수 여지도 크다. Helm으로 값과 구조를 분리해두면 `values.yaml`만 바꿔서 반복 배포할 수 있다는 게 가장 큰 이유였다.

### 1-2. Chart.yaml 필드별 선택 이유

| 필드 | 값 | 선택 이유 |
| --- | --- | --- |
| `apiVersion` | `v2` | Helm 3 이상에서는 v2가 표준이고, 템플릿 구조/기능 정의 기준이 된다. |
| `name` | 최상위 폴더명과 동일 (`backend-app`) | Helm 공식 권장사항. 다르게 하면 `helm lint`에서 경고가 뜨고, `helm package`로 패키징할 때 폴더 이름이 Chart.yaml의 name을 강제로 따라가서 형상관리 시 혼란이 생긴다. |
| `type` | `application` | `library`(재사용 가능한 공용 템플릿용)가 아니라 실제 배포 대상 애플리케이션이므로. |
| `version` | `0.1.0` | 아직 개발 단계라 낮게 시작. |
| `appVersion` | `"0.1.0"` | 실제 배포 대상(도커 이미지)의 버전을 명시. |

### 1-3. values.yaml로 뺄 값과 템플릿에 남길 값의 기준

`maxSurge`, `maxUnavailable` 같은 세부 값은 `values`로 뺐지만, `strategy.type` 자체(`RollingUpdate`/`Recreate`)는 값이 바뀌면 리소스의 **구조** 자체가 바뀐다고 판단해서 템플릿에 하드코딩했다. Secret의 `type: Opaque`도 같은 논리로, **값이 바뀌면 구조가 바뀌는 필드**와 **값만 바뀌는 필드**를 구분해서 후자만 `values`로 뺐다.

### 1-4. envFrom vs env + valueFrom

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| `env` + `valueFrom.configMapKeyRef` | 어떤 값이 어디서 오는지 명시적, 필요한 키만 선택적 주입 | ConfigMap/Secret에 값이 늘어날 때마다 Deployment yaml도 같이 수정 필요 |
| `envFrom.configMapRef` (선택) | ConfigMap/Secret 전체 주입, 값이 늘어나도 Deployment 안 건드림 | 어떤 키가 실제 주입되는지 Deployment만 봐서는 알기 어려움 |

백엔드에 필요한 설정값이 앞으로 계속 늘어날 가능성이 높다고 보고 `envFrom`을 선택했다. 대신 어떤 값이 주입되는지 파악하기 어렵다는 단점을 보완하기 위해 ConfigMap/Secret 설계를 문서로 먼저 정리하고 시작했다.

### 1-5. ConfigMap / Secret 분리 기준

- ConfigMap은 접근 권한만 있으면 누구나 값을 바로 읽을 수 있는 저장소다.
- Secret도 사실 Base64 인코딩일 뿐 암호화가 아니라 완벽한 보안 수단은 아니지만, 최소한 `kubectl` 명령 실행 시 값이 바로 노출되지 않고 민감 정보라는 시그널을 준다는 차이가 있다.

이 기준으로 `SPRING_PROFILES_ACTIVE`, `API_BASE_URL`처럼 노출돼도 사고로 안 이어지는 값은 ConfigMap에, `DB_PASSWORD`/`JWT_SECRET`/AWS Secret Key처럼 유출 시 직접 피해로 이어지는 값은 Secret에 넣었다.

### 1-6. Secret type: Opaque를 선택한 이유

Secret의 `type`에는 `kubernetes.io/dockerconfigjson`(레지스트리 인증 전용), `kubernetes.io/tls`(인증서 전용) 등 K8s가 정해둔 스키마가 있다. `DB_PASSWORD`, `JWT_SECRET`, AWS 자격증명은 이런 정해진 스키마에 해당하지 않는 애플리케이션 고유의 key-value라 `Opaque`를 선택했다. 반대로 ECR 이미지를 pull하기 위한 `ecr-secret`은 K8s가 이미지 레지스트리 인증 정보라는 걸 인식하고 kubelet이 자동으로 활용할 수 있도록 `kubernetes.io/dockerconfigjson` 타입으로 별도 생성해서 `imagePullSecrets`에 연결했다.

### 1-7. data vs stringData

`data`를 쓰면 매번 `echo -n "값" | base64`를 실행해서 인코딩된 문자열을 손으로 붙여넣어야 하는데, `stringData`는 평문으로 작성하면 apply 시점에 K8s가 알아서 인코딩해준다. 휴먼 에러 가능성이 줄어드는 대신, yaml 파일에 평문이 남기 때문에 이 파일은 Git에 그대로 커밋하지 않고 별도 관리해야 한다는 트레이드오프를 인지하고 진행했다. (이 부분은 지금 미완성 상태로 남아있다.)

### 1-8. 수행 과정

1. 기존 `backend/`(순수 yaml 4개), `frontend/`(순수 yaml 3개) 디렉토리를 `backend-app/`, `frontend-app/` Chart 구조(`Chart.yaml`, `values.yaml`, `templates/`)로 재구성
2. Deployment/ConfigMap/Secret/Service 각 템플릿에서 환경마다 달라질 수 있는 값(replica 수, 이미지 태그, probe 경로, 시크릿 값 등)을 골라 `{{ .Values.xxx }}`로 치환
3. `{{ .Release.Name }}`으로 리소스 이름을 통일해서, release 이름만 바꾸면 동일 Chart로 여러 환경 배포가 가능하도록 구성
4. `helm install`로 로컬(EC2 마스터)에서 직접 설치해 동작 검증
5. ArgoCD 도입 이후에는 이 Chart들을 `Community-GitOps` 저장소로 옮겨서 ArgoCD Application이 참조하도록 전환

---

## 2. Ingress

### 2-1. AWS 로드밸런서 선택 — ALB vs NLB

지금 클러스터는 EKS가 아니라 kubeadm 기반 자체 구축 클러스터(마스터 1대 + 워커 2대)다. EKS가 아니므로 `cloud-controller-manager`가 없고, `Service type: LoadBalancer`를 선언해도 AWS가 자동으로 NLB/ALB를 만들어주지 않는다.

| 방식 | 동작 | 이번 구조에 맞는가 |
| --- | --- | --- |
| ALB (L7) | HTTP 헤더/경로 라우팅, TLS 종료 가능 | 이미 `ingress-nginx`가 path 라우팅과 TLS 종료(cert-manager)를 다 하고 있어서, ALB가 또 L7 라우팅을 하면 로직이 두 군데(ALB 타겟그룹 + Ingress rule)로 쪼개진다. `aws-load-balancer-controller`도 EKS 전제라 자동 반영이 안 돼서 타겟그룹을 수동으로 계속 맞춰야 한다. |
| NLB (L4, TCP 패스스루) (선택) | 80/443을 그대로 NodePort로 전달만 함 | 라우팅과 TLS 종료를 전부 `ingress-nginx` + `cert-manager`에 위임 가능. 순수 전달자라 설정이 단순하다. |

최종 구조는 다음과 같다.

```
NLB(TCP 80/443) → 워커노드 NodePort(30080/30443) → ingress-nginx → path 기반 라우팅
```

### 2-2. Ingress Controller 선택 — NGINX

| 후보 | 검토 결과 |
| --- | --- |
| Traefik | 핫 리로드는 매력적이지만 대규모 트래픽 처리량 이슈 사례가 있음 |
| Istio Gateway | 서비스 메시 전체 도입이 필요해 현재 규모에는 오버킬 |
| 클라우드 매니지드형 | EKS 전제라 자체 구축 클러스터 구조에 맞지 않음 |
| **NGINX Ingress Controller** (선택) | 특정 클라우드에 종속되지 않고 사실상의 표준이라, 문제 발생 시 레퍼런스를 찾기 가장 쉬움 |

### 2-3. 수행 과정

1. `ingress-nginx`를 Helm으로 설치 (`controller.service.type=NodePort`, 포트 30080/30443 고정, `controller.kind=DaemonSet`으로 모든 워커에 강제 배치)
2. `cert-manager` 설치 + `ClusterIssuer`(Let's Encrypt, HTTP-01) 구성
3. AWS 콘솔에서 NLB 생성 — 리스너 TCP 80/443, 대상 그룹 2개(각각 30080/30443 인스턴스 타겟), 헬스체크는 HTTP `/healthz`
4. Gabia DNS 설정 — apex 도메인(`lulu-roh.xyz`)은 기존 CloudFront로, 새로 필요해진 `argocd.`/`grafana.` 서브도메인은 NLB로 CNAME 연결
5. CloudFront를 쓰고 있어서, 오리진 SNI 인증서 불일치 문제를 풀기 위해 `origin.lulu-roh.xyz` 서브도메인을 추가로 만들고 인증서 SAN에 포함, CloudFront 오리진 요청 정책을 `AllViewer`로 설정해 원본 Host 헤더가 유지되도록 함
6. `app-ingress.yaml`로 `/api` → backend-service, `/` → frontend-service path 라우팅 적용

### 2-4. 마주친 문제와 해결

**문제 1 — NLB 자체에 접속이 안 됨**

`curl`이 거부가 아니라 136초 넘게 무한 대기하다 타임아웃되는 게 단서였다. NLB 콘솔의 보안 탭을 확인하니, **NLB 자신에게 붙은 보안그룹**이 인바운드를 `default` SG 멤버로만 한정하고 있었다. 워커 노드 SG만 계속 들여다보다가 NLB 자체에도 SG가 붙는다는 걸 놓치고 있었던 것이다. NLB SG의 인바운드에 TCP 80/443을 `0.0.0.0/0`으로 추가해서 해결했다.

**문제 2 — HTTPS 대상 그룹이 계속 Unhealthy**

헬스체크가 30443(HTTPS) 포트에 평문 HTTP로 검사 요청을 보내고 있어서 프로토콜이 안 맞아 실패하고 있었다. 헬스체크의 트래픽 포트 재정의를 30080으로 지정해서(ingress-nginx의 `/healthz`는 항상 평문 HTTP로만 응답) 해결했다.

이 두 문제를 겪으면서, 이후로는 네트워크 경로 위의 각 구간(NLB 자체 SG → 워커노드 SG → ingress-nginx)을 curl로 하나씩 분리해서 테스트하는 습관이 생겼다. 막연히 전체를 의심하기보다 어디까지는 되고 어디서부터 안 되는지를 좁혀나가는 게 훨씬 빨랐다.

---

## 3. ArgoCD

### 3-1. SSH 직접 배포 vs GitOps(ArgoCD)

기존에는 GitHub Actions가 이미지를 빌드/푸시한 뒤 배스천을 경유해 EC2에 SSH로 접속, `docker compose up -d`를 실행하는 방식이었다.

| 항목 | 기존 (SSH 직접 배포) | GitOps (ArgoCD) |
| --- | --- | --- |
| 배포 주체 | GitHub Actions가 직접 | ArgoCD가 클러스터 안에서 스스로 |
| Actions가 하는 일 | 빌드 + 푸시 + **SSH 배포까지** | 빌드 + 푸시 + "이 버전 쓰세요"라고 **Git에 기록만** |
| 접근 권한 | Actions가 SSH 키를 계속 들고 있어야 함 | Actions는 이미지 레지스트리 + Git 쓰기 권한만, 클러스터 적용 권한은 ArgoCD만 보유 |
| 배포 이력 | Actions 로그에만 남음 | Git 커밋 이력 = 배포 이력 |
| 롤백 | 이전 태그로 SSH 스크립트 재실행 | `git revert` 한 번 |

GitOps를 선택한 이유는 단순히 요구사항이라서만은 아니었다. SSH 방식은 Actions가 서버에 대한 강한 접근 권한을 계속 들고 있어야 한다는 게 마음에 걸렸는데, ArgoCD 방식은 Actions의 역할을 "이미지 만들고 Git에 태그 적기"로 줄이고 실제 클러스터 적용 권한은 클러스터 내부의 ArgoCD만 갖도록 권한을 분리할 수 있다는 점이 좋았다.

### 3-2. 이미지 태그를 latest에서 github.sha로 바꾼 이유

ArgoCD는 "Git 파일 내용이 실제로 바뀌었는가"로 변경을 감지한다. `tag: latest`처럼 고정 문자열이면 이미지가 새로 빌드돼도 `values.yaml` 파일 자체는 안 바뀌니 ArgoCD 입장에선 "변경사항 없음"으로 보인다. 커밋 SHA를 태그로 쓰면 매 배포마다 파일 내용이 실제로 달라져서 GitOps의 선언적 상태 감지 방식과 맞는다.

### 3-3. App of Apps에서 플랫폼 계층을 제외한 이유

처음엔 ingress-nginx/cert-manager/PLG 스택까지 전부 ArgoCD Application으로 등록할까 고민했다. 그런데 **ArgoCD 자기 자신의 Ingress(`argocd.lulu-roh.xyz`)가 ingress-nginx와 cert-manager 위에서 서빙**되고 있다는 걸 깨달았다. 이걸 ArgoCD Application으로 등록했다가 sync 도중 문제가 생기면 ArgoCD UI 자체에 접속할 수 없는 상황이 생길 수 있어서, 아래처럼 계층을 나눴다.

| 계층 | 포함 | 관리 방식 |
| --- | --- | --- |
| 플랫폼 계층 | ingress-nginx, cert-manager, Prometheus/Grafana, Loki | EC2에서 `helm install`/`upgrade`로 직접 |
| 애플리케이션 계층 | backend-app, frontend-app | ArgoCD Application (GitOps) |

### 3-4. 수행 과정

1. `Community-GitOps` 저장소 신규 생성 (`apps/`, `charts/` 구조)
2. `lulu-frontend`/`lulu-backend`의 workflow에서 SSH 배포 블록 제거, gitops 저장소를 체크아웃해서 `values.yaml`의 이미지 태그만 `sed`로 바꿔 커밋/푸시하는 단계로 교체
3. GitHub PAT(`GITOPS_PAT`) 발급, 두 저장소에 Secret 등록
4. ArgoCD 설치 (`kubectl apply -f install.yaml`), 초기 admin 비밀번호 확인
5. ArgoCD 전용 Ingress(`argocd.lulu-roh.xyz`) 구성
6. `root-app`(App of Apps) 1회 apply → `apps/backend-app.yaml`, `apps/front-app.yaml`이 하위에서 자동 적용
7. 코드 push → Actions 성공 → gitops 저장소 커밋 → ArgoCD 자동 sync → 롤링 업데이트까지 전체 사이클 검증

### 3-5. 마주친 문제와 해결

**문제 1 — `revision main must be resolved`, Sync Status Unknown**

`Community-GitOps`를 Private으로 만들었는데, ArgoCD는 GitHub Actions가 쓰는 `GITOPS_PAT`와 완전히 별개의 자격증명이 필요하다는 걸 몰랐다. ArgoCD UI의 `Settings → Repositories`에서 같은 PAT로 저장소를 별도 등록하고 나서야 풀렸다.

**문제 2 — ECR 인증 토큰 만료로 `403 Forbidden`**

repo 연결 문제를 풀고 sync는 성공했는데, 파드가 `ImagePullBackOff`로 안 떴다. `describe pod`로 `403 Forbidden`을 확인했고, 이미지 태그 자체는 ECR에 정상 존재했다. `ecr-secret` 생성 시각을 보니 11일 전이었다 — ECR 토큰은 12시간 만료인데 그 뒤로 한 번도 갱신을 안 한 게 원인이었다. 당장은 `kubectl create secret docker-registry`를 재실행해서 해결했지만, 12시간마다 반복될 문제라 CronJob 자동 갱신이 필요하다는 걸 인지했다.

**문제 3 — 롤링 업데이트가 끝없이 반복되는 것처럼 보였던 사건**

파드 ReplicaSet 해시가 계속 바뀌고 새 파드가 `0/1 Running`에서 몇 분씩 안 넘어가서, 처음엔 readinessProbe 실패나 ArgoCD의 반복 sync를 의심했다. 그런데 `describe pod`를 까보니 `0/3 nodes are available: 3 node(s) had untolerated taint(s)`였고, 워커 노드 2대가 `unreachable` taint를 달고 `NotReady` 상태였다.

- SSH도 안 되는 상황이라 AWS Serial Console → SSM Session Manager로 접속 경로를 바꿔가며 겨우 들어갔고, `free -h`로 메모리가 바닥(`available: 89Mi`)난 걸 확인했다.
- `ps aux`로 보니 이전 롤아웃이 실패하면서 `Terminating`으로 멈춘 파드의 Java 프로세스가 안 죽고 메모리를 계속 붙잡고 있었다.
- 근본 대응으로 워커 인스턴스를 t3.medium으로 업그레이드했는데, 이 과정에서 응급처치로 넣었던 swap이 오히려 `running with swap on is not supported`로 kubelet 자체를 막아버리는 문제가 또 터졌다.

"메모리 부족엔 swap"이라는 일반 서버 상식이 쿠버네티스 노드에는 원칙적으로 안 맞는다는 걸 이때 직접 겪으며 배웠다. swap을 끄고 나서야 정상화됐다.

---

## 4. 모니터링 / 로그 확인 구성

### 4-1. 메트릭 — kube-prometheus-stack

Prometheus+Grafana는 쿠버네티스 진영의 사실상 표준이고, Prometheus Operator가 `ServiceMonitor` CRD로 스크레이핑 대상을 선언적으로 관리해준다는 점, Grafana가 대시보드/알림/데이터소스 통합 허브 역할을 한다는 점에서 별다른 대안 없이 채택했다.

### 4-2. 로그 — Loki vs ELK

| 항목 | ELK (Elasticsearch+Logstash+Kibana) | Loki (선택) |
| --- | --- | --- |
| 인덱싱 방식 | 로그 본문 전체를 풀텍스트 인덱싱 | 라벨(메타데이터)만 인덱싱, 본문은 압축 저장 |
| 리소스 요구량 | 무겁다 (Elasticsearch가 JVM 기반) | 훨씬 가볍다 |
| Grafana 연동 | 별도 Kibana 필요 | Grafana에 데이터소스로 바로 연결 |

클러스터가 저사양 노드로 구성돼 있어서 무거운 Elasticsearch를 올리는 건 처음부터 무리라고 판단했다. Loki는 이미 설치한 Grafana에 데이터소스만 추가하면 되니 관측 도구를 하나로 통일할 수 있다는 것도 선택 이유였다.

### 4-3. 수행 과정

1. `kube-prometheus-stack` Helm 설치 — Grafana Ingress(`grafana.lulu-roh.xyz`) + cert-manager 연동, `serviceMonitorSelectorNilUsesHelmValues: false`로 전체 네임스페이스 ServiceMonitor 자동 수집 설정
2. `loki-stack`(Loki+Promtail) Helm 설치 — `grafana.enabled: false`로 중복 Grafana 설치 방지
3. `kube-prometheus-stack` values의 `grafana.additionalDataSources`에 Loki 데이터소스 추가 후 `helm upgrade`로 반영
4. Grafana `Explore`에서 Prometheus(`up` 쿼리)와 Loki(`{namespace="default"}` 쿼리) 양쪽 모두 조회 확인

### 4-4. 마주친 문제와 해결

**문제 1 — 같은 스택을 중복 설치**

`monitoring` 네임스페이스에 파드가 비정상적으로 많이 떠 있어서 확인해보니, 이전에 테스트 삼아 설치했던 `kube-prometheus-stack`이라는 이름의 release가 3일 넘게 그대로 남아있었고, 그 위에 새 release(`monitoring`)를 또 설치한 상태였다. `monitoring-prometheus-node-exporter`가 전부 `Pending`이었는데, node-exporter가 hostPort(9100)를 직접 점유하는 방식이라 기존 release가 이미 포트를 선점하고 있어서 새 파드가 스케줄링이 안 된 거였다. 오래된 release를 `helm uninstall`로 정리하고 나서야 풀렸다.

**문제 2 — Loki StatefulSet 업그레이드 실패**

`helm upgrade`가 `cannot patch "loki" with kind StatefulSet: ... forbidden`으로 실패했다. StatefulSet은 `replicas`, `template` 등 일부 필드만 나중에 수정 가능하고 `volumeClaimTemplates`(볼륨 설정) 같은 필드는 한 번 만들어지면 수정할 수 없다는 걸 이때 알았다. 이전 설치 때의 볼륨 설정과 지금 `values.yaml`의 값이 달라서 생긴 충돌이라, `helm uninstall` 후 재설치로 해결했다.

**문제 3 — `loki-0`가 계속 `Pending`**

`describe pod`로 확인하니 `pod has unbound immediate PersistentVolumeClaims`였다. `kubectl get storageclass`를 확인해보니 자체 구축 클러스터에는 EKS와 달리 **기본 스토리지클래스(동적 볼륨 프로비저너)가 아예 없다**는 걸 알게 됐다. PVC를 만들어도 실제 디스크를 할당해줄 주체가 없었던 것이다. `local-path-provisioner`를 설치하고 기본 스토리지클래스로 지정해서 해결했다. 이 조치는 Loki뿐 아니라 앞으로 PVC가 필요한 모든 컴포넌트(Prometheus 등)에 공통으로 재사용될 인프라라는 점에서, 뒤늦게라도 미리 해결해둔 게 다행이었다고 생각한다.

**문제 4 — apex 도메인 인증서가 계속 `pending`, CloudFront는 502**

파드도 다 뜨고 네트워크도 다 뚫었다고 생각했는데, 정작 `https://lulu-roh.xyz`로 메인 서비스에 들어가려니 CloudFront가 502를 냈다. `kubectl get certificate`로 확인해보니 `app-tls-secret`이 만들어진 지 이틀이 넘었는데도 `READY: False`였고, `kubectl get challenge`로 보니 `origin.lulu-roh.xyz`(NLB로 직결) 챌린지는 `valid`인데 `lulu-roh.xyz`(CloudFront 경유) 챌린지만 계속 `pending`이었다.

- **원인 1 — 순환 구조.** CloudFront는 뷰어의 HTTP를 HTTPS로 리다이렉트한 뒤, 오리진(NLB)에도 HTTPS로 접속하면서 TLS 인증서를 검증한다. 그런데 그 오리진 인증서가 바로 지금 발급하려는 `app-tls-secret` 그 자체였다. 인증서를 발급받으려면 CloudFront가 오리진과 유효한 TLS로 통신해야 하는데, 그 TLS 인증서는 아직 발급 전이라 유효하지 않은 구조에 갇혀서 apex 도메인 챌린지는 처음부터 성공할 수 없는 상태였다. `origin.lulu-roh.xyz`는 CloudFront를 거치지 않고 Gabia에서 NLB로 직결되어 있어서 이 문제를 안 겪었고, 그래서 그동안 인증서 발급 실패를 눈치채지 못했다.
- **원인 2 — 오리진 설정 오류.** CloudFront 콘솔을 다시 보니, 기본 동작(`*`)의 오리진이 `origin.lulu-roh.xyz`가 아니라 NLB의 **원본 DNS 이름**으로 직접 설정되어 있었다. 이 상태로는 설령 인증서가 발급되더라도, CloudFront가 오리진에 보내는 SNI가 인증서의 SAN 목록(`lulu-roh.xyz`, `origin.lulu-roh.xyz`) 어디에도 안 걸려서 계속 502가 날 수밖에 없었다.

해결은 오리진 전용 서브도메인 + SNI 우회 접근 대신, **CloudFront↔NLB 구간의 TLS 검증 자체를 없애는 방향**으로 단순화했다.

1. 오리진 프로토콜을 HTTPS Only에서 HTTP Only(80번 포트)로 변경 → CloudFront가 오리진에 접속할 때 인증서 검증을 하지 않음
2. 사용자↔CloudFront 구간은 여전히 CloudFront의 `HTTP를 HTTPS로 리디렉션` 정책으로 HTTPS만 강제되므로 뷰어 입장의 보안은 유지
3. `ingress-nginx`의 `ssl-redirect: "true"`가 평문으로 들어온 요청을 다시 https로 리다이렉트하려다 무한 루프에 빠질 수 있어서, 이 어노테이션을 `"false"`로 함께 변경
4. 멈춰있던 Challenge/Certificate를 삭제해 재시도를 유도 → apex 도메인 인증서 `READY: True`, CloudFront 502 해소

---

## 5. 회고

### 아쉬운 점 / 다음 과제에 남길 것

- **ECR 토큰 자동 갱신 미완성** — 지금은 12시간마다 수동 갱신이 필요하다. CronJob + ServiceAccount/RBAC로 자동화하는 걸 추후 해결 과제로 남긴다.
- **인프라 매니페스트가 EC2 마스터 로컬에만 존재** — backend-app/frontend-app은 GitOps로 안전하지만, ingress/cert-manager/PLG values.yaml은 마스터 노드가 망가지면 복구할 방법이 없다. 별도 저장소로 버전 관리할 필요가 있다.
- **Secret이 Git에 평문으로 커밋됨** — `stringData`의 편의성을 택하면서 감수한 트레이드오프인데, 추가적인 보안 작업이 필요하다. 찰리가 말씀하셨던 Vault 도입을 고려할 예정이다.
- **인스턴스 사양을 사전에 가늠하지 못함** — 이번 주 문제의 상당수(flannel 크래시, Prometheus/Loki 이중 설치로 인한 자원 경합, 워커 NotReady, swap 소동)가 결국 t3.small 저사양 노드에 컨트롤플레인/애플리케이션/모니터링 스택을 전부 욱여넣은 데서 비롯됐다. 처음부터 "이 스택을 얹으려면 노드당 최소 얼마의 메모리가 필요한지"를 어림잡아 계산하고 시작했다면 여러 장애를 줄일 수 있었을 것 같다.
- **Alertmanager 규칙 부재** — 지금은 메트릭/로그를 수집만 하고 있고 실제 알림(Slack/이메일)은 구성하지 않았다. 장애 감지를 사람이 Grafana를 직접 열어봐야만 알 수 있는 상태라, 다음 단계로 알림 규칙을 붙이고 싶다.
