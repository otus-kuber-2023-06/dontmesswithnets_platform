# ___Kubernetes Network___

### ___В этой работе разберу взаимодействие с подами через использование Services___

_Как обычно, начинаю с подготовки стенда, разворачиваю одну ВМ с ОС Ubuntu 20.0.4 со следующими характеристиками_

```bash
user@isangulovK8S:~$ df -H
Filesystem                         Size  Used Avail Use% Mounted on
udev                               4.2G     0  4.2G   0% /dev
tmpfs                              834M  1.3M  832M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   67G  6.9G   57G  11% /
tmpfs                              4.2G     0  4.2G   0% /dev/shm
tmpfs                              5.3M     0  5.3M   0% /run/lock
tmpfs                              4.2G     0  4.2G   0% /sys/fs/cgroup
/dev/sda2                          1.1G  427M  524M  45% /boot
/dev/loop1                          59M   59M     0 100% /snap/core18/2785
/dev/loop0                          59M   59M     0 100% /snap/core18/2697
/dev/loop2                          67M   67M     0 100% /snap/core20/1822
/dev/loop4                          56M   56M     0 100% /snap/snapd/19457
/dev/loop3                          72M   72M     0 100% /snap/lxd/22753
/dev/loop5                          97M   97M     0 100% /snap/lxd/24061
/dev/loop6                          53M   53M     0 100% /snap/snapd/18357
/dev/loop7                          67M   67M     0 100% /snap/core20/1974
tmpfs                              834M     0  834M   0% /run/user/1000
user@isangulovK8S:~$ uname -a
Linux isangulovK8S 5.4.0-153-generic #170-Ubuntu SMP Fri Jun 16 13:43:31 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```

_Выполняю команды_ ___sudo apt update && sudo apt upgrade -y___

_Для выполнения ДЗ требуется кластер Minikube, поэтому вначале ставлю его_

_Прежде всего устанавливаю CRI, в моем случае Docker_

```bash
sudo apt install -y curl wget apt-transport-https
curl -fsSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo systemctl status docker
```

_Затем Minikube_

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

_Теперь kubectl_

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

_И в завершение_

```bash
sudo usermod -aG docker $USER && newgrp docker
minikube start --driver=docker
```

_После того, как все успешно установилось и кластер собрался, можно приступать к выполнению ДЗ. Первым делом нам говорят добавить в манифест с одного из предыдущих ДЗ readinessProbe. В итоге манифест будет выглядеть следующим образом_

```bash
apiVersion: v1
kind: Pod
metadata:
  name: my-web
  labels:
    app: web
spec:
  volumes:
    - name: app
      emptyDir: {}
  containers:
    - name: my-container
      image: ildarvildanovich/kubernetes-intro:1
      readinessProbe:
        httpGet:
          path: /index.html
          port: 80
      volumeMounts:
        - name: app
          mountPath: /app
  initContainers:
    - name: init-container-1
      image: busybox:latest
      volumeMounts:
        - name: app
          mountPath: /app
      command: ['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh']
```

_Применим его командой_ ___kubectl apply -f web-pod.yaml___

_После этого видим, что под в состоянии Running, однако он не в READY_

```bash
user@isangulovK8S:~$ kubectl get pod/my-web
NAME     READY   STATUS    RESTARTS   AGE
my-web   0/1     Running   0          70s
```

_Оно и не удивительно, так как_

```bash
Warning  Unhealthy  10s (x13 over 94s)  kubelet            Readiness probe failed: Get "http://10.244.0.3:80/index.html": dial tcp 10.244.0.3:80: connect: connection refused
```

_Далее нам говорят добавить еще одну проверку -_ ___livenesProbe___

```bash
livenessProbe:
  tcpSocket:
    port: 8000
```

_Перезапускаем наш под и видим, что, разумеется, это не помогло_

_Далее нам нужно создать_ ___web-deploy.yaml___ _, немного подкорректировав имеющийся у нас_ ___web-pod.yaml___

_Выглядеть он будет так_

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      name: my-web
      labels:
        app: web
    spec:
      volumes:
        - name: app
          emptyDir: {}
      containers:
        - name: my-container
          image: ildarvildanovich/kubernetes-intro:1
          readinessProbe:
            httpGet:
              path: /index.html
              port: 80
          livenessProbe:
            tcpSocket:
              port: 8000
          volumeMounts:
            - name: app
              mountPath: /app
      initContainers:
        - name: init-container-1
          image: busybox:latest
          volumeMounts:
            - name: app
              mountPath: /app
          command: ['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh']
```

_Теперь удалим старый под командой_ ___kubectl delete pod/my-web --grace-period=0 --force___ _и применим наш новый Deployment_

```bash
user@isangulovK8S:~$ kubectl apply -f web-deploy.yaml
deployment.apps/web created
user@isangulovK8S:~$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-56cb5d5554-gc8jj   0/1     Running   0          26s
```

_Если посмотреть через describe, то можно увидеть, что под не переходит в состояние Ready из-за неуспешной проверки readinessProbe. Исправим это в нашем манифесте, выставив число реплик в 3 и поменяв порт на 8000_

_После этого все заработает_

```bash
user@isangulovK8S:~$ kubectl get deployment
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    3/3     3            3           21m
```

_Далее нам нужно добавить блок Strategy в наш deployment_

```bash
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 100%
```

_После этого сохраняем yaml файл и применяем через_ ___kubectl apply -f web-deploy.yaml___

### ___Создание Service___

_Начнем с создания сервиса с типом_ ___ClusterIP___ _с создания манифеста_ ___web-svc-cip.yaml___

```bash
apiVersion: v1
kind: Service
metadata:
  name: web-svc-cip
spec:
  selector:
    app: web
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

_Проверяем_

```bash
user@isangulovK8S:~$ kubectl apply -f web-svc-cip.yaml
service/web-svc-cip created
user@isangulovK8S:~$ kubectl get svc
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP   17h
web-svc-cip   ClusterIP   10.103.7.108   <none>        80/TCP    13s
```

_Теперь, если подключиться к миникубу через_ ___minikube ssh___ _и выполнить_ ___curl http://10.103.7.108/index.html___ _можно увидеть, что файл отдается_

_Но если попытаться отправить icmp echo, то ничего не получится, в этот момент на самой ноде можно наблюдать через_ ___tcpdump___ однонаправленный трафик в никуда_

```bash
user@isangulovK8S:~$ sudo tcpdump -i br-31d0e667b02d icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br-31d0e667b02d, link-type EN10MB (Ethernet), capture size 262144 bytes
08:22:29.627686 IP 192.168.49.2 > 10.103.7.108: ICMP echo request, id 1, seq 30, length 64
08:22:30.647686 IP 192.168.49.2 > 10.103.7.108: ICMP echo request, id 1, seq 31, length 64
08:22:31.671733 IP 192.168.49.2 > 10.103.7.108: ICMP echo request, id 1, seq 32, length 64
08:22:32.699726 IP 192.168.49.2 > 10.103.7.108: ICMP echo request, id 1, seq 33, length 64
```

_Все дело в том, что на IP адресе 10.103.7.108 настроено dst nat правило, которое ждет подключений на 80-ый порт и транслирует их на порт 8000. Посмотреть можно так (под миникубом, естественно)_

```bash
root@minikube:~# iptables -t nat -S KUBE-SERVICES
-N KUBE-SERVICES
-A KUBE-SERVICES -d 10.96.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SERVICES -d 10.96.0.10/32 -p udp -m comment --comment "kube-system/kube-dns:dns cluster IP" -m udp --dport 53 -j KUBE-SVC-TCOU7JCQXEZGVUNU
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp cluster IP" -m tcp --dport 53 -j KUBE-SVC-ERIFXISQEP7F7OF4
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m comment --comment "kube-system/kube-dns:metrics cluster IP" -m tcp --dport 9153 -j KUBE-SVC-JD5MR3NA4I4DYORP
-A KUBE-SERVICES -d 10.103.7.108/32 -p tcp -m comment --comment "default/web-svc-cip cluster IP" -m tcp --dport 80 -j KUBE-SVC-6CZTMAROCN3AQODZ
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```

### ___Включение IPVS___

_Для включения IPVS подредактируем ConfigMap_

_Выполняем команду_ ___kubectl --namespace kube-system edit configmap/kube-proxy___ _, выставляем ключ mode в ipvs, а ключ strictARP в true_

_Теперь нужно удалить существующий под командой_ ___kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'___ _и он сам создатся заново_

_Проверяем, что под перезапустился_

```bash
user@isangulovK8S:~$ kubectl --namespace kube-system get pod --selector='k8s-app=kube-proxy'
NAME               READY   STATUS    RESTARTS   AGE
kube-proxy-ztwgw   1/1     Running   0          52s
```

_После того, как я поменял нужные значения, kube-proxy выполнил все необходимые настройки, но не удалил уже имеющиеся и не актуальные. Чтобы это исправить, внутри ВМ с minikube создадим файл /tmp/iptables.cleanup со следующим содержимым_

```bash
*nat
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
COMMIT
*filter
COMMIT
*mangle
COMMIT
```

_И после этого выполним команду_ ___iptables-restore /tmp/iptables.cleanup___

_После того, как я выставил mode : ipvs, можно внутри ВМ с миникубом вывести все интересующие нас правила_

```bash
root@minikube:~# ipvsadm --list -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.49.2:8443            Masq    1      0          0
TCP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.0.2:9153              Masq    1      0          0
TCP  10.101.125.128:80 rr
  -> 10.244.0.3:8000              Masq    1      0          0
  -> 10.244.0.4:8000              Masq    1      0          0
  -> 10.244.0.5:8000              Masq    1      0          0
UDP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0
```

### ___Работа с LoadBalancer и Ingress___

_Для начала нужно установить MetalLB. Для этого воспользуемся следующей командой_

```bash
https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

_После этого можно проверить, что все заработало_

```bash
user@isangulovK8S:~$ kubectl --namespace metallb-system get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-595f88d88f-kj4gw   1/1     Running   0          80s
pod/speaker-9q86r                 1/1     Running   0          80s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/webhook-service   ClusterIP   10.98.109.196   <none>        443/TCP   80s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   1         1         1       1            1           kubernetes.io/os=linux   80s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           80s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-595f88d88f   1         1         1       80s
```

_Для установки балансировщика используем следующий манифест_

```bash
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.17.255.1-172.17.255.255
```

_Применяем его через kubectl apply -f и идем дальше_

_Делаем копию файла wev-svc-cip.yaml в web-svc-lb-yaml и редактируем его, меняя на тип LoadBalancer_

_Получается вот так_

```bash
apiVersion: v1
kind: Service
metadata:
  name: web-svc-lb
spec:
  selector:
    app: web
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

_Применяем его и идем дальше_

_Для того, чтобы основная ОС знала о маршруте до адреса LB мы можем прописать статический маршрут через nexthop=ip address minikube. Сделаю это командой ниже_

```bash
sudo ip route add 172.17.255.0/24 via 192.168.49.2
```

_Теперь можем выполнить curl до адреса LB и посмотреть результат_

```bash
user@isangulovK8S:~$ kubectl get svc
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
kubernetes    ClusterIP      10.96.0.1        <none>         443/TCP        142m
web-svc-cip   ClusterIP      10.101.125.128   <none>         80/TCP         102m
web-svc-lb    LoadBalancer   10.106.221.87    172.17.255.1   80:31204/TCP   32m

user@isangulovK8S:~$ curl -I http://172.17.255.1
HTTP/1.1 200 OK
Server: nginx/1.25.1
Date: Tue, 25 Jul 2023 12:44:07 GMT
Content-Type: text/html
Content-Length: 83683
Last-Modified: Tue, 25 Jul 2023 11:01:15 GMT
Connection: keep-alive
ETag: "64bfab7b-146e3"
Accept-Ranges: bytes
```

### ___Создание Ingress___

_Для начала поднимем сервис ingress следующей командой_

```bash
user@isangulovK8S:~$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
```

_Далее создаем_ ___nginx-lb.yaml___

```bash
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/component: controller
spec:
  externalTrafficPolicy: Local
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/component: controller
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https
```

_Применяем его и проверяем_

```bash
user@isangulovK8S:~$ kubectl apply -f nginx-lb.yaml
service/ingress-nginx created

user@isangulovK8S:~$ kubectl --namespace ingress-nginx get svc
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
ingress-nginx                        LoadBalancer   10.99.136.106    172.17.255.2   80:30435/TCP,443:31672/TCP   101s
ingress-nginx-controller             NodePort       10.109.110.223   <none>         80:31454/TCP,443:30163/TCP   25m
ingress-nginx-controller-admission   ClusterIP      10.100.151.9     <none>         443/TCP                      25m
user@isangulovK8S:~$ curl -I http://172.17.255.2 port 31672
HTTP/1.1 404 Not Found
Date: Tue, 25 Jul 2023 13:18:34 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive

curl: (6) Could not resolve host: port
curl: (28) Failed to connect to 31672 port 80: Connection timed out
```

_Так и должно быть, все по плану_

### ___Создание Headless___

_Копируем содержимое web-svc-cip.yaml в web-svc-headless.yaml, при этом меняя имя сервиса и добавляя еще один параметр. Получается вот так_

```bash
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  type: ClusterIP
  clusterIP: None
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

_Как видим, IP адрес ему действительно не назначается_

```bash
user@isangulovK8S:~$ kubectl get svc
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
kubernetes    ClusterIP      10.96.0.1        <none>         443/TCP        3h5m
web-svc       ClusterIP      None             <none>         80/TCP         14s
web-svc-cip   ClusterIP      10.101.125.128   <none>         80/TCP         144m
web-svc-lb    LoadBalancer   10.106.221.87    172.17.255.1   80:31204/TCP   74m
```

_Далее создаем_ ___web-ingress.yaml___

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
    - http:
        paths:
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 8000
```

_Применяем и проверяем, получаем ли ответ от LB_

```bash
user@isangulovK8S:~$ curl http://172.17.255.1/web/index.html
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.25.1</center>
</body>
</html>
```

_Убеждаемся, что все работает_