# Istio性能测试

# 压测大纲
![](https://raw.githubusercontent.com/kbsonlong/notes_statics/main/images/images20220523221425.png)
<!--more-->
# 压测的必要性

# 压测部署架构图

![](https://raw.githubusercontent.com/kbsonlong/notes_statics/main/diagrams/Istio-Fortio-Benchmarking.drawio.png)

# 环境准备
## 部署 Istio

## 部署监控组件

## 部署压测服务

### Fortio v1版本
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-server-l3
  labels:
    app: fortio-server-l3
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio-server-l3
  template:
    metadata:
      labels:
        app: fortio-server-l3
        version: v1
    spec:
      containers:
        - name: fortio-server-l3
          # 在中国，你可以使用 docker.mirrors.ustc.edu.cn/fortio/fortio:latest
          image: fortio/fortio:latest
          ports:
            - containerPort: 8080
              name: http-port
            - containerPort: 8078
              name: udp-port
            - containerPort: 8079
              name: grpc-port
            - containerPort: 8081
              name: https-port
          command:
            - fortio
            - server
---
apiVersion: v1
kind: Service
metadata:
  name: fortio-server-l3
spec:
  ports:
    - name: http-port
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: https-port
      port: 8081
      protocol: TCP
      targetPort: 8081
    - name: http2-grpc
      port: 8079
      protocol: TCP
      targetPort: 8079
    - name: udp-grpc
      port: 8078
      protocol: UDP
      targetPort: 8078
  selector:
    app: fortio-server-l3
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-server-l2
  labels:
    app: fortio-server-l2
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio-server-l2
  template:
    metadata:
      labels:
        app: fortio-server-l2
        version: v1
    spec:
      containers:
        - name: fortio-server-l2
          # 在中国，你可以使用 docker.mirrors.ustc.edu.cn/fortio/fortio:latest
          image: fortio/fortio:latest
          ports:
            - containerPort: 8080
              name: http-port
            - containerPort: 8072
              name: http-m
            - containerPort: 8078
              name: udp-port
            - containerPort: 8079
              name: grpc-port
            - containerPort: 8081
              name: https-port
          command:
            - fortio
          args: ["server", "-P", "8072 fortio-server-l3:8080"]
---
apiVersion: v1
kind: Service
metadata:
  name: fortio-server-l2
spec:
  ports:
    - name: http-m
      port: 8072
      protocol: TCP
      targetPort: 8072
    - name: http-port
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: https-port
      port: 8081
      protocol: TCP
      targetPort: 8081
    - name: http2-grpc
      port: 8079
      protocol: TCP
      targetPort: 8079
    - name: udp-grpc
      port: 8078
      protocol: UDP
      targetPort: 8078
  selector:
    app: fortio-server-l2
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-server-l1
  labels:
    app: fortio-server-l1
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio-server-l1
  template:
    metadata:
      labels:
        app: fortio-server-l1
        version: v1
    spec:
      containers:
        - name: fortio-server-l1
          # 在中国，你可以使用 docker.mirrors.ustc.edu.cn/fortio/fortio:latest
          image: fortio/fortio:latest
          ports:
            - containerPort: 8080
              name: http-port
            - containerPort: 8071
              name: http-m
            - containerPort: 8078
              name: udp-port
            - containerPort: 8079
              name: grpc-port
            - containerPort: 8081
              name: https-port
          command:
            - fortio
          args: ["server", "-P", "8071 fortio-server-l2:8072"]
---
apiVersion: v1
kind: Service
metadata:
  name: fortio-server-l1
spec:
  ports:
    - name: http-m
      port: 8071
      protocol: TCP
      targetPort: 8071
    - name: http-port
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: https-port
      port: 8081
      protocol: TCP
      targetPort: 8081
    - name: http2-grpc
      port: 8079
      protocol: TCP
      targetPort: 8079
    - name: udp-grpc
      port: 8078
      protocol: UDP
      targetPort: 8078
  selector:
    app: fortio-server-l1
  type: NodePort
```
### Fortio v2版本
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-server-l3
  labels:
    app: fortio-server-l3
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio-server-l3
  template:
    metadata:
      labels:
        app: fortio-server-l3
        version: v1
    spec:
      containers:
        - name: fortio-server-l3
          # 在中国，你可以使用 docker.mirrors.ustc.edu.cn/fortio/fortio:latest
          image: fortio/fortio:latest
          ports:
            - containerPort: 8080
              name: http-port
            - containerPort: 8078
              name: udp-port
            - containerPort: 8079
              name: grpc-port
            - containerPort: 8081
              name: https-port
          command:
            - fortio
            - server
---
apiVersion: v1
kind: Service
metadata:
  name: fortio-server-l3
spec:
  ports:
    - name: http-port
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: https-port
      port: 8081
      protocol: TCP
      targetPort: 8081
    - name: http2-grpc
      port: 8079
      protocol: TCP
      targetPort: 8079
    - name: udp-grpc
      port: 8078
      protocol: UDP
      targetPort: 8078
  selector:
    app: fortio-server-l3
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-server-l2
  labels:
    app: fortio-server-l2
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio-server-l2
  template:
    metadata:
      labels:
        app: fortio-server-l2
        version: v1
    spec:
      containers:
        - name: fortio-server-l2
          # 在中国，你可以使用 docker.mirrors.ustc.edu.cn/fortio/fortio:latest
          image: fortio/fortio:latest
          ports:
            - containerPort: 8080
              name: http-port
            - containerPort: 8072
              name: http-m
            - containerPort: 8078
              name: udp-port
            - containerPort: 8079
              name: grpc-port
            - containerPort: 8081
              name: https-port
          command:
            - fortio
          args: ["server", "-P", "8072 fortio-server-l3:8080"]
---
apiVersion: v1
kind: Service
metadata:
  name: fortio-server-l2
spec:
  ports:
    - name: http-m
      port: 8072
      protocol: TCP
      targetPort: 8072
    - name: http-port
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: https-port
      port: 8081
      protocol: TCP
      targetPort: 8081
    - name: http2-grpc
      port: 8079
      protocol: TCP
      targetPort: 8079
    - name: udp-grpc
      port: 8078
      protocol: UDP
      targetPort: 8078
  selector:
    app: fortio-server-l2
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-server-l1
  labels:
    app: fortio-server-l1
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio-server-l1
  template:
    metadata:
      labels:
        app: fortio-server-l1
        version: v1
    spec:
      containers:
        - name: fortio-server-l1
          # 在中国，你可以使用 docker.mirrors.ustc.edu.cn/fortio/fortio:latest
          image: fortio/fortio:latest
          ports:
            - containerPort: 8080
              name: http-port
            - containerPort: 8071
              name: http-m
            - containerPort: 8078
              name: udp-port
            - containerPort: 8079
              name: grpc-port
            - containerPort: 8081
              name: https-port
          command:
            - fortio
          args: ["server", "-P", "8071 fortio-server-l2:8072"]
---
apiVersion: v1
kind: Service
metadata:
  name: fortio-server-l1
spec:
  ports:
    - name: http-m
      port: 8071
      protocol: TCP
      targetPort: 8071
    - name: http-port
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: https-port
      port: 8081
      protocol: TCP
      targetPort: 8081
    - name: http2-grpc
      port: 8079
      protocol: TCP
      targetPort: 8079
    - name: udp-grpc
      port: 8078
      protocol: UDP
      targetPort: 8078
  selector:
    app: fortio-server-l1
  type: NodePort
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: fortio-server-l1
spec:
  hosts:
    - fortio-server-l1
  http:
  - route:
    - destination:
        host: fortio-server-l1
        subset: v1
      weight: 10
    - destination:
        host: fortio-server-l1
        subset: v2
      weight: 90

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: fortio-server-l2
spec:
  hosts:
    - fortio-server-l2
  http:
  - route:
    - destination:
        host: fortio-server-l2
        subset: v1
      weight: 0
    - destination:
        host: fortio-server-l2
        subset: v2
      weight: 100

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: fortio-server-l1
spec:
  host: fortio-server-l1
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: fortio-server-l2
spec:
  host: fortio-server-l2
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## 部署压测工具

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-client
  labels:
    app: fortio-client
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio-client
  template:
    metadata:
      labels:
        app: fortio-client
        version: v2
    spec:
      containers:
        - name: fortio-client
          image: fortio/fortio:latest
          ports:
            - containerPort: 8080
              name: http-port
            - containerPort: 8078
              name: udp-port
            - containerPort: 8079
              name: grpc-port
            - containerPort: 8081
              name: https-port
          command:
            - fortio
            - server
---
apiVersion: v1
kind: Service
metadata:
  name: fortio-client
spec:
  ports:
    - name: http-port
      port: 8080
      protocol: TCP
      targetPort: 8080
    - name: https-port
      port: 8081
      protocol: TCP
      targetPort: 8081
    - name: http2-grpc
      port: 8079
      protocol: TCP
      targetPort: 8079
    - name: udp-grpc
      port: 8078
      protocol: UDP
      targetPort: 8078
  selector:
    app: fortio-client
  type: NodePort
```

# 性能压测

# 压测报告

![](https://raw.githubusercontent.com/kbsonlong/notes_statics/master/images/20220908154225.png)

画红线部分
- min: 最小响应时间2.97ms
- average: 平均响应时间11.286ms
- P50: 50%的请求响应时间在10.18ms内
- P75: 75%的请求响应时间在13.48ms内
- P90: 90%的请求响应时间在17.13ms内
- P99: 99%的请求响应时间在29.89ms内
- P99.9: 99.9%的请求响应时间在80.91ms内
- max: 最大响应时间324.597ms
