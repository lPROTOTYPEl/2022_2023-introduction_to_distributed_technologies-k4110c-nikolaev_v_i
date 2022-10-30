# Лабораторная работа №2 "Развертывание веб сервиса в Minikube, доступ к веб интерфейсу сервиса. Мониторинг сервиса.

## Общая информация

University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)

Year: 2022/2023

Group: K4110C

Author: Nikolaev Vladislav Igorevich

Lab: Lab2

Date of create: 29.10.2022

Date of finished:

## Ход работы

Создадим отдельный namespace для развертывания приложения:

```bash
$ kubectl create namespace lab2
```

Проверим, что namespace действительно создался:

```bash
$ kubectl get ns

# Output
NAME              STATUS   AGE
default           Active   17m
kube-node-lease   Active   17m
kube-public       Active   17m
kube-system       Active   17m
lab2              Active   2s
```

## Создание Deployment

Для развертывания приложения был создан манифест [deployment.yaml](deployment.yaml). Deployment - это абстракция Kubernetes, которая позволяет управлять приложением, его версиями и обновлениями. В объекте Deployment хранится информация о конфигурации подов, количестве необходимых реплик и методе обновления подов в случае изменения их конфигурации.

При развертывании deployment создается еще одна абстракция - ReplicaSet. Она отвечает за описание и контроль за несколькими экземплярами приложений.

Для развертывания deployment используем следующую команду:

```bash
$ kubectl apply -f deployment.yaml -n lab2
```

Проверим какие ресурсы были созданы в итоге выполнения команды:

```bash
$ kubectl get all -n lab2

# Output:
NAME                            READY   STATUS    RESTARTS   AGE
pod/lab2-app-6df9f56b49-jmr9l   1/1     Running   0          84m
pod/lab2-app-6df9f56b49-qnq8k   1/1     Running   0          84m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/lab2-app   2/2     2            2           84m

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/lab2-app-6df9f56b49   2         2         2       84m
```

Как видно из вывода, deployment создал новый объект ReplicaSet, который, в свою очередь, создал необходимое количество реплик приложений.

## Создание Service типа LoadBalancer

Для предоставления доступа к подам деплоймента `lab2-app` был создан манифест [service.yaml](service.yaml). Данный манифест определяет Service типа LoadBalancer и выбирает поды, которые имеют selectorLabel `app=lab2-app`.

Сервисы в Kubernetes позволяют определить набор подов и правила доступа к ним. С помощью сервисов разные части приложения могут "общаться" друг с другом (например, фронтенд с бэкендом).

Для создания описанного в манифесте сервиса была использована следующая команда:

```bash
$ kubectl apply -f service.yaml
```

Для того, чтобы определить правильно ли настроены селекторы для подов, выведем описание созданного сервиса:

```bash
$ kubectl describe svc lab2-app -n lab2

# Output
Name:                     lab2-app
Namespace:                lab2
Labels:                   app=lab2-app
Annotations:              <none>
Selector:                 app=lab2-app
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.98.69.74
IPs:                      10.98.69.74
Port:                     http  3000/TCP
TargetPort:               3000/TCP
NodePort:                 http  31121/TCP
Endpoints:                172.17.0.2:3000,172.17.0.3:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

В строке Enpoints приведены IP адреса подов, на которые сервис будет проксировать трафик. Сравним их с IP-адресам подов деплоймента lab2-app:

```bash
$ kubectl get po -n lab2 -o wide

# Output
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
lab2-app-6df9f56b49-jmr9l   1/1     Running   0          93m   172.17.0.3   minikube   <none>           <none>
lab2-app-6df9f56b49-qnq8k   1/1     Running   0          93m   172.17.0.2   minikube   <none>           <none>
```

Как видно из вывода - IP-адреса совпадают.


## Тестирование работы сервиса

Команда `minikube tunnel` дает возможность получить доступ к сервису типа LoadBalancer на хосте. Используем ее для тестирования нашего приложения.

После выполнения данной команды откроем в браузере страницу http://localhost:3000 и обновим несколько раз.

<img src="img/lab2_1.png" alt="drawing" width="500"/>
<img src="img/lab2_2.png" alt="drawing" width="500"/>

Как видно на рисунках, имя и IP контейнера изменяется. Это происходит потому, что сервис проксирует запросы то на один под, то на другой.

## Логи контейнеров

Для получения логов контейнера в Kubernetes используется следующая команда:

```bash
$ kubectl logs <pod_name> -n <namespace>
```

Логи пода lab2-app-6df9f56b49-jmr9l:

```bash
$ kubectl logs lab2-app-6df9f56b49-jmr9l -n lab2

# Output
Builing frontend
build finished
Server started on port 3000
```
