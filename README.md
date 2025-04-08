## GitOps with Jenkins & ArgoCD

> **Jenkins와 ArgoCD를 통해 구축해보는 GitOps 기반 CI / CD 파이프라인**

본 저장소는 Jenkins와 ArgoCD를 활용해 구축한 GitOps CI / CD 파이프라인에 대한 Helm 차트 저장소이다.

## Description

### GitOps

GitOps는 Kubernetes의 특징인 코드형 인프라(Infrastructure as Code, IaC)와 Git의 형상 관리를 결합하여 클라우드 네이티브 환경을 효율적으로 관리하는 DevOps의 구현 방법 중 하나이다.<br />
Kubernets의 매니페스트(Manifest)들을 GitHub 레포지토리에 저장해 인프라에 대한 버전 관리가 가능하게 된다.<br />
게다가 Terraform 등의 프로비저닝(Provisioning) 도구까지 함께 사용하면 클라우드부터 클라우드 내의 애플리케이션 배포까지 Git을 통해 관리하고 언제든지 복구할 수 있게 된다.

### CI / CD

일반적으로 GitOps는 Kubernetes의 지속적 배포 도구 중 하나인 ArgoCD를 활용해 구현할 수 있다.<br />
이때, 매니페스트들의 관리를 편리하게 하기 위해 Helm을 함께 사용했다.

다음 절차는 ArgoCD에 CI / CD 도구인 Jenkins를 함께 사용해 구축한 GitOps CI / CD 파이프라인이다.

![](https://github.com/user-attachments/assets/16f6921f-92f9-4adc-aa06-3e407be69dfe)

1. 애플리케이션 레포지토리의 변경사항을 감지한 Jenkins가 해당 레포지토리를 새로 가져옴.
2. 업데이트된 애플리케이션의 새로운 이미지 빌드
3. 만들어진 이미지를 컨테이너 레지스트리에 게시
4. 차트 레포지토리 내의 해당하는 이미지 태그를 새로 게시한 이미지의 태그로 수정
5. 차트 레포지토리의 변경사항을 감지한 ArgoCD가 해당 레포지토리를 새로 가져옴.
6. 업데이트된 차트를 Kubernetes 클러스터에 배포
7. kubelet이 해당 차트에 맞는 새로운 이미지를 가져와 컨테이너 구축 및 실행

이때, 클러스터에 문제가 발생해 롤백(Rollback)을 수행하고 싶다면 단순히 차트 레포지토리에 대해 `git revert`를 수행하면 된다.<br />
이렇게 인프라에 대해 버전 관리를 수행할 수 있고 이러한 버전 관리를 개발자들에게 친숙한 형상 관리 도구인 Git으로 한다는 점이 GitOps의 큰 장점 중 하나이다.

## How to use

```sh
> git clone https://github.com/earlgrey02/gitops-chart
> cd gitops-chart
> helm dependencies update 
> helm install gitops .
```

본 레포지토리를 통해 Nginx Ingress Controller, Jenkins, ArgoCD를 Helm으로 한 번에 구축할 수 있다.

```sh
> kubectl get service -n ingress-nginx
NAME                              TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
gitops-ingress-nginx-controller   NodePort   10.103.83.33   <none>        80:32748/TCP,443:30083/TCP   13m
```

차트 내의 Nginx Ingress Controller는 `NodePort` 타입을 사용하므로 해당하는 포트와 함께 각각 `/jenkins`, `/argocd`로 접속하면 Jenkins와 ArgoCD의 대시보드에 접속할 수 있다.<br />
Jenkins와 ArgoCD는 모두 `admin`과 `root`를 아이디, 패스워드로 가지고 있다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: {{ index .Values "argo-cd" "namespaceOverride" }}
type: Opaque
stringData:
  discord-url: <Webhook URL>
```

ArgoCD는 `example-application.yaml`을 `Application`으로 가지고 있는데 해당 `Application`이 동기화될 때마다 알림이 가도록 설정되어 있다.<br />
만약 실제로 알림을 받고 싶다면 위 `secret.yaml`에 실제 Discord의 웹훅(Webhook) URL을 작성하면 된다.
