# Mock Exam - 1 - Solution

- `nginx:alpine` 이미지를 사용하여 이름이 `nginx-pod` 파드를 배포하시오.
`Weight: 6`
    - Name: nginx-pod
    - Image: nginx:alpine

    ```yaml
    # nginx-pod.yaml

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-pod
    spec:
      containers:
      - name: nginx-con
        image: nginx:alpine
    ```

    ```bash
    kubectl create -f nginx-pod.yaml
    ```

    ---

    ---

    ---

    ```bash
    kubectl run --generator=run-pod/v1 \
        nginx-pod --image=nginx:alpine \
        --dry-run

    kubectl run --generator=run-pod/v1 \
        nginx-pod --image=nginx:alpine \
        -o yaml --dry-run
    ```

    [Conventions d'utilisation de kubectl](https://kubernetes.io/fr/docs/reference/kubectl/conventions/)

- 레이블이 `tier=msg`, 이미지가 `redis:alpine` , 이름이 `messaging`인 파드를 배포하시오.
`Weight: 8`
    - Pod Name: messaging
    - Image: redis:alpine
    - Labels: tier=msg

    ```yaml
    # messaging-pod.yaml

    apiVersion: v1
    kind: Pod
    metadata:
      name: messaging
      labels:
        tier: msg # key: value, key=value 아님
    spec:
      containers:
      - name: messaging-con
        image: redis:alpine
    ```

    ```bash
    kubectl create -f messaging-pod.yaml
    ```

    ---

    ---

    ---

    ```bash
    kubectl run --generator=run-pod/v1 \
        messaging --image=redis:alpine \
        --labels=tier=msg \
        --dry-run

    kubectl run --generator=run-pod/v1 \
        messaging --image=redis:alpine \
        --labels=tier=msg \
        -o yaml \
        --dry-run
    ```

    [Conventions d'utilisation de kubectl](https://kubernetes.io/fr/docs/reference/kubectl/conventions/)

- 이름이 `apx-x9984574` 네임 스페이스 만들기
`Weight: 4`
    - Namespace: apx-x9984574

    ```bash
    kubectl create ns apx-x9984574 --dry-run
    kubectl create ns apx-x9984574
    kubectl get ns
    ```

- JSON 형식의 노드 목록을 가져와서 `/out/outputs/nodes-z3444kd9.json`에 저장하십시오. 
`Weight: 7`

    ```bash
    kubectl get nodes -o json > /opt/outputs/nodes-z3444kd9.json
    ```

    [JSONPath Support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)

- 이름이 `messaging-service`이고, `messaging` 파드를 클러스터 포트 `6379`로 expose하는 서비스를 Command만 사용하여 만드시오.
`Weight: 12`
    - Service: messaging-service
    - Port: 6379
    - Type: ClusterIp
    - Use the right labels

    ```bash
    kubectl get pods

    kubectl expose pods messaging \
        --type=ClusterIP \
        --name=messaging-service \
        --port=6379 \ # port, targetPort 둘 다 6379로 설정됨
        --labels=tier=msg \
        --dry-run

    kubectl expose pods messaging \
        --type=ClusterIP \
        --name=messaging-service \
        --port=6379 \ # port, targetPort 둘 다 6379로 설정됨
        --labels=tier=msg
    ```

- replicas `2`이고 이름이 `hr-web-app`,이미지가 `kodekloud/webapp-color`인 deployment를 만드시오.
`Weight: 11`
    - Name: hr-web-app
    - Image: kodekloud/webapp-color
    - Replicas: 2

    ```yaml
    # hr-web-app-deployment.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hr-web-app
      labels:
        tier: msg
    spec:
      replicas: 2
      selector:
        matchLabels:
          tier: msg
      template:
        metadata:
          labels:
            tier: msg
        spec:
          containers:
          - name: hr-web-app-con
            image: kodekloud/webapp-color
    ```

    ```bash
    # 아래처럼 하면 안 됨. 
    kubectl run --generator=deployment/apps.v1 \
        hr-web-app \
        --image=kodekloud/webapp-color \
        --replicas=2 \
        --labels=tier=msg \
        -o yaml \
        --dry-run

    # 정답
    kubectl create deployment hr-web-app --image=kodekloud/webapp-color
    kubectl scale deployment/hr-web-app --replicas=2
    ```

    [Conventions d'utilisation de kubectl](https://kubernetes.io/fr/docs/reference/kubectl/conventions/)

- 이름이 `static-busybox` 인 정적 파드를 마스터 노드에 `busybox` 이미지를 사용하고 command `sleep 1000`을 사용하여 만드시오.
`Weight: 8`
    - Name: static-busybox
    - Image: busybox

    ```bash
    ps -aux | grep -i kubelet
    # --config=/var/lib/kubelet/config.yaml

    cat /var/lib/kubelet/config.yaml | grep -i static
    # staticPodPath: /etc/kubernetes/manifests

    cd /etc/kubernetes/manifests

    vim busybox-static-pod.yaml
    ```

    ```yaml
    # busybox-static-pod.yaml

    apiVersion: v1
    kind: Pod
    metadata:
      name: static-busybox
    spec:
      containers:
      - name: static-busybox-con
        image: busybox
        command: ["sleep", "1000"]
    #  nodeName: master  
    ```

    ```bash
    # kubectl create -f busybox-static-pod.yaml 안 해도 자동 생성
    kubectl get pods
    ```

- 네임스페이스가 `finance`이고 이름이 `temp-bus`, 이미지가 `redis:alpine`인 파드를 만드시오.
`Weight: 12`
    - Name: temp-bus
    - Image Name: redis:alpine

    ```yaml
    # temp-bus-pod.yaml

    apiVersion: v1
    kind: Pod
    metadata:
      name: temp-bus
      namespace: finance
    spec:
      containers:
      - name: temp-bus-con
        image: redis:alpine
    ```

    ```bash
    kubectl create -f temp-bus-pod.yaml
    ```

    ---

    ---

    ---

    ```bash
    kubectl run --generator=run-pod/v1 \
        temp-bus \
        --image=redis:alpine \
        --namespace=finance \
        --dry-run

    kubectl run --generator=run-pod/v1 \
        temp-bus \
        --image=redis:alpine \
        --namespace=finance \
        -o yaml \
        --dry-run
    ```

- 새로운 응용 프로그램 `orange`이 배포되었지만 문제가 있다. 문제를 식별하고 해결하시오.
`Weight: 8`

    ```bash
    kubectl get pods
    # Init:CrashLoopBackOff

    kubectl get pods orange -o yaml > orange-pod.yaml
    kubectl delete -f orange-pod.yaml
    vim orange-pod.yaml
    /init
    ```

    ```yaml
    # vim orange-pod.yaml

    initContainers:
      - command:
        - sh
        - -c
        - sleep 2; # sleeeep 2에서 변경
    ```

    ```bash
    kubectl create -f orange-pod.yaml
    ```

- 클러스터의 노드에서 `hr-web-app`이`8080`으로 expose하고 있다. `hr-web-app-service`는 `30082` 으로 expose하게 `hr-web-app-service`를 작성하시오.
`Weight: 10`
    - Name: hr-web-app-service
    - Type: NodePort
    - Endpoints: 2
    - Port: 8080
    - NodePort: 30082

    ```bash
    kubectl get deploy

    kubectl expose deploy hr-web-app \
        --port=8080 \
        --name=hr-web-app-service \
        --type=NodePort \
        --dry-run

    kubectl get ep

    kubectl edit svc hr-web-app-service

    nodePort: 30082로 수정
    ```

    ```yaml
    # hr-web-app-service.yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: hr-web-app-service
      labels:
        tier: msg
    spec:
      ports:
        - protocol: TCP
          port: 8080
          targetPort: 8080
          nodePort: 30082
      selector:
        tier: msg
      type: NodePort

    # labels, selector 필수
    # endpoint는 포트포워딩 되는 부분, 2개가 포트포워딩 된다.
    ```

    ```bash
    kubetctl create -f hr-web-app-service.yaml
    ```

- JSON PATH 쿼리를 사용하여 모든 노드의 `osImage` 검색하여 `/opt/outputs/nodes_os_x43kj56.txt`에 저장하시오.
`osImages`는 `status.nodeInfo.osImages` 에 있다.
`Weight: 6`

    ```bash
    kubectl get nodes -o \
        jsonpath="{.items[*].status.nodeInfo.osImage}" \
        > /opt/outputs/nodes_os_x43kj56.txt
    ```

    [JSONPath Support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)

- `Persistent Volume`을 아래 조건에 맞게 작성하시오.
`Weight: 8`
    - Volume Name: pv-analytics
    - Storage: 100Mi
    - Access modes: ReadWriteMany
    - Host Path: /pv/data-analytics

    ```yaml
    # pv-analytics.yaml

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-analytics
    spec:
      capacity:
        storage: 100Mi
      accessModes:
        - ReadWriteMany
      hostPath:
        path: /pv/data-analytics
    ```

    ```bash
    kubectl create -f pv-analytics.yaml
    ```

    [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)