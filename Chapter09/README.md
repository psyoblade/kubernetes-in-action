# 쿠버네티스 인 액션 9장. 디플로이먼트: 선언적 애플리케이션 업데이트

* 목차
```text
파드를 최신 버전으로 교체
관리되는 파드 업데이트
디플로이먼트 리소스로 파드의 선언적 업데이트
롤링 업데이트 수행
잘못된 버전의 롤아웃 자동 차단
롤아웃 속도 제어
이전 버전으로 파드 되돌리기
```

## 1. 파드를 최신 버전으로 교체
* ReplicationController 를 통한 서비스를 기동합니다.
  - 로컬 minikube 환경에서는 서비스 연동을 위해 터널링을 수행합니다
```bash
kubectl create -f kubia-rc-and-service-v1.yaml

bash> 
minikube tunnel
❗  The service kubia requires privileged ports to be exposed: [80]
🔑  sudo permission will be asked for it.
🏃  Starting tunnel for service kubia.
Password:

curl localhost:80
This is v1 running in pod kubia-v1-82k4s
```
* 기존 파드를 모두 삭제한 다음 새 파드를 시작합니다
  - 기존의 이미지의 v1 을 v2 로 변경합니다
  - 애플리케이션을 종료하고, v2 를 다시 기동합니다
  - 터널링도 다시 기동하고, localhost 에 액세스합니다
```bash
kubectl delete -f kubia-rc-and-service-v1.yaml

bash>
cat kubia-rc-and-service-v2.yaml
...
metadata:
  name: kubia-v2
spec:
  replicas: 3
  template:
    ...
    spec:
      containers:
      - image: luksa/kubia:v2
        name: nodejs
...

kubectl create -f kubia-rc-and-service-v2.yaml

bash>
minikube tunnel
curl localhost
This is v2 running in pod kubia-v2-26qkt
```
* 새로운 파드를 시작하고, 기동하면 기존 파드를 삭제한다.
  - 2개의 애플리케이션을 기동하기 위해서는 완전히 다른 ReplicationController 를 가져야 합니다.
  - 도중에 터널링 오류가 발생하는데 다시 재시작 해주면 v1 에서 v2 로 변경됩니다
  - 터널링 이슈 때문에 순차적으로 파드를 추가 제거하는 실습은 패스합니다
```bash
kubectl delete -f kubia-rc-and-service-v1.yaml
kubectl delete -f kubia-rc-and-service-v2.yaml

kubectl get rc
NAME       DESIRED   CURRENT   READY   AGE
kubia-v1   3         3         3       84s
kubia-v2   3         3         3       24s

kubectl get po
NAME             READY   STATUS    RESTARTS   AGE
kubia-v1-fv6kg   1/1     Running   0          107s
kubia-v1-kpsd6   1/1     Running   0          107s
kubia-v1-lqw6h   1/1     Running   0          107s
kubia-v2-bj8rz   1/1     Running   0          47s
kubia-v2-prl59   1/1     Running   0          47s
kubia-v2-vf8kw   1/1     Running   0          47s

bash>
E0903 23:42:09.155548   89047 ssh_tunnel.go:127] error stopping ssh tunnel: operation not permitted
❗  The service kubia-v2 requires privileged ports to be exposed: [80]
🔑  sudo permission will be asked for it.
🏃  Starting tunnel for service kubia-v2.

bash>
curl localhost
This is v1 running in pod kubia-v1-fv6kg

bash>
minikube tunnel

curl localhost
This is v2 running in pod kubia-v2-prl59
```


## 2. 관리되는 파드 업데이트
* 서비스 되고 있는 3개의 레플리케이션 서비스가 정상화인지 확인합니다
```bash
while true; do curl localhost ; sleep 1 ; done
This is v2 running in pod kubia-v2-vf8kw
This is v2 running in pod kubia-v2-bj8rz
This is v2 running in pod kubia-v2-bj8rz
This is v2 running in pod kubia-v2-prl59
This is v2 running in pod kubia-v2-prl59

```
* 레플리케이션컨트롤러로 자동 롤링 업데이트 수행
  - 현재 v2 가 서비스 되고 있으므로 이 서비스를 v3 로 롤링업데이트 실습을 합니다
  - 최종 적으로 모든 서비스는 순차적으로 v3 로 전환되었습니다
```bash
bash>
kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP        30h
kubia-v2     LoadBalancer   10.110.74.163   127.0.0.1     80:30961/TCP   5m43s

bash>
kubectl rolling-update kubia-v2 kubia-v3 --image=luksa/kubia:v3
Command "rolling-update" is deprecated, use "rollout" instead
Created kubia-v3
Scaling up kubia-v3 from 0 to 3, scaling down kubia-v2 from 3 to 0 (keep 3 pods available, don't exceed 4 pods)
Scaling kubia-v3 up to 1
Scaling kubia-v2 down to 2
Scaling kubia-v3 up to 2


kubectl get po
NAME             READY   STATUS    RESTARTS   AGE
kubia-v2-bj8rz   1/1     Running   0          9m19s
kubia-v2-prl59   1/1     Running   0          9m19s
kubia-v2-vf8kw   1/1     Running   0          9m19s
kubia-v3-cf5df   1/1     Running   0          18s

kubectl get rc
NAME       DESIRED   CURRENT   READY   AGE
kubia-v2   3         3         3       9m24s
kubia-v3   1         1         1       24s

kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP        30h
kubia-v2     LoadBalancer   10.110.74.163   127.0.0.1     80:30961/TCP   9m27s

kubectl get rc
NAME       DESIRED   CURRENT   READY   AGE
kubia-v2   1         1         1       11m
kubia-v3   3         3         3       2m26s

...
Scaling kubia-v2 down to 0
Update succeeded. Deleting kubia-v2
replicationcontroller/kubia-v3 rolling updated to "kubia-v3"

get rc
NAME       DESIRED   CURRENT   READY   AGE
kubia-v3   3         3         3       3m28s

bash>
minikube tunnel

while true ; do curl localhost ; sleep 1 ; done
This is v3 running in pod kubia-v3-rr6xc
This is v3 running in pod kubia-v3-ss6lz
This is v3 running in pod kubia-v3-rr6xc
This is v3 running in pod kubia-v3-cf5df
This is v3 running in pod kubia-v3-cf5df
```
* 롤링 업데이트가 완료된 이후의 커네이너 상태를 확인합니다
  - v2 의 ReplicationController 를 그대로 복사하고
  - 해당 파드 템플릿에서 이미지를 변경해 새로운 ReplicationController 를 생성합니다
```bash
bash>
kubectl describe rc kubia-v3

Name:         kubia-v3
Namespace:    default
Selector:     app=kubia-v2,deployment=c5ad0a0f056d2391b1c1ff35a4c11c9d
Labels:       app=kubia-v2
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=kubia-v2
           deployment=c5ad0a0f056d2391b1c1ff35a4c11c9d
  Containers:
   nodejs:
    Image:        luksa/kubia:v3
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                    Message
  ----    ------            ----   ----                    -------
  Normal  SuccessfulCreate  6m7s   replication-controller  Created pod: kubia-v3-cf5df
  Normal  SuccessfulCreate  5m1s   replication-controller  Created pod: kubia-v3-ss6lz
  Normal  SuccessfulCreate  3m55s  replication-controller  Created pod: kubia-v3-rr6xc
```
* 롤링 업데이트 시에 상세한 로그를 남길 수 있습니다
  - "--v 6" 가 가장 상세한 로깅을 남기는 옵션입니다
```bash
kubectl rolling-update kubia-v2 kubia-v3 --image=luksa/kubia:v3 --v 6
```



## 3. 디플로이먼트 리소스로 파드의 선언적 업데이트
## 4. 롤링 업데이트 수행
## 5. 잘못된 버전의 롤아웃 자동 차단
## 6. 롤아웃 속도 제어
## 7. 이전 버전으로 파드 되돌리기

