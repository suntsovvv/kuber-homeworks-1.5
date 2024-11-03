# Домашнее задание к занятию «Сетевое взаимодействие в K8S. Часть 2»

### Цель задания

В тестовой среде Kubernetes необходимо обеспечить доступ к двум приложениям снаружи кластера по разным путям.

------


### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция](https://microk8s.io/docs/getting-started) по установке MicroK8S.
2. [Описание](https://kubernetes.io/docs/concepts/services-networking/service/) Service.
3. [Описание](https://kubernetes.io/docs/concepts/services-networking/ingress/) Ingress.
4. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment приложений backend и frontend

Cоздал манифесты: 
frontend.yaml
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

```
frontend-svc.yaml
```yml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      name: ngiinx
      port: 80
      targetPort: 80

```
backend.yaml
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "8080"
```
backend-svc.yaml
```yml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
    - protocol: TCP
      name: multitool
      port: 8080
      targetPort: 8080

```
Применил манифесты и проверил:
```bash
user@microk8s:~/kuber-homeworks-1.5$ kubectl apply -f frontend.yaml 
deployment.apps/frontend created
user@microk8s:~/kuber-homeworks-1.5$ kubectl apply -f frontend-svc.yaml 
service/frontend-svc created
user@microk8s:~/kuber-homeworks-1.5$ kubectl apply -f backend.yaml 
deployment.apps/backend created
user@microk8s:~/kuber-homeworks-1.5$ kubectl apply -f backend-svc.yaml 
service/backend-svc created
user@microk8s:~/kuber-homeworks-1.5$ kubectl get deployments.apps 
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
backend    1/1     1            1           25m
frontend   3/3     3            3           25m
user@microk8s:~/kuber-homeworks-1.5$ kubectl get service
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
backend-svc    ClusterIP   10.152.183.151   <none>        8080/TCP   25m
frontend-svc   ClusterIP   10.152.183.156   <none>        80/TCP     26m
kubernetes     ClusterIP   10.152.183.1     <none>        443/TCP    9d
```
Проверил что есть доступ к приложениям внутри кластера, при помощи пода multitool:
```bash
user@microk8s:~/kuber-homeworks-1.5$ kubectl exec multitool -ti -- bash
multitool:/# curl frontend-svc:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
multitool:/# curl backend-svc:8080
WBITT Network MultiTool (with NGINX) - backend-847c446948-pn6ps - 10.1.128.250 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```
------

### Задание 2. Создать Ingress и обеспечить доступ к приложениям снаружи кластера


Включил Ingress-controller в MicroK8S:
```bash
user@microk8s:~/kuber-homeworks-1.5$ microk8s enable ingress
Infer repository core for addon ingress
Enabling Ingress
ingressclass.networking.k8s.io/public created
ingressclass.networking.k8s.io/nginx created
namespace/ingress created
serviceaccount/nginx-ingress-microk8s-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-microk8s-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-microk8s-role created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
configmap/nginx-load-balancer-microk8s-conf created
configmap/nginx-ingress-tcp-microk8s-conf created
configmap/nginx-ingress-udp-microk8s-conf created
daemonset.apps/nginx-ingress-microk8s-controller created
Ingress is enabled
user@microk8s:~/kuber-homeworks-1.5$ kubectl get ingressclass
NAME     CONTROLLER             PARAMETERS   AGE
nginx    k8s.io/ingress-nginx   <none>       9s
public   k8s.io/ingress-nginx   <none>       9s
```
Создал манифест ingress.yaml:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: suntsovvv.ru
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 8080
```
Применил:
```bash
user@microk8s:~/kuber-homeworks-1.5$ kubectl apply -f ingress.yaml 
ingress.networking.k8s.io/my-ingress created
user@microk8s:~/kuber-homeworks-1.5$ kubectl get ingress
NAME         CLASS   HOSTS          ADDRESS   PORTS   AGE
my-ingress   nginx   suntsovvv.ru             80      10s
```
На локальной машине добавил в файл "hosts" запись _51.250.111.155     suntsovvv.ru _:

```cmd
C:\Users\suntsov.vadim>ping suntsovvv.ru

Обмен пакетами с suntsovvv.ru [51.250.111.155] с 32 байтами данных:
Ответ от 51.250.111.155: число байт=32 время=105мс TTL=51
Ответ от 51.250.111.155: число байт=32 время=106мс TTL=51
Ответ от 51.250.111.155: число байт=32 время=108мс TTL=51
Ответ от 51.250.111.155: число байт=32 время=105мс TTL=51

Статистика Ping для 51.250.111.155:
    Пакетов: отправлено = 4, получено = 4, потеряно = 0
    (0% потерь)
Приблизительное время приема-передачи в мс:
    Минимальное = 105мсек, Максимальное = 108 мсек, Среднее = 106 мсек
```
Проверил доступ с помощью браузера или  с локального компьютера:

------

## Ссылка на манифесты:


