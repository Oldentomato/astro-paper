---
  author: jowoosung
  pubDatetime: 2025-10-14T04:19:50.812Z
  modDatetime: 2025-10-14T04:19:51.079Z
  title: oauth-proxy 미들웨어 구성
  slug: oauth-proxy-미들웨어-구성
  featured: true
  draft: false
  tags:
    - infra
  description: 개인화 사이트진입을 위한 google기반 oauth-proxy 미들웨어 구성법
---

## Table of contents

## 개요  
기존에 개발한 markdown-blog-uploader 웹을 개인서버의 k3s환경을 통해 배포했었다.  
하지만 이 웹은 자동으로 git push되기 때문에 아무나 들어오면 안된다. 그러기에 로그인 기능을 추가하여 보안을 강화하기로 했다.  
조사해보니, oauth2-proxy라는 미들웨어가 있었다. 이 미들웨어를 웹 앞에 구성해놓으면 내 웹에 접속하기전에 oauth2-proxy가 요청을 가로채고 인증을 위한 웹화면으로 리디렉션된다.
그 다음, 인증이 완료되면 다시 이동되어야할 웹화면으로 다시 리디렉션되는 구조이다. 아키택쳐는 다음과 같다.  
![이미지 설명](https://miro.medium.com/v2/resize:fit:720/format:webp/1*JBVCjQGg7ydTreYFnNsUXQ.png)  
이 다음부터 구성법에 대해 설명하겠다.  

## oauth-proxy 설치  
### helm 설치  
우선 helm을 통해 oauth-proxy를 설치해준다.  
```bash
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests

helm repo update

helm upgrade --install oauth2-proxy \
  oauth2-proxy/oauth2-proxy --version "7.12.6" \
  -f ./custom_values.yaml --namespace o11y --create-namespace
```
여기서 custom_values.yaml이 보이는데 여기에 세부 설정을 해준다.  

```yaml
config:
  existingSecret: oauth2-secret
  configFile: |
    provider = "<인증할 서비스 예:google>"
    authenticated_emails_file = "<허용 url 리스트 txt파일 path>"
    cookie_secure = true
    upstreams = ["<인증 후 다시 리다이렉트할 url>"]
    redirect_url = "<인증 리다이렉트 url>"
    pass_authorization_header = true
    set_authorization_header = true
    cookie_name = "_oauth2_proxy"
    cookie_refresh = "2m"
    cookie_expire = "24h"
    whitelist_domains = ["<도메인 화이트리스트 예:kro.kr>"]
    set_xauthrequest = true
    skip_auth_regex = "<특정 url이 포함 시 인증에서 제외>" 

extraArgs:
  - --reverse-proxy
  - --skip-provider-button

extraVolumes:
  - name: auth-emails
    configMap:
      name: oauth2-proxy-auth-emails

extraVolumeMounts:
  - name: auth-emails
    mountPath: /etc/oauth2-proxy

```
여기에 보면 아래에 extraVolumes와 extraVolumeMounts가 보이는데 authenticated_emails_file = "<허용 url 리스트 txt파일 path>" 을 위한 설정들이다. 이 txt파일에 허용할 email들을 입력하면 된다. 그 설정 명령어는

```bash
kubectl create configmap oauth2-proxy-auth-emails \
  --from-literal=authenticated-emails.txt="jwsjws99@gmail.com" \
  -n o11y
```
위와 같다. 우선 위 명령어로 configmap을 생성하고 oauth2-proxy를 설치하는 것을 추천한다.  

### yaml 설정  
다음 3개의 yaml을 kubectl apply를 통해 설정해준다.  

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: forward-auth
  namespace: o11y
spec:
  forwardAuth:
    address: http://oauth2-proxy.o11y.svc.cluster.local/oauth2/
    trustForwardHeader: true
    authResponseHeaders:
      - "X-Auth-Request-User"
      - "X-Auth-Request-Email"
      - "Authorization"
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: oauth2-proxy-ingress
  namespace: o11y
  annotations:
    traefik.ingress.kubernetes.io/auth-type: basic
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  ingressClassName: traefik 
  rules:
    - host: <리다이렉트 url>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: oauth2-proxy
                port:
                  number: 80
  tls:
  - hosts:
      - <리다이렉트 url>
    secretName: cert-manager-staging
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: oauth2-secret
  namespace: o11y
type: Opaque
stringData:
  client-id: "<google client id>"
  client-secret: "<google secret key>"
  cookie-secret: "<랜덤 생성 base64키>"
```
여기에 랜덤 생성 base64키가 있는데 아래 코드를 이용해서 키 생성해서 넣어도 상관없다.  
```python
import os
import base64

# 32바이트(=256비트) 랜덤 키 생성
random_key = os.urandom(32)

# Base64 인코딩
key_base64 = base64.b64encode(random_key).decode()

print("랜덤 키 (Base64):", key_base64)
```
그리고 현재 google 관련 키들이 있는데 발급방법은 다음 링크에서 따라하면 된다.  

[google oauth api 발급](https://min-0.tistory.com/entry/OAuth2-%EC%86%8C%EC%85%9C-%EB%A1%9C%EA%B7%B8%EC%9D%B8-Google-API-%ED%82%A4-%EB%B0%9C%EA%B8%89-%EB%B0%A9%EB%B2%95)  

마지막으로 실제로 운영하는 웹의 ingress에 이 annotation을 추가해주면 된다.  
```yaml
annotations:
  ...
  traefik.ingress.kubernetes.io/router.middlewares: "o11y-forward-auth@kubernetescrd"
  ...
```

## tls 설정  
위 yaml에서 ingress에 cert-manager.io/cluster-issuer: letsencrypt-staging가 있는 것을 볼 수 있다.  
google oauth는 리다이렉션 사이트를 무조건 https로 되어있어야 인증이 된다고 한다.  
그러기에 tls설정을 해주어야 한다.  
### cert manager 설치  
우선 helm을 통해 cert manager를 설치해준다.  
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.1/cert-manager.yaml
```

### yaml 설정  
다음 3개의 yaml을 kubectl apply 하여 설정해주면 tls설정은 끝난다.
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cert-manager-staging
  namespace: <서비스의 네임스페이스>
spec:
  secretName: cert-manager-staging # 여기는 아무거나 적어도 된다고 하지만 Certificate의 name과 동일하게 하면 헷갈리지 않는다.
  issuerRef:
    name: letsencrypt-staging # cluster issuer 이름 적기
    kind: ClusterIssuer
  dnsNames:
    - <지정할 웹 url>
```
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # server: https://acme-v02.api.letsencrypt.org/directory
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: <사용할 email>
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: <사용하고 있는 proxy class 이름 예:traefik>
```
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: post-ingress-route
  namespace: <서비스의 네임스페이스>
spec:
  entryPoints:
    - websecure
    - web
  routes:
  - match: Host(`<tls를 설정할 웹 url>`)
    kind: Rule
    services:
    - name: <해당 웹의 서비스명>
      port: 80
  - match: Host(`<tls를 설정할 웹 url>`)
    kind: Rule
    services:
    - name: <해당 웹의 서비스명>
      port: 4000
  tls:
    secretName: cert-manager-staging # certificate에서 설정한 secretName과 동일한 이름

```
ClusterIssuer부분을 보면 server: https://acme-staging-v02.api.letsencrypt.org/directory 로 되어있고 이름도 -staging이 붙어있다.  
이는 실제 acme 인증서발급이 아니라 테스트를 위한 임시 인증서 발급이다. 실제 인증서를 발급하려면 위 주석을 해제하고 아래거는 지운 다음 모든 곳에 -staging이라고 붙은 것들을 지워준다.  

### 참고 사항  
위 내용처럼 가짜 인증서를 발급하여 테스트하고자 하려면 맨 위에 cert-manager를 설치할 때 yaml부분에서  
cookie_secure = false 
로 바꿔주면 임시적으로 잘 동작시킬 수 있다.  