# Домашнее задание к занятию «Хранение в K8s. Часть 1»

## Цель задания

В тестовой среде Kubernetes необходимо:
1. Создать Deployment приложения с общим хранилищем для контейнеров
2. Создать DaemonSet для чтения логов ноды

## Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (MicroK8S)
2. Установленный локальный kubectl
3. Редактор YAML-файлов с подключённым git-репозиторием

## Задание 1. Создание Deployment приложения

### Манифест Deployment

Файл [deployment.yaml](task1/deployment.yaml):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-multitool
  template:
    metadata:
      labels:
        app: busybox-multitool
    spec:
      containers:
      - name: busybox
        image: busybox
        command: 
        - /bin/sh
        - -c
        - "while true; do echo $(date) >> /shared-data/date.log; sleep 5; done"
        volumeMounts:
        - name: shared-data
          mountPath: /shared-data
      - name: multitool
        image: wbitt/network-multitool
        volumeMounts:
        - name: shared-data
          mountPath: /shared-data
      volumes:
      - name: shared-data
        emptyDir: {}
```

### Результаты

1. Создание и проверка подов:
```bash
kubectl get pods
```

![image](https://github.com/temagraf/6-StorageK8s-I/blob/main/1-1.png)


2. Проверка работы общего хранилища:
```bash
kubectl exec busybox-multitool-c8d568fd9-gd6m5 -c multitool -- tail -f /shared-data/date.log
```

![image](https://github.com/temagraf/6-StorageK8s-I/blob/main/1-2.png)


## Задание 2. Создание DaemonSet для работы с логами

### Манифест DaemonSet

Файл [daemonset.yaml](task2/daemonset.yaml):
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: multitool-ds
spec:
  selector:
    matchLabels:
      app: multitool-ds
  template:
    metadata:
      labels:
        app: multitool-ds
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        securityContext:
          privileged: true
        volumeMounts:
        - name: varlog
          mountPath: /host/var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
          type: Directory
```

### Результаты

1. Проверка работы DaemonSet:
```bash
kubectl get pods -l app=multitool-ds
```
```
NAME                 READY   STATUS    RESTARTS   AGE
multitool-ds-pjggl   1/1     Running   0          16m
```
![image](https://github.com/temagraf/6-StorageK8s-I/blob/main/2-1.png)


2. Проверка доступа к логам:
```bash
kubectl exec $(kubectl get pod -l app=multitool-ds -o jsonpath='{.items[0].metadata.name}') -- tail /host/var/log/boot.log
```
```
[  OK  ] Started systemd-timedated.service
[  OK  ] Finished snapd.seeded.service
[  OK  ] Started snap.microk8s.daemon-containerd.service
[  OK  ] Started snap.microk8s.daemon-kubelite.service
```
![image](https://github.com/temagraf/6-StorageK8s-I/blob/main/2-2.png)

Ну тут я увидел сообщение [FAILED] Failed to start vboxadd.service - что связано с VirtualBox Guest Additions и не должно влиять на работу Kubernetes и наших подов. 
Появилось в логах загрузки системы, потому что работаю в виртуальной машине VirtualBox.
