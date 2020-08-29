---
layout: post
title: socket.io 서버 컨테이너 환경으로 구축하기      # 페이지 최상단 타이틀
author: albireoS2                  # authors.yml 에 작성한 본인의 이
thumbnail: thumbnail/socket-io-k8s.png    # thumbnail
permalink: /socket-io-k8s       # staging url 로 공유할 링크. 원하는대로 입력
staging: false                       # staging 페이지의 경우, staging true 옵션을 !!!무조건!!! 줄것.
---

socket.io 기반 웹소켓 서버를 오토스케일링 가능한 컨테이너 환경으로 구성하면서 
발생하는 많은 이슈들과 그 해결 과정들을 소개해드리겠습니다.

## 배경
네이버 스포츠 라이브 중계 서비스에는 실시간 이벤트 전송을 위한 socket.io 푸쉬 서버가 있습니다.
원래 물리장비 6대로 가동하고 있던 이 서버에는 아주 많은 비효율이 발생하고 있었습니다.

1. 물리서버의 cpu 및 리소스는 매우 풍부한데 비하여, 서버별 네트워크 커넥션 개수의 한계로 인해 제 성능을 발휘하지 못하고 있었습니다. 
-  작은 리소스로 여러대의 서버 생성 하는게 효율적.

2. 경기가 중계중일때는 동시에 10만명 이상 접속을 하지만, 경기가 없을 때는 말그대로 '노는' 서버가 됩니다. 
- 트래픽에 증가와 감소에 따라 오토스케일링필요.

위 두가지 이유로 socket.io 푸쉬 서버의 컨테이너화 및 오토스케일링 환경 구성을 진행하게 되었습니다.


<br>


### 0. 초기 셋팅
1. 순간 트래픽 보다는 최대한 많은 동시 커넥션을 안정적으로 유지하는게 목표였기에, reverse proxy를 통한 websocket 연결시 [port 고갈 이슈](https://making.pusher.com/ephemeral-port-exhaustion-and-how-to-avoid-it/)가 생기는 nginx 를 빼고 [nodeJs express](https://expressjs.com/ko/) 로만 직접 서비스 하도록 결정했습니다. 

2. CPU 자원 대비 최대한 많은 동시커넥션을 받기위해 [pm2 cluster](https://pm2.keymetrics.io/docs/usage/cluster-mode/) 환경을 구성하였습니다.
`1 cluster = 1 CPU` 로 1 Pod 당 cluster 개수를 바꿔가며 실험한 결과 
1 Pod 당 4 cluster  일때 가장 효율이 좋았습니다.

이렇게 아래 Pod 구조로 서비스를 구성하고, 오토스케일링을 포함한 쿠버네티스 환경을 셋팅하였습니다.
![image](https://media.oss.navercorp.com/user/7829/files/f827c580-a9cc-11ea-9853-ed3a7c9a913c)

<br>

### 1. Pod 이 줄어들 때 -> 대규모 리커넥션 트래픽
오토스케일링을 설정하게 되면 요청량에 따라 Pod 개수가 늘어나거나 줄어듭니다. 
Pod 개수가 줄어든다는 건, Pod의 '종료'를 의미합니다.

일반적인 웹서버는 가동중인 일부 서버가 종료되어도 큰 지장이 없지만, 
socket.io 서버의 경우 종료할때에 기존에 연결되어있던 웹소켓들을 전부 Disconnect 시키게 됩니다.

보통 사용자가 사용중인 서비스를 유지하기 위해 리커넥션 설정을 해놓는데요.

예를 들면 현재 15만명이 서비스에 동시접속 중일때, 현재 Pod 이 3개인 상태라고 가정합니다.
사용자가 12만명으로 줄어서 Pod 2개 만으로도 서비스할 수 있는 상태가 되었습니다.
그래서 Pod 1개를 종료시키게 되면, 해당 Pod 의 웹소켓 커넥션 4만개가 일시에 끊어지고,
4만명 사용자의 자바스크립트에서 동시에 리커넥션을 시도하게 됩니다.

**이렇게 되면 다른 서버에게 엄청난 순간 트래픽이 가해지기 때문에, 리커넥션 요청이 잘 분산되도록 설정해야합니다.**

[socket.io client](https://socket.io/docs/client-api/#new-Manager-url-options) 에서 리커넥션 관련 설정들이 있어 20초에 걸쳐 리커넥션 요청이 분산되도록 설정했습니다. 
```js
	var connectionOptions = {
		"force new connection": true,
		"timeout": 5000,
		"transports": ["websocket"],
		"reconnection": true,
		"reconnectionAttempts": 10,
		"reconnectionDelay": 10000,
		"reconnectionDelayMax": 20000,
		"randomizationFactor": 1 
	};
```

서버측에서도 갖고 있던 소켓들을 천천히 Disconnect 완료 후 종료하도록 설정합니다.
```js
// Graceful Shutdown
const sigs = ['SIGINT', 'SIGTERM', 'SIGQUIT'];
sigs.forEach(sig => {
	process.on(sig, () => {
		socketMap.values().forEach((socket, index) => {
			setTimeout(() => socket.disconnect(true), index); // 1ms 간격으로 disconnect
		});
                ....
	})
});
```

<br>

### 2. Pod 이 늘어날때 -> 기존 Pod의 커넥션이 옮겨가지 않음
HTTP 웹서버는 오토스케일링 되어 Pod이 늘어나면 트래픽이 바로 분산되지만, 
socket.io 서버는 새로운 Pod이 추가되더라도 기존 웹소켓 연결이 계속 유지되고, 신규 요청만 분산되게 됩니다.

예를 들면 socket.io 서버 Pod 1대가 1만 커넥션을 안정적으로 서비스 할 수 있다고 가정합니다.
최초에 Pod 1대가 있고, 트래픽이 유입되어 1만 커넥션이 넘으면, Pod 하나가 더 추가됩니다.
이후 1만 커넥션이 더 들어오면, 이 1만 커넥션은 두대의 Pod에 분산되는데요,
결과적으로 처음 Pod은 1만5천 커넥션, 두번째 Pod은 5천 커넥션이 됩니다.

이런 식으로 Pod 이 계속 늘어나게 되면 최초 생성된 Pod은 당초 예상보다 훨씬 많은 커넥션을 맺게 되고, 이로 인한 장애 상황이 발생하게 됩니다. 

가장 먼저 생각한 방법은 로드밸런서를 least connection 룰로 설정하는 것이었습니다.
쿠버네티스 [ kube-router](https://github.com/cloudnativelabs/kube-router)에 least connection scheduler 를 구현하여 Service 오브젝트 에서 사용할 수 있습니다.
```kube-router.io/service.scheduler: "wlc"```
그러나 난이도가 높고, Connection 이 가장 적은 Pod 에 트래픽이 몰리는 위험성이 있어 다른 방법을 찾았습니다.

다음 방법으로는 커넥션이 일정 개수를 넘어가는 경우, 리커넥션 시켜버리는 방법입니다.
자료구조에 connection을 저장하고, 정해진 개수를 초과하는 경우, 수 초에 걸쳐 리커넥션하도록 설정하였습니다. 리커넥션 방법은 아래와 같이 직접 disconnect 후 자동으로 리커넥션되는 설정을 활용하였습니다.

```js
	if (socketMap.count() > RECOMMENDED_CONNECTION_SIZE) {
		socketMap.values().slice(-CONNECTION_RECONNECT_BUFFER_SIZE).forEach((socket, index) => {
			setTimeout(() => socket.disconnect(true), index * 5); // 5ms 간격으로 요청
		});
	}
```
단순 리커넥션의 경우, 일부 요청이 원래 서버로 다시 들어온다는 단점이 있습니다만, 
성능 테스트를 통해 평시 Pod 개수, buffer 사이즈 및 리커넥션 주기를 잘 잡으면, 
Pod 들 간에 커넥션을 충분히 효과적으로 분산시킬 수 있었습니다.


<br>

### 3. CPU 수치 보다 더 정확한 오토스케일링 metric 필요.
오토스케일링의 기준이 되는 metric 중 가장 쉽고 많이 쓰는 것은 CPU 입니다. 
그러나 socket.io 서버에서 CPU metric 으로 오토스케일링을 하는 경우, Pod 이 종료될 때 큰 트래픽이 발생하는 현상 때문에, 줄어든 Pod 이 다시 늘어나는 등 필요하지 않은 오토스케일링이 자주 일어나는 문제가 발생하게 됩니다. 

현재 Pod 이 맺고있는 커넥션 숫자를 metric으로 하면, 일시적인 트래픽이 아닌, 실제 사용자 수 에 기반한 안정적을 오토스케일링을 할 수 있게 됩니다.
 [socket.io-prometheus-metrics](https://www.npmjs.com/package/socket.io-prometheus-metrics) 에서는 아래와 같은 metric 제공하고 있는데, socket_io_connected 수치가 원하는 커넥션 수와 일치하여 이를 수집하였습니다.

`{socket-io서버도메인}:9090/metrics`
```
# HELP socket_io_connected Number of currently connected sockets
# TYPE socket_io_connected gauge
socket_io_connected 8285

# HELP socket_io_connect_total Total count of socket.io connection requests
# TYPE socket_io_connect_total counter
socket_io_connect_total 21958809

# HELP socket_io_disconnect_total Total count of socket.io disconnections
# TYPE socket_io_disconnect_total counter
socket_io_disconnect_total 21950529

# HELP socket_io_events_received_total Total count of socket.io recieved events
# TYPE socket_io_events_received_total counter
socket_io_events_received_total{event="joinByGameId"} 21455228
socket_io_events_received_total{event="visibilityChange"} 17064979

# HELP socket_io_errors_total Total socket.io errors
# TYPE socket_io_errors_total counter
socket_io_errors_total 0
```

``` yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: socket-io-push
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: socket-io-push-200710-132807
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: socket_io_connected
      target:
        type: AverageValue
        averageValue: 10000
```

이를 기반으로 socket.io 관련 Grafana 모니터링 페이지도 커스터마이징할 수 있었습니다.

![image](https://media.oss.navercorp.com/user/7829/files/70d96100-aa63-11ea-879d-c98e96afeeb9)

<br>

- **cluster 환경에서의 매트릭 수집**
위 metric 은 사실 4개 cluster 중 랜덤하게 하나의 수치를 나타내고 있는데요, 
PM2에서 round-robin 을 잘 해주기 때문에, metric 수집 주기의 4배수(cluster 개수)로 평균을 내면 적당히 쓸만한 그래프를 얻을 수 있습니다.
대략적인 평균치가 아닌, 클러스터들의 metric 을 정확히 합산, 평균 내고 싶다면 [prom-client](https://github.com/siimon/prom-client) 기반으로 exporter 를 만들어볼 수 있을것 같습니다.

<br>

### 4. 네트워크(L3)  단에서의 connection 제한
성능테스트에서 80만 이상 동시접속 가능함을 확인하였음에도 불구하고, 
리얼 배포후 모니터링에서 커넥션이 12만개로 제한되는 것이 발견되었습니다.
이후 확인하여 보니 네트워크 단에서 커넥션 개수 제한이 있었습니다.

수십만개의 websocket 을 유지한다는게 꽤나 특이한 상황이라, 
커넥션 수과 관련된 모든 접점에서 확인해볼 필요가 있을 것 같습니다. 


<br><br>


## 성능테스트
위 이슈들을 해결하고 socket.io 서버에 대한 최종적인 성능테스트를 진행하였는데요, 
참고 하실 만한 부분이 있을듯 하여 간단하게 진행 및 결과를 정리해보았습니다.

### 테스트 도구
  - socket.io client 구동이 필요하여 nodeJs express로 간단한 tester 를 만들어 사용하였습니다.
  - 클라이언트 동일하게 socket.io 환경 설정 후 쿠버네티스 환경에 여러개를 띄워 테스트 하였습니다.
  - 동시 요청하는 tester pod 개수를 조절하여 rps를 조절하였습니다.

```js
function doSocket(ip) {
	const connectionOptions = {
		"force new connection": true,
		"timeout": 5000,
		"transports": ["websocket"],
		"reconnection": true,
		"reconnectionAttempts": 10,
		"reconnectionDelay": 10000,
		"reconnectionDelayMax": 20000,
		"randomizationFactor": 1 
	};

	let socket = ioClient.connect("ws://" + ip, connectionOptions);

	socket.on('connect', () => {
		socketMap.push(socket);
		setTimeout(() => {connectedCount++;}, 0);

		socket.emit('joinByGameId', {gameId: "sampleGameId"});
	});

	socket.on('reconnect_failed', () => {
		socketMap.delete(socket);
	});
}
```

<br>

### 단일 Pod 테스트
  - 조건: 신규 커넥션 4000 rps 부하
  - 결과
    - 웹소켓 커넥션 에러나 소실 없음 확인
    -  CPU 사용량이 부하 초반동안 2.0/4.0 임을 확인 

<br>

### 오토스케일링 테스트
  - 조건: 500 rps 로 총 16만 커넥션까지 연결
  - 결과
      - Pod 1개 -> 4개로 정상적으로 오토스케일링.
      - 전체 시간동안 1.5/4.0 이하의 CPU 사용
  - ![image](https://media.oss.navercorp.com/user/7829/files/b6ecef80-aa78-11ea-9ddd-93d64caee819)

<br>

### 최대 동시 커넥션 테스트
  - 조건: 200 rps 로 총 160만 커넥션까지 연결
  - 결과
     - Pod 4개 -> 20개 까지 정상적으로 오토스케일링.
     - 리커넥션 최대횟수 초과로 서서히 총 커넥션 개수 감소.
  - ![image](https://media.oss.navercorp.com/user/7829/files/a38e5400-aa79-11ea-8f5c-48ee0c9c580d)

<br>

### Pod 종료 테스트
  - 조건: Pod 4개, 총 16만 conn 으로, cluster 당 1만개 conn 상태에서 1개 Pod 강제종료.
  - 결과: 종료되는 Pod의 4만개 커넥션이 유실되지 않고 잘 리커넥션 됨을 확인


<br>

### 결과
- 신규커넥션에 대한 순간 가용 rps: 4000
- 지속적으로 오토스케일링이 가능한 rps: 500
- 최대 동시접속 connection: 약 150만

최종적으로 위 스펙을 만족하는 socket.io 서버를 쿠버네티스 환경으로 구축할 수 있었습니다.
