# 쿠버네티스 인 액션 3장

* 목차
```text
파드의 생성, 실행 정지
파드와 다른 리소스를 레이블로 조직화하기
특정 레이블을 가진 모든 파드에서 작업 수행
네임스페이스를 사용해 파드를 겹치지 않는 그룹으로 나누기
특정한 형식을 가진 워커 노드에 파드배치
```


## 1. 파드의 생성, 실행 정지
### YAML 디스크립터로 파드 생성
```bash
kubectl create -f kubia-manual.yaml
kubectl get po kubia-manual -o yaml
kubectl get po kubia-manual -o json
```
### 애플리케이션 로그 확인
```bash
kubectl logs kubia-manual
```
### 테스트 및 디버깅을 위한 포트 포워딩
* 포트포워드 자체도 해당 터미널이 유지되어야 실행되며 Ctrl+C 로 종료합니다
```bash
kubectl port-forward kubia-manual 8888:8080
curl http://localhost:8888/
```


## 2. 파드와 다른 리소스를 레이블로 조직화하기
* 라벨을 포함한 파드생성
  - metadata.labels 하위에 값을 넣을 수 있습니다
```
kubectl create kubia-manual-with-labels.yaml
kubectl get po --show-labels
```
* 라벨을 별도의 컬럼으로 출력하기
  - 이미 생성된 라벨이 creation\_method, env 인 경우
```bash
kubectl get po -L creation_method,env
```
* 기존 파드에 레이블 추가
```bash
kubectl label po kubia-manual creation_method=manual
kubectl get po --show-labels
```
* 기존에 존재하는 레이블 수정 (Overwrite)
```bash
kubectl label po kubia-manual-v2 env=debug --overwrite
```


## 3. 특정 레이블을 가진 모든 파드에서 작업 수행
* 라벨을 이용하여 필터하기
```bash
kubectl get po -L creation_method,env -l env=debug
```
* 특정 라벨 존재유무를 통한 필터
```bash
kubectl get po --show-labels -L creation_method,env -l 'env'
kubectl get po --show-labels -L creation_method,env -l '!env'
kubectl get po --show-labels -L creation_method,env -l 'creation_method != manual'
kubectl get po --show-labels -L creation_method,env -l 'env in (prod,devel)'
kubectl get po --show-labels -L creation_method,env -l 'env notin (debug)'
```

* 어노테이션 추가 및 확인
  - 어노테이션을 통해 오브젝트를 선택할 수는 없으나, 더 많은 정보를 담는다
```bash
kubectl annotate pod kubia-manual suhyuk.me/psyoblade="park.suhyuk"

kc describe pod kubia-manual | grep psyoblade
Annotations:  suhyuk.me/psyoblade: park.suhyuk
```


## 4. 네임스페이스를 사용해 파드를 겹치지 않는 그룹으로 나누기
* 네임스페이스 확인하기
  - "--namespace" 대신 -n 을 사용할 수 있다
```bash
kubectl get ns
NAME                   STATUS   AGE
default                Active   19h
kube-node-lease        Active   19h
kube-public            Active   19h
kube-system            Active   19h
kubernetes-dashboard   Active   19h

kc get po --namespace default
NAME              READY   STATUS    RESTARTS   AGE
kubia-gpu         1/1     Running   0          8m26s
kubia-manual      1/1     Running   0          55m
kubia-manual-v2   1/1     Running   0          36m
```
* 네임스페이스 생성
```bash
kubectl create -f custom-namespace.yaml
kubectl create namespace custom-namespace
```
* 특정 네임스페이스에 파드생성하기
```bash
kubectl create -f kubia-manual.yaml -n custom-namespace
kubectl get po -n custom-namespace
```
* 특정 네임스페이스를 삭제
  - 해당 네임스페이스 내에 존재하는 모든 파드도 삭제됨에 주의
  - namespace 대신 ns 를 사용해도 됩니다
```bash
kubectl delete namespace custom-namespace
```
* 특정 네임스페이스 내의 모든 파드를 삭제
```bash
kubectl delete po --all -n custom-namespace
```
* 특정 네임스페이스 내의 (거의) 모든 리소스 삭제
  - all : 모든 유형의 리소스(ReplicationController, Pod, Service 등)를 삭제
  - --all : 리소스 이름이 아닌 모든 리소스 인스턴스를 삭제할 것을 지정
  - 단, all 키워드를 사용하더라도 특정 리소스(시크릿)은 보존되어 있어 명시적으로 삭제해야만 삭제됩니다
  - *kubernetes 서비스는 삭제되어도 잠시 후 다시 생성된다*
```bash
kubectl delete all --all
```


## 5. 특정한 형식을 가진 워커 노드에 파드배치
> 파드 뿐만 아니라 모든 쿠버네티스 오브젝트는 라벨을 부착할 수 있으며, 노드 분류에도 사용할 수 있습니다
* 특정 노드에 라벨 부착하기
```bash
kubectl label node minikube gpu=true
kubectl get nodes -L gpu
```
* 레이블 셀렉터를 통해 특정 노드에 파드 스케줄링
  - spec.nodeSelector 속성에 gpu: "true" 설정을 통해 노드 선택
  - 단, 해당 라벨에 해당하는 노드가 없는 경우 Pending 상태로 빠지므로 삭제 후 다시 생성할 수 있다
```bash
kubectl create -f kubia-gpu.yaml
```
* 특정 파드를 이름으로 삭제
```bash
kubectl delete po kubia-gpu
```
* 특정 라벨을 가진 파드를 삭제
```bash
kubectl get po --show-labels
kubectl delete po -l creation_method=manual
```

