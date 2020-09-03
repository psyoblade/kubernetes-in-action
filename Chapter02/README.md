# 쿠버네티스 인 액션 2장

## 1. Kubia 웹 서버 띄우기
> 초기 설정을 위한 실습을 수행합니다. 

### 약식으로 레플리케이션 컨트롤러 띄우기
* 레플리케이션 컨트롤러를 정의해서 수행도 가능하지만, 약식으로 커맨드라인 실행은 아래와 같이 --generator 를 이용할 수 있습니다"
```bash
kubectl run kubia --image=luksa/kubia --port=8080 --generator=run/v1
kubectl describe rc kubia
```

### 외부 접근을 위한 로드밸런서 띄우기
* 개별 파드는 IP를 가지지만 컨테이너 내부에서만 사용되며, 외부 접속을 위해서는 반드시 LoadBalancer 유형의 서비스가 필요합니다
  - 단, minikube 혹은 kubeadm 의 경우 Custom Kubernetes Cluster 가 아니므로 NodePort 나 Ingress Controller 를 써야합니다.
  - [minikube 환경에서 tunnel 을 통한 localhost 수행](https://stackoverflow.com/questions/44110876/kubernetes-service-external-ip-pending)
```bash
kubectl expose rc kubia --type=LoadBalancer --name kubia-http
minikube tunnel
```

### 레플리카 수 조정하기
* 임의의 레플리카 수를 적용하기
```bash
kubectl scale rc kubia --replicas=3
kubectl get rc
kubectl get pods
```
* 모든 컨테이너가 호출을 받는지 확인해 봅니다
```bash
while true; do curl 127.0.0.1:8080; sleep 1 ; done

kubectl describe pods kubia-444nc | grep Node
Node:         minikube/172.17.0.2
```
### 대시보드를 통해 상태 확인하기
* 대시보드 확인 및 실행
  - [Kubernetes Dashboard](http://localhost:60190/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/overview?namespace=default)
```bash
minikube dashboard
```

### 모든 작업이 완료 되었다면 컨트롤러 및 파드 삭제하기
```
kubectl delete -f kubia.yaml
```
