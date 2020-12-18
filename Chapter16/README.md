# 쿠버네티스 인 액션 16장 고급 스케줄링
> 

- 목차
  * [1. 테인트와 톨러레이션을 사용한 특정 노드에서 파드 실행 제한](#1-테인트와-톨러레이션을-이용한-파드실행-제한)
  * [2. 노드 어피니티를 사용해 파드를 특정 노드로 유인하기](#2-노드-어피니티를-이용한-노드유인)
  * [3. 파드 어피니티와 안티 어피니티를 이용해 파드 함께 배치하기](#3-파드-어피니티와-안티어피티니를-이용한-파드배치)



## 1 테인트와 톨러레이션을 이용한 파드실행 제한
> 노드 테인트와 파드 톨러레이션을 통해 *어떤 파드가 특정 노드를 사용할 수 있는지를 제한*할 때에 사용합니다. 즉 노드의 테인트가 허용된(tolerate) 경우에만 스케줄링 될 수 있습니다. 기존의 *노드 셀렉터나 노드 어피니티 규칙*의 경우 특정 정보를 파드에 추가해서 파드가 명시적으로 스케줄링 되는 노드를 구분하는 반면, 테인트는 기존 파드를 수정하지 않고, 노드에 테인트를 추가함으로써 특정 파드가 특정 노드에 배포되지 않도록 하는 기법입니다. (하위 객체인 파드를 손데지 않고, 상위 객체인 노드에 색깔을 부여함으로써 좀 더 우아한 파드 실행 제한이 가능한 것 같다)

### 16.1.1 테인트와 톨러레이션
> **테인트(Taint)**는 특정 노드에 제약(Constraints)을 정하여 원하지 않는 파드가 배포되지 않도록 하기 위함이며, 반대로 파드의 톨러레이션(Toleration)은 특정 노드의 테인트에 대해 가능한지 여부를 지정할 수 있습니다. 예를 들어, 특정 노드에 반드시 특정 파드를 수행해야 하는 경우에 Taint 와 Toleration 을 같이 부여해야만 가능한 경우입니다.
> 즉, 테인트는 노드에 부여하고, 톨러레이션은 파드에 부여합니다

### 16.1.2 노드에 사용자 정의 테인트 추가하기
```bash
bash> kubectl taint node gke-kubia-default-pool-694e5d84-5h7t node-type=production:NoSchedule

bash> cat deployment-wo-toleration.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80

bash> kubectl create -f deployment-wo-toleration.yaml
bash> kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE                                   NOMINATED NODE   READINESS GATES
nginx-deployment-574b87c764-4wpkh   1/1     Running   0          76s   10.84.0.11   gke-kubia-default-pool-694e5d84-6ch3   <none>           <none>
nginx-deployment-574b87c764-55wrs   1/1     Running   0          76s   10.84.2.5    gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
nginx-deployment-574b87c764-jjc57   1/1     Running   0          76s   10.84.2.4    gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>

bash> kubectl get nodes
NAME                                   STATUS   ROLES    AGE   VERSION
gke-kubia-default-pool-694e5d84-5h7t   Ready    <none>   30m   v1.16.15-gke.4300
gke-kubia-default-pool-694e5d84-6ch3   Ready    <none>   30m   v1.16.15-gke.4300
gke-kubia-default-pool-694e5d84-j8hg   Ready    <none>   30m   v1.16.15-gke.4300
```

### 16.1.3 파드에 톨러레이션 추가
```bash
bash> cat deployment-w-toleration.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-w-toleration
  labels:
    app: prod
spec:
  replicas: 5
  selector:
    matchLabels:
      app: prod
  template:
    metadata:
      labels:
        app: prod
    spec:
      containers:
      - args:
        - sleep
        - "99999"
        image: busybox
        name: main
      tolerations:
      - key: node-type
        operator: Equal
        value: production
        effect: NoSchedule

bash> kubectl create -f deployment-w-toleration.yaml
bash> kubectl get pods -o wide
NAME                                    READY   STATUS    RESTARTS   AGE    IP           NODE                                   NOMINATED NODE   READINESS GATES
busybox-w-toleration-78b9c548f7-8kt5p   1/1     Running   0          108s   10.84.2.7    gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
busybox-w-toleration-78b9c548f7-d84c6   1/1     Running   0          108s   10.84.2.6    gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
busybox-w-toleration-78b9c548f7-k72fv   1/1     Running   0          108s   10.84.1.3    gke-kubia-default-pool-694e5d84-5h7t   <none>           <none>
busybox-w-toleration-78b9c548f7-mpxm7   1/1     Running   0          108s   10.84.1.2    gke-kubia-default-pool-694e5d84-5h7t   <none>           <none>
busybox-w-toleration-78b9c548f7-vpbcb   1/1     Running   0          108s   10.84.0.12   gke-kubia-default-pool-694e5d84-6ch3   <none>           <none>
```

### 16.1.4 테이트와 톨러레이션의 활용 방안 이해
> 테인트는 키와 효과만 갖고 있어도 제약을 걸 수 있으므로, 값을 꼭 필요로 하지 않는다. 반면 톨러레이션은 Equal 혹은 Exists 연산자를 이용하여 테인트 키에 여러 값을 허용할 수도 있습니다
* 테인트는 
  - 스케줄링을 방지 : NoSchedule
  - 선호하지 않는 노드를 정의하고 : PrefereNoSchedule
  - 노드에서 기존 파드를 제거한다 : NoExecute
* 노드 실패 후 파드를 재스케줄링하기 까지 시간
  - [Taint based Evictions](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/#taint-based-evictions) 기준에 따르면 Kubernetes v1.18 부터는 **node controller 가 특정 상황(not-ready, unreachable 등)이 되는 경우 자동으로 taints** 하게 되어 아래의 경우에 eviction 될 수 있습니다
```yaml
  tolerations:
  - effect: NoSchedule
    key: node-type
    operator: Equal
    value: production
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
```

## 2 노드 어피니티를 이용한 노드유인
> 여태까지의 테인트는 파드를 특정 노드에서 떨어뜨려 놓는 데 사용되었습니다. "노드 어피니티"라는 메커니즘은 **특정 노드 집합에만 파드를 스케줄링하도록 지시**할 수 있습니다.

* 노드 셀렉터는 항상 해당 레이블을 포함시켜야 하는 불편함이 있는데, 노드 어피니티는 선호도만 명시하면 해당 노드들에 스케줄링하려고 시도하게 됩니다
  - 노드 어피니티는 노드 셀렉터와 같은 방식으로 레이블을 기반으로 노드를 선택합니다.
  - 노드의 라벨 가운데 중요한 레이블(region, zone, hostname 등)을 눈여겨 봅니다
  - 이미 3장에서 노드의 라벨을 활용하여 특정 노드에만 파드를 배포하는 것은 실습하였습니다
```bash
bash> kubectl describe node <gke-node-name>
Name:               gke-kubia-default-pool-694e5d84-5h7t
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=g1-small
                    beta.kubernetes.io/os=linux
                    cloud.google.com/gke-nodepool=default-pool
                    cloud.google.com/gke-os-distribution=cos
                    failure-domain.beta.kubernetes.io/region=asia-northeast3
                    failure-domain.beta.kubernetes.io/zone=asia-northeast3-a
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=gke-kubia-default-pool-694e5d84-5h7t
                    kubernetes.io/os=linux
```

### 16.2.1 하드 노드 어피니티 규칙 지정
* 노드 셀렉터를 사용하는 파드
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - image: luksa/kubia
    name: kubia
```
* 노드 어피티니 규칙을 사용하는 파드
  - [kia.16.2](images/kia.16.2.png)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  affinity:
    nodeAffinity:
      # requiredDuringScheduling ... : 파드가 노드로 스케줄링 시에 필요한 레이블
      # ... IgnoredDuringExecution   : 이미 실행 중인 파드는 영향을 주지 않음
      requiredDuringSchedulingIgnoredDuringExecution:
        # gpu=true 인 라벨이 붙은 노드에만 스케줄링 됩니다
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
  containers:
  - image: luksa/kubia
    name: kubia
```

### 16.2.2 파드의 스케줄링 시점에 노드 우선순위 지정
> 여기까지는 기존의 노드 셀렉터와 큰 차이가 없으나, preferredDuringScheduling... 을 통해 파드의 스케줄링 시에 우선순위를 지정할 수 있는 기법을 학습합니다

[kia.16.3](images/kia.16.3.png)

* 선호하는 zone 과 machine 을 지정하지만, 여유 노드가 있다면 스케줄링 되지만, 그렇지 않다고 하더라도 문제는 아닌 경우
  - availability-zone 에 가중치 80, shre-type 에 가중치 20 으로 모두 노드1에 배포되기를 기대했으나, 4개는 1개의 노드에 나머지는 2번째 노드에 배포 되었습니다
  - 스케줄러가 노드의 위치를 지정할 때에 Selector-SpreadPriority 기능을 통해 특정 노드에만 배포되면 장애 시에 위험할 수 있으므로 여러 노드에 분산하는 우선순위가 적용되기 때문입니다
```bash
bash> kubectl label node gke-kubia-default-pool-694e5d84-5h7t availability-zone=zone1
bash> kubectl label node gke-kubia-default-pool-694e5d84-5h7t share-type=dedicated
bash> kubectl label node gke-kubia-default-pool-694e5d84-6ch3 availability-zone=zone2
bash> kubectl label node gke-kubia-default-pool-694e5d84-6ch3 share-type=shared
bash> kubectl get nodes -L availability-zone -L share-type
NAME                                   STATUS   ROLES    AGE    VERSION             AVAILABILITY-ZONE   SHARE-TYPE
gke-kubia-default-pool-694e5d84-5h7t   Ready    <none>   2d1h   v1.16.15-gke.4300   zone1               dedicated
gke-kubia-default-pool-694e5d84-6ch3   Ready    <none>   2d1h   v1.16.15-gke.4300   zone2               shared
gke-kubia-default-pool-694e5d84-j8hg   Ready    <none>   2d1h   v1.16.15-gke.4300


bash> kubectl create -f preferred-deployment.yaml
bash> k get po -o wide
NAME                                    READY   STATUS    RESTARTS   AGE    IP           NODE                                   NOMINATED NODE   READINESS GATES
preferred-deployment-6585cd5654-4kdxj   1/1     Running   0          6m7s   10.84.2.11   gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
preferred-deployment-6585cd5654-6frzl   1/1     Running   0          6m7s   10.84.0.13   gke-kubia-default-pool-694e5d84-6ch3   <none>           <none>
preferred-deployment-6585cd5654-6jr6x   1/1     Running   0          6m7s   10.84.2.10   gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
preferred-deployment-6585cd5654-7kwls   1/1     Running   0          6m7s   10.84.2.8    gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
preferred-deployment-6585cd5654-zkdtv   1/1     Running   0          6m7s   10.84.2.9    gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
```

* 이번에는 우리의 가정이 적절한지 확인을 하기위해 10개의 레플리카로 변경합니다
  - 6개는 
```bash
bash> kubectl create -f preferred-deployment-v2.yaml
bash> kubectl get po -o wide
NAME                                       READY   STATUS    RESTARTS   AGE   IP           NODE                                   NOMINATED NODE   READINESS GATES
preferred-deployment-v2-5564f595fb-26b65   1/1     Running   0          19s   10.84.2.13   gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
preferred-deployment-v2-5564f595fb-5hsgg   1/1     Running   0          19s   10.84.0.14   gke-kubia-default-pool-694e5d84-6ch3   <none>           <none>
preferred-deployment-v2-5564f595fb-652lj   1/1     Running   0          19s   10.84.2.15   gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
preferred-deployment-v2-5564f595fb-6wt98   1/1     Running   0          19s   10.84.2.16   gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
preferred-deployment-v2-5564f595fb-7pdrk   1/1     Running   0          19s   10.84.2.14   gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
preferred-deployment-v2-5564f595fb-8zxzd   0/1     Pending   0          19s   <none>       <none>                                 <none>           <none>
preferred-deployment-v2-5564f595fb-fjxgv   0/1     Pending   0          19s   <none>       <none>                                 <none>           <none>
preferred-deployment-v2-5564f595fb-k56hj   0/1     Pending   0          19s   <none>       <none>                                 <none>           <none>
preferred-deployment-v2-5564f595fb-l7ckw   1/1     Running   0          19s   10.84.2.12   gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>
preferred-deployment-v2-5564f595fb-rgn6t   1/1     Running   0          19s   10.84.2.17   gke-kubia-default-pool-694e5d84-j8hg   <none>           <none>

bash> kubectl label node gke-kubia-default-pool-694e5d84-j8hg availability-zone=zone3
bash> kubectl label node gke-kubia-default-pool-694e5d84-j8hg share-type=shared
bash> kubectl get nodes -L availability-zone -L share-type
NAME                                   STATUS   ROLES    AGE    VERSION             AVAILABILITY-ZONE   SHARE-TYPE
gke-kubia-default-pool-694e5d84-5h7t   Ready    <none>   2d2h   v1.16.15-gke.4300   zone1               dedicated
gke-kubia-default-pool-694e5d84-6ch3   Ready    <none>   2d2h   v1.16.15-gke.4300   zone2               shared
gke-kubia-default-pool-694e5d84-j8hg   Ready    <none>   2d2h   v1.16.15-gke.4300   zone3               shared
```

* g1-small 스펙이 안 좋아서 그런지, 제대로 파드배치가 안 되는 느낌이라 재생성
  - 아무런 라벨을 명시하지 않고 파드를 배포한 경우 잘 분산 된다
```bash
bash> k get po -o wide | awk '{ print $7 }' | sort  | uniq -c
   1 NODE
   3 gke-kubia-default-pool-001c8246-3jw4
   3 gke-kubia-default-pool-001c8246-6hjr
   4 gke-kubia-default-pool-001c8246-zljb
```

* 이번에는 라벨을 붙이고 다시 배포 테스트
  - 결과는 이전과 큰 차이 없이 리소스 기준으로 배포된 것 처럼 보입니다
```bash
bash>
k get node -L availability-zone -L share-type
NAME                                   STATUS   ROLES    AGE   VERSION             AVAILABILITY-ZONE   SHARE-TYPE
gke-kubia-default-pool-001c8246-3jw4   Ready    <none>   25m   v1.16.15-gke.4901   zone1               dedicated
gke-kubia-default-pool-001c8246-6hjr   Ready    <none>   25m   v1.16.15-gke.4901   zone2               shared
gke-kubia-default-pool-001c8246-zljb   Ready    <none>   25m   v1.16.15-gke.4901   zone2               shared

k get po -o wide | awk '{ print $7 }' | sort  | uniq -c
   1 NODE
   4 gke-kubia-default-pool-001c8246-3jw4
   3 gke-kubia-default-pool-001c8246-6hjr
   3 gke-kubia-default-pool-001c8246-zljb
```

* Preferred 설정은 동작하지 않아 Required 설정으로 테스트
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: required-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      app: required
  template:
    metadata:
      labels:
        app: required
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: availability-zone
                  operator: In
                  values: 
                  - zone1
      containers:
      - args:
        - sleep
        - "99999"
        image: busybox
        name: main
```
* 테스트 및 실습은 잘 동작하며, 일부는 Pending 상태로 남았습니다
```bash
bash> k create -f requred-deployment.yaml
bash> k get po -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP           NODE                                   NOMINATED NODE   READINESS GATES
required-deployment-66ddb79f7b-2m6gt   1/1     Running   0          2m58s   10.84.1.15   gke-kubia-default-pool-001c8246-3jw4   <none>           <none>
required-deployment-66ddb79f7b-2xljs   1/1     Running   0          2m58s   10.84.1.13   gke-kubia-default-pool-001c8246-3jw4   <none>           <none>
required-deployment-66ddb79f7b-dwxqz   0/1     Pending   0          2m57s   <none>       <none>                                 <none>           <none>
required-deployment-66ddb79f7b-jxmmj   1/1     Running   0          2m57s   10.84.1.12   gke-kubia-default-pool-001c8246-3jw4   <none>           <none>
required-deployment-66ddb79f7b-mb9qj   1/1     Running   0          2m58s   10.84.1.14   gke-kubia-default-pool-001c8246-3jw4   <none>           <none>

```



## 3 파드 어피니티와 안티어피티니를 이용한 파드배치

### 16.3.3
```bash
bash> k create -f backend-busybox.yaml
bash> k create -f frontend-podaffinity-host.yaml
bash> k get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE                                   NOMINATED NODE   READINESS GATES
backend-7c78f5b858-js2d6    1/1     Running   0          72s   10.84.2.10   gke-kubia-default-pool-001c8246-6hjr   <none>           <none>
backend-7c78f5b858-mj98d    1/1     Running   0          72s   10.84.0.14   gke-kubia-default-pool-001c8246-zljb   <none>           <none>
frontend-6b7cf99bb5-4sk25   1/1     Running   0          14s   10.84.2.11   gke-kubia-default-pool-001c8246-6hjr   <none>           <none>
frontend-6b7cf99bb5-h7fpd   1/1     Running   0          14s   10.84.0.15   gke-kubia-default-pool-001c8246-zljb   <none>           <none>
```

### 16.3.4 파드 안티-어피니티를 사용하여 파드들이 서로 떨어지게 스케줄링하기
> 두 파드가 동일한 노드에 실행되는 경우 서로 성능에 영향을 주는 경우 (IO를 많이 사용하는 파드와 CPU 사용많은 파드를 섞는게 좋다. 그 반대의 경우는 Anti-Pattern)에 사용합니다

* 안티-어피니티를 이용하여 프론트엔드와 떨어진 노드에 배포하기, 펜딩은 되어도 같이는 못 살겠다 !!
```bash
bash> k create -f frontend-podantiaffinity-host.yaml
bash> k get po -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE                                   NOMINATED NODE   READINESS GATES
frontend-6b7cf99bb5-4sk25        1/1     Running   0          35m   10.84.2.11   gke-kubia-default-pool-001c8246-6hjr   <none>           <none>
frontend-6b7cf99bb5-h7fpd        1/1     Running   0          35m   10.84.0.15   gke-kubia-default-pool-001c8246-zljb   <none>           <none>
frontend-anti-79f9d8d6f5-66qqz   1/1     Running   0          27s   10.84.1.17   gke-kubia-default-pool-001c8246-3jw4   <none>           <none>
frontend-anti-79f9d8d6f5-6lbfn   0/1     Pending   0          27s   <none>       <none>                                 <none>           <none>
frontend-anti-79f9d8d6f5-762s4   1/1     Running   0          27s   10.84.1.18   gke-kubia-default-pool-001c8246-3jw4   <none>           <none>
frontend-anti-79f9d8d6f5-n82n2   1/1     Running   0          27s   10.84.1.16   gke-kubia-default-pool-001c8246-3jw4   <none>           <none>
frontend-anti-79f9d8d6f5-vsvhq   1/1     Running   0          27s   10.84.1.19   gke-kubia-default-pool-001c8246-3jw4   <none>           <none>
```


## 4. 실습해보고 싶은 내용
* 노드 3개에 골고루 busybox 를 배포하고, 특정 노드에만 NoExecute Taint 를 추가하면 어떻게 되는가?
* Taint & Tolerance 의 경우 특정 Node 에 Pod 를 배포되어야 하거나, 되지 말아야 하는 경우
  - Taint 는 Label 이 아니라 Key, Value, Effect 로 구분된 별도의 메타정보입니다
  - 아직 준비되지 않은 노드 혹은 응답이 없는 노드에 배포하지 말아야 하는 경우
  - 프로덕션 노드에 개발 파드가 배포되지 않도록, LIVE, RC, DEV 등을 구분해야 하는 경우
* Node Affinity 통한 
  - Node Affinity 는 Label 에 지정된 값을 통해 특정 노드로 파드를 유인합니다
  - Region, Zone, Hostname 등의 사전에 정의된 라벨을 통하여 노드를 선택할 수 있습니다
  - preferredDuringScheduling 과 같이 강제하지 않을 수도 있고, requiredDuringScheduling 과 같이 강제할 수도 있습니다
* Pod Affinity 와 Anti Affinity 는 파드 간의 같이 혹은 떨어진 상태로 배포를 합니다
  - Pod 간에 서로 같이 배포 혹은 떨어뜨려서 배포해야 하는 경우 사용 (어플리케이션의 성능, 튜닝에 가까운 설정)
  - Topology Key 를 활용하여 Rack 수준에서의 배포도 가능합니다 (이 또한 어플리케이션의 네트워크 성능을 올리는 것이므로, 튜닝에 가까운 설정)
  - Taint 와 Toleration 은 시스템 관리자가 설정해 두는 영역이고, Affinity 와 Anti-affinity 는 개발자가 설정하는 영역으로 보인다


