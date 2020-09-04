# 쿠버네티스 인 액션 4장

* 목차
```text
파드의 안정적인 유지
동일한 파드의 여러 인스턴스 실행
노드 장애 시 자동으로 파드 재스케줄링
파드의 수평 스케줄링
각 클러스터 노드에서 시스템 수준의 파드 실행
배치 잡 실행
잡을 주기적 또는 한 번만 실행하도록 스케줄링
```


## 1. 파드의 안정적인 유지
* HTTP 기반의 Liveness Probe 를 생성합니다
  - 책에서는 1분 30초라고 되어 있으나 아래와 같이 Pulling 에 8분 가량 소요되어 그 이후에 Running 이 되었다
```bash

kubectl create -f kubia-liveness-probe.yaml
kubectl get po
kubectl describe po kubia-liveness

Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  8m4s              default-scheduler  Successfully assigned default/kubia-liveness to minikube
  Normal   Pulling    7m58s             kubelet, minikube  Pulling image "luksa/kubia-unhealthy"
  Normal   Pulled     82s               kubelet, minikube  Successfully pulled image "luksa/kubia-unhealthy"
  Normal   Created    81s               kubelet, minikube  Created container kubia
  Normal   Started    81s               kubelet, minikube  Started container kubia
  Warning  Unhealthy  1s (x3 over 21s)  kubelet, minikube  Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    1s                kubelet, minikube  Container kubia failed liveness probe, will be restarted

```
* 주기적으로 계속 확인해 보면 Terminated 된 상태를 확인할 수 있습니다
  - "--previous" 옵션을 추가해 이전의 로그까지 logs 명령으로 학인할 수 있습니다
  - 종료 코드가 137번이며 = 128 + x 로써 SIGKILL (9) 으로 종료된 것을 확인할 수 있습니다
  - 128 숫자는 docker 종료 시에 반환하는 숫자라서 그 숫자와의 합으로 표현합니다 
```bash
kubectl describe po kubia-liveness
kubectl logs kubia-liveness --previous
...
    Last State:     Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Thu, 03 Sep 2020 16:18:59 +0900
      Finished:     Thu, 03 Sep 2020 16:20:49 +0900
    Ready:          True
    Restart Count:  1
    Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
...
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  10m                   default-scheduler  Successfully assigned default/kubia-liveness to minikube
  Normal   Pulling    105s (x2 over 10m)    kubelet, minikube  Pulling image "luksa/kubia-unhealthy"
  Normal   Pulled     102s (x2 over 3m36s)  kubelet, minikube  Successfully pulled image "luksa/kubia-unhealthy"
  Normal   Created    102s (x2 over 3m35s)  kubelet, minikube  Created container kubia
  Normal   Started    102s (x2 over 3m35s)  kubelet, minikube  Started container kubia
  Warning  Unhealthy  25s (x6 over 2m35s)   kubelet, minikube  Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    25s (x2 over 2m15s)   kubelet, minikube  Container kubia failed liveness probe, will be restarted
```
* 초기에 지연시간을 주기 위해 initialDelaySeconds 값을 지정할 수 있습니다
  - 이는 기동 시에 오래 걸리는 애플리케이션의 경우 liveness 지연 시간을 주지 않는 경우 위와 같은 원인으로 파드가 종료될 수 있습니다
  - 운영 중인 파드는 반드시 라이브니스 프로브를 정의해야만 쿠버네티스가 애플리케이션이 살아있음을 알 수 있습니다
  - 더 나은 라이브니스 프로브를 우해서 특정 URL (/health) 를 구성하는 것이 유용하며, 반드시 엔드포인트 인증 필요여부를 체크합니다
  - 애플리케이션 내부만 체크해야지, 외부 의존 서비스에 대한 문제를 해당 애플리케이션에서 고려해서는 안된다 (데이터베이스 접근)
  - 최대한 가볍게 1초 이내에 수행될 수 있도록 구성하며, 재시도하는 것은 의미가 없습니다 (여러번 해도 쿠버네티스는 1번실패로 간주)
  - 라이브니스 프로브는 해당 쿠블렛 수준의 영역이므로, 애플리케이션이 다른 노드에 수행되도록 하는 것은 RC 등의 다른 메커니즘을 통해 파드를 관리해야 합니다
```bash
kubectl create -f kubia-liveness-probe-initial-delay.yaml
```


## 2. 동일한 파드의 여러 인스턴스 실행
> 노드 장애 시에 해당 노드에서 수행되던 파드 가운데 리플리케이션 컨트롤러에 의해 관리되는 경우에만 다른 노드에서 수행됩니다
* 레플리케이션컨트롤러의 3가지 요소
  - label selector : 라벨을 통해 관리 대상 파드를 선택
  - replica count : 파드의 레플리카 수
  - pod template : 새로운 파드 레플리카를 만들 때에 사용하는 붕어빵 틀
```bash
kubectl create -f kubia-rc.yaml
```
* 레플리케이션이 잘 관리되는지 일부 파드를 삭제합니다
  - 리플리케이션 컨트롤러의 상태도 같이 확인합니다
```bash
kubectl get po -l app=kubia
kubectl delete po kubia-7hjkc

NAME             READY   STATUS             RESTARTS   AGE
kubia-7hjkc      0/1     Terminating        0          5m8s
kubia-9w7gs      1/1     Running            0          34s
kubia-f4d5l      1/1     Running            0          5m8s
kubia-r9xtd      1/1     Running            0          5m8s

kubectl get rc
```
* 상세한 수행 내역은 describe 를 통해 확인합니다
  - 특정 파드가 삭제 혹은 종료가 새로운 파드의 생성을 트리거링 하는 것이 아닙니다.
  - 리플리케이션컨트롤러는 현재 상태의 비정상을 확인하고 파드의 상태를 유지하는 것입니다
```bash
kc describe rc kubia

Name:         kubia
Namespace:    default
Selector:     app=kubia
Labels:       app=kubia
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=kubia
  Containers:
   kubia:
    Image:        luksa/kubia
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                    Message
  ----    ------            ----   ----                    -------
  Normal  SuccessfulCreate  6m37s  replication-controller  Created pod: kubia-r9xtd
  Normal  SuccessfulCreate  6m37s  replication-controller  Created pod: kubia-7hjkc
  Normal  SuccessfulCreate  6m37s  replication-controller  Created pod: kubia-f4d5l
  Normal  SuccessfulCreate  2m3s   replication-controller  Created pod: kubia-9w7gs
```

## 3. 노드 장애 시 자동으로 파드 재스케줄링
> 로컬 Minikube 의 경우 하나의 노드만 존재하므로 노드 장애를 실습하기는 어렵습니다
* 구글 클라우드 명령(gcloud compute ssh)으로 특정 노드에 접속 후 네트워크 인터페이스를 종료(sudo ifconfig eth0 down)합니다
  - "-o wide" 옵션으로 파드를 조회해, 적어도 하나 이상 실행 중인 노드를 선택합니다
  - 네트워크 문제 뿐만 아니라 장비의 장애의 경우에도 유사하게 스스로 치유됩니다
```bash
gcloud compute ssh
sudo ifconfig eth0 down

kubectl get node
kubectl get pods
```
* 레플리케이션컨트롤러 범위 안팎으로 파드 이동하기 (featured by label)
  - 파드의 레이블을 변경하는 경우 파드가 사라진 것으로 인지하고 레플리케이션컨트롤러는 레플리카 수를 유지하기 위해 파드를 생성하며
  - 레플리케이션컨트롤러의 레이블 셀렉터와 일치하지 않게 만들어야만 합니다
```bash
kubectl get po --show-labels
kubectl label po kubia-r9xtd type=special

kubectl label po kubia-9w7gs app=foo --overwrite
kubectl get po --show-labels
```
* 레플리케이션컨트롤러를 직접 편집기로 수정할 수 있습니다
  - 환경 변수에서 KUBE\_EDITOR 값으로 편집기 지정이 가능합니다
```bash

export KUBE_EDITOR = "/usr/bin/vim"

kubectl edit rc kubia
labels.app : kubia -> foo
spec.selector.pp : kubia -> foo

kubectl get po --show-labels
NAME          READY   STATUS    RESTARTS   AGE     LABELS
kubia-8zw5n   1/1     Running   0          62s     app=foo
kubia-9w7gs   1/1     Running   0          24m     app=foo
kubia-f4d5l   1/1     Running   0          28m     app=kubia
kubia-f8d86   1/1     Running   0          7m23s   app=kubia
kubia-mdj5s   1/1     Running   0          62s     app=foo
kubia-r9xtd   1/1     Running   0          28m     app=kubia,type=special
```


## 4. 파드의 수평 스케줄링
* 레플리카수를 변경하는 것은 파드를 변경하는 것이 아니므로 쉽게 적용이 가능합니다
  - 마찬가지로 kubectl edit rc 명령을 통해서 선언적인 수정도 가능합니다
```bash
kubectl scale rc kubia --replicas=10
```
* 레플리케이션 컨트롤러의 삭제 시에 포함된 모든 파드도 같이 삭제됩니다
  - "--cascade=false" 옵션을 통해서 파드는 살려둘 수도 있습니다
```bash
kubectl delete rc kubia
kubectl get po

NAME          READY   STATUS        RESTARTS   AGE
kubia-8zw5n   1/1     Terminating   0          6m56s
kubia-9w7gs   1/1     Terminating   0          30m
kubia-f4d5l   1/1     Running       0          34m
kubia-f8d86   1/1     Running       0          13m
kubia-mdj5s   1/1     Terminating   0          6m56s
kubia-r9xtd   1/1     Running       0          34m
```


## 5. 각 클러스터 노드에서 시스템 수준의 파드 실행
* 레플리카셋을 통한 파드 생성
  - 책에서는 "apps/v1beta2"을 사용하고 있으나 2020년 현재 정식 릴리스가 되어 beta2 를 제거해야 정상 동작합니다
  - ReplicationController 의 경우 spec.selector.app : kubia 형식이지만,
  - ReplicaSet 의 경우 spec.selector.matchLabels.app : kubia 형식으로 지정하는 부분만 다릅니다
  - apiVersion 속성은 "apiGroup/apiVersion" 형식으로 지정할 수 있으며 여기서는 "apps/v1" 으로 정의되어 있습니다
```bash
kubectl create -f kubia-replicaset.yaml
kubectl get rs
```
* ReplicaSet 을 파일로 생성하는 경우 라벨이 동작하지 않는다
  - 왜 그런지 모르겠지만, -f 통해 생성후 get -o yaml 로 확인하면 해당 정보가 존재하지 않습니다
  - 명시적으로 라벨을 지정해 주어야 추가되는 것을 확인할 수 있었습니다
```bash
kubectl create -f kubia-replicaset.yaml
kubectl label rs kubia app=kubia
```
* ReplicaSet 에서는 matchLabels 보다 더 강력한 matchExpressions 를 이용할 수 있습니다
  - In : 지정된 값 중 하나와 일치
  - NotIn : 지정된 값과 일치하지 않아야 함
  - Exists : 지정된 키를 가진 레이블이 포함되어야 한다 (값은 중요하지 않으며, 값 필드를 지정하지 않아야 한다)
  - DoesNotExist : 지정된 키를 가진 레이블이 포함되지 않아야 한다 (값 필드를 지정하지 않아야 한다)
```bash
kubectl create -f kubia-replicaset-matchexpressions.yaml

cat kubia-replicaset-matchexpressions.yaml
spec:
  replicas: 3
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
         - kubia

kubectl delete rs kubia
```
* 데몬셋으로 모든 노드에 파드 실행하기
```bash
kubectl create -f ssd-monitor-daemonset.yaml

cat ssd-monitor-daemonset.yaml
spec:
  ...
  template:
    ...
    spec:
      nodeSelector:
        disk: ssd
```
* 처음에는 생성되지 않을 것이며 별도로 minikube 에 라벨을 추가해야 파드가 생성됩니다
```bash
kubectl get po
kubectl label node minikube disk=ssd
kubectl describe po ssd-monitor-hkpq4
```
* 생성 후 해당 노드의 라벨을 제거 혹은 변경하는 경우 어떻게 되나?
```bash
kubectl label node minikube disk=hdd --overwrite
kubectl get po
NAME                READY   STATUS        RESTARTS   AGE
ssd-monitor-hkpq4   1/1     Terminating   0          2m30s
```


## 6. 배치 잡 실행
> ReplicationController, ReplicaSet, DaemonSet 등은 완료되지 않는 지속적인 태스크이지만, 때로는 완료 가능한 배치 성의 completable task 도 존재하며, 이를 "잡 리소스"이라고 말합니다
* 별도의 배치 잡을 생성하여 실행합니다
  - 잡 리소스에 대한 상태를 확인할 수 있습니다
  - 책에서는 "--show-all -a" 옵션을 소개하지만 현재 버전은 지정하지 않아도 완료된 잡을 출력합니다
  - 파드 명을 통해서 로그를 확인할 수 있습니다 ()
```bash
kubectl create -f exporter.yaml
kubectl get po
NAME              READY   STATUS    RESTARTS   AGE
batch-job-chwnr   1/1     Running   0          73s

kubectl get jobs
NAME        COMPLETIONS   DURATION   AGE
batch-job   0/1           80s        80s

...
kubectl get jobs
NAME        COMPLETIONS   DURATION   AGE
batch-job   1/1           2m5s       2m27s

kubectl get po
NAME              READY   STATUS      RESTARTS   AGE
batch-job-chwnr   0/1     Completed   0          2m44s

kubectl describe jobs batch-job
...
Events:
  Type    Reason            Age    From            Message
  ----    ------            ----   ----            -------
  Normal  SuccessfulCreate  3m11s  job-controller  Created pod: batch-job-chwnr
  Normal  Completed         66s    job-controller  Job completed

kubectl logs batch-job-chwnr
Thu Sep  3 13:35:47 UTC 2020 Batch job starting
Thu Sep  3 13:37:47 UTC 2020 Finished succesfully
```
* 잡에서 여러 파드 인스턴스 실행하기
  - 두 개 이상의 파드 인스턴스를 생성하여 순차 혹은 병렬처리를 할 수 있습니다
```bash
kubectl create -f multi-completion-batch-job.yaml
kubectl get jobs

NAME                         COMPLETIONS   DURATION   AGE
batch-job                    1/1           2m5s       9m3s
multi-completion-batch-job   0/5           14s        14s
```
* 동일한 이름의 잡 파드를 실행할 수 없으므로 이름을 변경하여 순차/병렬 잡을 동시에 수행합니다
  - metadata.name: parallel-completion-batch-job 와 같이 변경합니다
  - 잡은 2개가 다 떠 있음을 확인할 수 있었고, 파드는 순차의 경우 1개만 Running 이지만, 병렬은 2개가 Running 입니다
```bash
kubectl create -f multi-completion-parallel-batch-job.yaml

kubectl get jobs
NAME                            COMPLETIONS   DURATION   AGE
multi-completion-batch-job      1/5           2m50s      2m50s
parallel-completion-batch-job   0/5           13s        13s

kubectl get po
NAME                                  READY   STATUS      RESTARTS   AGE
multi-completion-batch-job-9rrv4      0/1     Completed   0          2m53s
multi-completion-batch-job-hk2ff      1/1     Running     0          49s
parallel-completion-batch-job-dnb6l   1/1     Running     0          16s
parallel-completion-batch-job-gdnlj   1/1     Running     0          16s

kubectl describe jobs multi-completion-batch
Events:
  Type    Reason            Age    From            Message
  ----    ------            ----   ----            -------
  Normal  SuccessfulCreate  6m50s  job-controller  Created pod: multi-completion-batch-job-9rrv4
  Normal  SuccessfulCreate  4m46s  job-controller  Created pod: multi-completion-batch-job-hk2ff
  Normal  SuccessfulCreate  2m42s  job-controller  Created pod: multi-completion-batch-job-274gg
  Normal  SuccessfulCreate  38s    job-controller  Created pod: multi-completion-batch-job-6ff52

kubectl describe jobs parallel-completion-batch-job
...
Events:
  Type    Reason            Age    From            Message
  ----    ------            ----   ----            -------
  Normal  SuccessfulCreate  3m33s  job-controller  Created pod: parallel-completion-batch-job-gdnlj
  Normal  SuccessfulCreate  3m33s  job-controller  Created pod: parallel-completion-batch-job-dnb6l
  Normal  SuccessfulCreate  89s    job-controller  Created pod: parallel-completion-batch-job-2c87n
  Normal  SuccessfulCreate  87s    job-controller  Created pod: parallel-completion-batch-job-chsh9

```
* 잡이 실행되는 도중에 parallelism 을 변경할 수도 있습니다
  - 책에서는 동작하는 것처럼 보이나 로컬 환경에서는 아래와 같은 오류가 발생했습니다
```bash
kubectl scale job multi-completion-batch-job --replicas=3
Error from server (NotFound): the server could not find the requested resource
```
* 여태까지 수행한 모든 작업을 삭제합니다
```bash
kubectl delete jobs --all
```


## 7. 잡을 주기적 또는 한 번만 실행하도록 스케줄링
* 크론잡 생성을 통해 주기적인 실행
  - 크론잡은 여전히 beta 이므로 batch/v1beta1 을 유지해야 합니다
  - 작업 삭제시에 크론잡과 잡을 모두 삭제해야 완전히 삭제가 됩니다
```bash
kubectl create -f cronjob.yaml
cronjob.batch/batch-job-every-fifteen-minutes created

kubectl get crontjobs
NAME                              SCHEDULE             SUSPEND   ACTIVE   LAST SCHEDULE   AGE
batch-job-every-fifteen-minutes   0,15,30,45 * * * *   False     0        <none>          79s

kubectl get po
NAME                                               READY   STATUS      RESTARTS   AGE
batch-job-every-fifteen-minutes-1599141600-nmkqc   0/1     Completed   0          15m
batch-job-every-fifteen-minutes-1599142500-s72bt   1/1     Running     0          28s

kubectl delete jobs --all
kubectl delete crontjobs --all
```

