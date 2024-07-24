# Домашнее задание к занятию «Как работает сеть в K8s» - Балдин

### Цель задания

Настроить сетевую политику доступа к подам.

### Чеклист готовности к домашнему заданию

1. Кластер K8s с установленным сетевым плагином Calico.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Calico](https://www.tigera.io/project-calico/).
2. [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
3. [About Network Policy](https://docs.projectcalico.org/about/about-network-policy).

-----

### Задание 1. Создать сетевую политику или несколько политик для обеспечения доступа

1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы.
2. В качестве образа использовать network-multitool.
3. Разместить поды в namespace App.

<blockquote>
  
#### Манифесты:

#### FRONTEND

<details>
<summary>Deployment</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: app
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
          name: frontend-http
        - containerPort: 443
          name: frontend-https
```
  
</details>

<details>
<summary> Service </summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-srv
  namespace: app
spec:
  selector:
    app: frontend
  ports:
  - name: front-srv-http
    protocol: TCP
    port: 80
    targetPort: frontend-http
  - name: front-srv-https
    protocol: TCP
    port: 443
    targetPort: frontend-https
```

</details>

#### BACKEND

<details>
<summary>Deployment</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: app
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
        - containerPort: 80
          name: backend-http
        - containerPort: 443
          name: backend-https
```

</details>

<details>
<summary> Service </summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-srv
  namespace: app
spec:
  selector:
    app: backend
  ports:
  - name: back-srv-http
    protocol: TCP
    port: 80
    targetPort: backend-http
  - name: back-srv-https
    protocol: TCP
    port: 443
    targetPort: backend-https
```

</details>


#### CACHE

<details>
<summary>Deployment</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
  namespace: app
  labels:
    app: cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
          name: cache-http
        - containerPort: 443
          name: cache-https
```

</details>

<details>
<summary> Service </summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cache-srv
  namespace: app
spec:
  selector:
    app: cache
  ports:
  - name: cache-srv-http
    protocol: TCP
    port: 80
    targetPort: cache-http
  - name: cache-srv-https
    protocol: TCP
    port: 443
    targetPort: cache-https
```

</details>
</blockquote>

4. Создать политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.

<blockquote>
<details>
<summary>DENY INGRESS</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-ingress
  namespace: app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

</details>

<details>
<summary> BACKEND </summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: frontend
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
```

</details>

<details>
<summary> CACHE </summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cache
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: cache
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: backend
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
```
</details>
</blockquote>
 
5. Продемонстрировать, что трафик разрешён и запрещён.

```bash
root@master01:~# kubectl exec -n app frontend-4fe2c685b3-s1dqh -- curl --max-time 5 -s backend-srv
WBITT Network MultiTool (with NGINX) - backend-ff15e432f4-e3pc1 - 10.234.83.42 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
root@master01:~# kubectl exec -n app backend-ff15e432f4-e3pc1 -- curl --max-time 5 -s cache-srv
WBITT Network MultiTool (with NGINX) - cache-24g55ec751-2g58t - 10.214.72.14 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
root@master01:~# kubectl exec -n app cache-24g55ec751-2g58t -- curl --max-time 5 -s backend-srv
command terminated with exit code 28
root@master01:~# kubectl exec -n app cache-24g55ec751-2g58t -- curl --max-time 5 -s frontend-srv
command terminated with exit code 28
root@master01:~# kubectl exec -n app backend-ff15e432f4-e3pc1 -- curl --max-time 5 -s frontend-srv
command terminated with exit code 28
```

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
