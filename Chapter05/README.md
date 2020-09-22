# 쿠버네티스 인 액션 5장 - 서비스: 클라이언트가 파드를 검색하고 통신을 가능하게 함
> 파드는 언제든 교체 변경될 수 있는 '일시적'인 특성, 파드는 노드에 스케줄링 되기 직전에 IP 주소가 할당되는 점 그리고 각 파드는 IP 주소를 통해서 액세스될 수 있어야 한다는 점을 해결하기 위해 '서비스' 라는 리소스를 제공합니다

* 목차
```text
단일 주소로 파드를 노출하는 서비스 리소스 만들기
클러스터 안에서 서비스 검색
외부 클라이언트에 서비스 노출
클러스터 내에서 외부 서비스 접속
파드가 서비스할 준비가 됐는지 제어하는 방법
서비스 문제 해결
```

## 1. 단일 주소로 파드를 노출하는 서비스 리소스 만들기
## 2. 클러스터 안에서 서비스 검색
## 3. 외부 클라이언트에 서비스 노출
## 4. 클러스터 내에서 외부 서비스 접속
## 5. 파드가 서비스할 준비가 됐는지 제어하는 방법
## 6. 서비스 문제 해결


## 레퍼런스
* [Create static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
* [Multi-container pods and container communication in Kubernetes](https://www.mirantis.com/blog/multi-container-pods-and-container-communication-in-kubernetes/)
* [Full Example of nginx.conf](https://www.nginx.com/resources/wiki/start/topics/examples/full/)


## 5.1 서비스 소개
> 동일한 서비스를 제공하는 파드 그룹에 지속적인 단일 접점을 제공할 때에 생성하는 리소스이며, 각 서비스는 서비스가 존재하는 동안 절대 변경되지 않는 IP, PORT 가 있습니다
![kia.5.1](images/kia.5.1.png)
> 그림과 같이 외부 서비스가 프론트엔드를 찾기 위한 서비스(IP-1) 그리고 프론트엔드가 백엔드를 찾기 위한 서비스(IP-2)가 생성되고 고정된 IP 주소가 노출되며, 개별 프론트엔드 파드의 IP 주소가 변경되더라도 해당 서비스의 IP 는 변경되지 않습니다.

### 5.1.1 서비스 생성
> 해당 서비스는 레이블 셀렉터 (Label selector)를 통해 자신이 어떤 파드를 담당해야 하는지 결정할 수 있습니다
> '서비스'를 생성하는 가장 쉬운 방법은 ```kubectl expose``` 명령어입니다. 이는 모든 파드를 단일 IP 주소와 PORT 로 노출시킵니다.
![kia.5.2](images/kia.5.2.png)
* 서비스를 생성하기 위한 설정파일을 만듭니다
  - 라벨이 'kubia' 인 파드의 8080 포트를 80 포트로 라우팅 하는 서비스를 생성합니다
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```
* 간단한 웹서버와 이를 연결하는 서비스를 구성합니다
  - 1. 서비스를 생성하고, 내부 IP 할당이 되었는지 확인합니다
  - 2. 책에서는 별도의 파드생성이 없기 때문에 4장에서 사용했던 kubia-rc.yaml 파일을 이용하여 파드를 생성합니다
  - 3. 생성된 서비스에 대해 정상여부를 확인하기 위해 ```kubectl exec``` 명령어를 통해 curl 커맨드를 이용합니다
  - 명령어의 더블 대시(--)는 kubectl 명령줄 옵션의 끝을 의미합니다. 즉, -- 이후의 모든 문자열은 파드 내에서 실행되는 명령어 이며, 명령줄 내에 대시로 시작하는 인수가 없다면 더블대시를 사용할 필요는 없습니다
```bash
bash> # 1. 서비스를 생성합니다
kubectl create -f kubia-svc.yaml
kubectl get all  # 내부 IP 가 10.102.166.80 으로 할당되어 있음을 확인할 수 있습니다

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   15d
service/kubia        ClusterIP   10.102.166.80   <none>        80/TCP    2m29s


bash> # 2. ReplicationController 파드를 생성
kubectl create -f kubia-rc.yaml
replicationcontroller/kubia created


bash> # 3. 임의의 파드에 대해서 curl 명령을 통해 자신과 서비스에 대해서도 테스트해 봅니다

kubectl get po -o=custom-columns=NAME:.metadata.name
NAME
kubia-6b9wj
kubia-pmzwn
kubia-txc9c

kubectl exec kubia-6b9wj -- curl -s http://localhost:8080
You've hit kubia-6b9wj

for x in $(seq 1 5) ; do kubectl exec kubia-6b9wj -- curl -s http://10.102.166.80 ; sleep 0.5 ; done
You've hit kubia-txc9c
You've hit kubia-txc9c
You've hit kubia-txc9c
You've hit kubia-pmzwn
You've hit kubia-txc9c
```
* 명령어를 수행하는 과정에 대한 설명
![kia.5.3](images/kia.5.3.png)
* 서비스의 세션 어피니티 구성
> 동일한 ClientIP 에 대해서 동일한 파드로 서비스를 제공할 수 있도록 리다이렉트 할 수 있도록 SessionAffinity 속성을 None(Default) -> ClientIP 로 설정합니다
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-clientip
spec:
  sessionAffinity: ClientIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia-client-ip
```
* 세션 어피니티 속성을 주고 테스트합니다
  - 쿠버네티스는 None 과 ClientIP 두 가지의 서비스 세션 어피니티만 지원합니다
  - 쿠키 기반의 세션 어피니티는 지원하지 않는데 이는 쿠버네티스가 HTTP 수준에서(L7) 동작하지 않기 때문입니다
  - 즉, L4 레이어 수준에서 TCP 와 UDP 패킷을 처리하고 페이로드(payload)는 신경쓰지 않습니다
```bash
bash> # 동일한 파드로만 전송됨을 확인합니다
for x in $(seq 1 10) ; do kubectl exec kubia-6b9wj -- curl -s http://10.111.2.56 ; sleep 0.5 ; done
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7
You've hit kubia-clientip-vhmx7

kubectl get svc kubia-clientip -o json | grep sessionAffinity
```
* 좀 더 단순한 예제를 위해 정적인 파드 설정파일을 생성합니다
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-named
spec:
  containers:
  - name: kubia-named
    image: luksa/kubia
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
```
* 해당 파드를 바라보는 서비스를 구성합니다
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-named
spec:
  containers:
  - name: kubia-named
    image: luksa/kubia
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
```
* [TODO] 동일한 서비스에서 여러 개의 포트 노출 <-- ***실습이 제대로 이루어지지 않아 다시 확인해 보아야 하는 부분***
  - 파드 하나에 여러개의 포트(http:80, https:443)를 노출하는 경우에도 하나의 서비스로 멀티포트를 지원할 수 있습니다
```bash
bash>
kubectl create -f kubia-svc-named-ports.yaml
service/kubia-named created

kubectl create -f kubia-po-named.yaml
pod/kubia-named created

```

### 5.1.2 서비스 검색
> 환경변수를 이용하는 방법과, DNS를 이용한 방법으로 IP 와 PORT 를 확인할 수 있습니다

* 환경변수를 이용하는 방법
```bash
kubectl exec kubia-named env
...
KUBIA_NAMED_SERVICE_HOST=10.98.203.142
KUBERNETES_SERVICE_HOST=10.96.0.1
...
```

* DNS를 이용하는 방법

* FQDN을 통한 서비스 연결
  - kubia 는 서비스 이름이고, svc.cluster.local 은 로컬 서비스 이름에 사용되는 클러스터의 도메인 접미사입니다
  - 같은 네임스페이스에 있는 경우 svc.cluster.local 과 namespace 는 생략이 가능하여 kubia 로 접근이 가능합니다
```bash
bash>
kubectl exec -it kubia-4pkgl bash
root@kubia-4pkgl:/# curl http://kubia
You've hit kubia-x586c

root@kubia-4pkgl:/# curl http://kubia.default.svc.cluster.local
You've hit kubia-x586c

root@kubia-4pkgl:/# curl http://kubia.default
You've hit kubia-x586c

# 파드 컨테이너 내부의 DNS resolver 구성을 확인할 수 있습니다 (쿠버네티스는 각 컨테이너의 /etc/resolv.conf 파일을 수정해서 적용합니다)
root@kubia-4pkgl:/# cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```
* 서비스 IP에 핑을 할 수 없는 이유
  - 서비스의 클러스터의 IP가 *가상 IP* 이므로 서비스 포트와 결합된 경우에만 의미가 있습니다
  - 즉, 파드(동일한 IP) 내에 여러 컨테이너 서비스를 포워드 할 수도 있기 때문에 IP+PORT 를 통해서 실제 컨테이너의 IP를 알 수 있기 때문입니다


## 5.2 클러스터 외부에 있는 서비스 연결
> 외부 요청이 클러스터 내부에 연결하는 여태까지의 서비스 기능과 반대로, 외부에 존재하는 서비스를 내부에서 접근할 수 있도록 외부 IP 와 PORT 로 연결을 전달할 수 있습니다

### 5.2.1 서비스 엔드포인트 소개
> 서비스는 파드와 직접 연결되지 않으며, 엔드포인트 리소스를 통해 연결됩니다. 그리고 파드 셀렉터는 엔드포인트를 생성할 때에 사용되는 질의문이라고 볼 수 있습니다
* 서비스의 상세정보와 엔드포인트 정보를 확인합니다
```bash
bash> 
kubectl describe service kubia
Name:              kubia
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=kubia  # 엔드포인트 목록 조회에 사용되며, 컨테이너 접근시에 직접 활용되지 않습니다
Type:              ClusterIP
IP:                10.99.90.252
Port:              <unset>  80/TCP
TargetPort:        8080/TCP
Endpoints:         172.18.0.5:8080,172.18.0.6:8080,172.18.0.7:8080  # 엔드포인트의 IP:PORT
Session Affinity:  None
Events:            <none>


kubectl get endpoints kubia  # 엔드포인트는 항상 복수입니다
NAME    ENDPOINTS                                         AGE
kubia   172.18.0.5:8080,172.18.0.6:8080,172.18.0.7:8080   29m

```




## 9. 질문과 답변

### 9.1 [PING 은 몇 번 PORT 일까?](https://m.blog.naver.com/ssamba/125695893)
> 예전 부터 궁금해 했던 부분인데 좋은 블로그가 있어 참고 하였습니다. PING 은 ICMP(Internet Control Message Protocol) 라는 프로토콜을 기반으로 작성된 프로그램인데 OSI 7 Layer 상에서 IP/PORT 를 통해 전송 되는 Layer 는 Trasport 즉 4 Layer 이나, ICMP 는 3 Layer 에 해당하며 PORT 와는 무관합니다. 
* [TCP/IP의 분해와 IP Address의 이해](https://m.blog.naver.com/ssamba/125712071)도 참고하면 좋겠습니다

