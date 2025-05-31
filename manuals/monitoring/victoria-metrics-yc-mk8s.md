# Yandex Cloud Managed Service for Kubernetes cluster monitoring with VictoriaMetrics

## Описание
В этом мануале развернём кластер MK8S в Yandex Cloud, установим в него Cluster версию VictoriaMetrics и проверим, можно ли скрейпить метрики. Дополнительно прикрутим к куберу Grafana и дашборды для удобного мониторинга.

Руководствоваться будем [официальной документацией VictoriaMetrics](https://docs.victoriametrics.com/guides/k8s-monitoring-via-vm-cluster/).

Нам понадобятся:
- Кластер Kubernetes с одной воркер-нодой;
- Интерфейс командной троки [YC](https://yandex.cloud/ru/docs/cli/quickstart);
- Интерфейс командной строки [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/);
- Диспетчер пакетов [helm](https://github.com/helm/helm);
- ~~Невероятное желание попасть на Т2~~.

Эксперимент проводился на кластере с [базовым типом мастера](https://yandex.cloud/ru/docs/managed-kubernetes/concepts/#master) и одной воркер-нодой, версия k8s — 1.30.

## Начало работы, подготовка окружения
Создадим кластер Kubernetes, пример команды (подробнее о процессе создания можно прочитать [здесь](https://yandex.cloud/ru/docs/managed-kubernetes/operations/kubernetes-cluster/kubernetes-cluster-create)):
```
yc managed-kubernetes cluster create \
  --name test-k8s \
  --network-name default \
  --public-ip \
  --release-channel regular \
  --version 1.30 \
  --cluster-ipv4-range 10.112.0.0/16 \
  --service-ipv4-range 10.96.0.0/16 \
  --security-group-ids enp1m09ssp727hl**** \
  --service-account-name mr-test \
  --node-service-account-name mr-test \
  --master-location zone=ru-central1-d,subnet-id=fl869ac3vhi****
```
Теперь создадим нод-группу, пример команды (подробности о процессе [тут](https://yandex.cloud/ru/docs/managed-kubernetes/operations/node-group/node-group-create)):
```
yc managed-kubernetes node-group create \
  --allowed-unsafe-sysctls <имена_небезопасных_параметров_ядра> \
  --cluster-name <имя_кластера> \
  --cores <количество_vCPU> \
  --core-fraction <гарантированная_доля_vCPU> \
  --daily-maintenance-window <настройки_окна_обновлений> \
  --disk-size <размер_хранилища_ГБ> \
  --disk-type <тип_хранилища> \
  --fixed-size <фиксированное_количество_узлов_в_группе> \
  --location zone=[<зона_доступности>],subnet-id=[<идентификатор_подсети>] \
  --memory <количество_ГБ_RAM> \
  --name <имя_группы_узлов> \
  --network-acceleration-type <тип_ускорения_сети> \
  --network-interface security-group-ids=[<идентификаторы_групп_безопасности>],ipv4-address=<способ_назначения_IP-адреса> \
  --platform-id <идентификатор_платформы> \
  --container-runtime containerd \
  --preemptible \
  --public-ip \
  --template-labels <ключ_облачной_метки=значение_облачной_метки> \
  --node-labels <ключ_k8s-метки=значение_k8s-метки>
  --version <версия_Kubernetes_на_узлах_группы> \
  --node-name <шаблон_имени_узлов> \
  --node-taints <taint-политики> \
  --container-network-settings pod-mtu=<значение_MTU_для_подов_группы>
```
Дождёмся, пока все ресурсы перейдут в статус ```Ready```, затем подключимся к кластеру:
```
yc managed-kubernetes cluster \
   get-credentials <имя_или_идентификатор_кластера> \
   --external
```
Проверим с помощью ```kubectl get pods``` и можем двигаться дальше.

## Установка VM Cluster и vmagent
Сначала добавим репозиторий VictoriaMetrics и установим нужный helm-чарт в кластер.

Неймспейс использовался ```test```, учитывайте это при указании ```remoteWrite```, иначе получите кучу совсем не интересных ошибок.

Репозиторий:
```
helm repo add vm https://victoriametrics.github.io/helm-charts/
```
Установка:
```
cat <<EOF | helm install vmcluster vm/victoria-metrics-cluster -f -
vmselect:
  podAnnotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "8481"

vminsert:
  podAnnotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "8480"

vmstorage:
  podAnnotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "8482"
EOF
```
Ожидаемый результат:
```
~ % kubectl get pods
NAME                                                           READY   STATUS    RESTARTS   AGE
vmcluster-victoria-metrics-cluster-vminsert-8647b754f-8jttx    1/1     Running   0          10m
vmcluster-victoria-metrics-cluster-vminsert-8647b754f-pmzh7    1/1     Running   0          10m
vmcluster-victoria-metrics-cluster-vmselect-7d64b7bb8f-rgjxv   1/1     Running   0          10m
vmcluster-victoria-metrics-cluster-vmselect-7d64b7bb8f-zpjfp   1/1     Running   0          10m
vmcluster-victoria-metrics-cluster-vmstorage-0                 1/1     Running   0          10m
vmcluster-victoria-metrics-cluster-vmstorage-1                 1/1     Running   0          10m
```
Затем установим сущность, которая и будет скрейпить метрики нашего кластера — ```vmagent```.

Команда:
```
helm install vmagent vm/victoria-metrics-agent -f yc-scrape-config.yaml
```
Аутентифицироваться будем с помощью [OAuth-токена](https://yandex.cloud/ru/docs/iam/concepts/authorization/oauth-token).

Содержимое файла ```yc-scrape-config.yaml``` (пример можно найти в [официальной документации VictoriaMetrics](https://docs.victoriametrics.com/victoriametrics/sd_configs/#yandexcloud_sd_configs)):
```
remoteWrite:
  - url: http://vmcluster-victoria-metrics-cluster-vminsert.test.svc.cluster.local:8480/insert/0/prometheus/

config:
  global:
    scrape_interval: 10s

scrape_configs:
- job_name: YC_with_oauth
  yandexcloud_sd_configs:
  - service: compute
    yandex_passport_oauth_token: "OAuth_token_here"
  relabel_configs:
  - source_labels: [__meta_yandexcloud_instance_public_ip_0]
    target_label: __address__
    replacement: "$1:9100"
```
Проверим, как себя чувствует под ```vmagent```:
```
~ % kubectl get pods
NAME                                                           READY   STATUS    RESTARTS   AGE
vmagent-victoria-metrics-agent-6b8c7d86f4-vlqpr                1/1     Running   0          19s
```
Если на этом этапе все поды VM находятся в статусе ```Running```, успех близок.

Теперь к самой приятной части.

## Установка Grafana, вывод метрик на дашборды

Всё как раньше, добавляем репозиторий Grafana:
```
helm repo add grafana https://grafana.github.io/helm-charts
```
Теперь устанавливаем графану в кластер и добавляем красивые дашборды:
```
cat <<EOF | helm install my-grafana grafana/grafana -f -
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: victoriametrics
          type: prometheus
          orgId: 1
          url: http://vmcluster-victoria-metrics-cluster-vmselect.test.svc.cluster.local:8481/select/0/prometheus/
          access: proxy
          isDefault: true
          updateIntervalSeconds: 10
          editable: true

  dashboardProviders:
   dashboardproviders.yaml:
     apiVersion: 1
     providers:
     - name: 'default'
       orgId: 1
       folder: ''
       type: file
       disableDeletion: true
       editable: true
       options:
         path: /var/lib/grafana/dashboards/default

  dashboards:
    default:
      victoriametrics:
        gnetId: 11176
        revision: 18
        datasource: victoriametrics
      vmagent:
        gnetId: 12683
        revision: 7
        datasource: victoriametrics
      kubernetes:
        gnetId: 14205
        revision: 1
        datasource: victoriametrics
EOF
```
Ожидаемый вывод примерно вот такой:
```
NAME: my-grafana
LAST DEPLOYED: Sat May 31 00:49:28 2025
NAMESPACE: test
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace test my-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo


2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   my-grafana.test.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace test -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=my-grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace test port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####
#################################################################################
```
Проверяем состояние пода ```my-grafana-*```:
```
my-grafana-6d6865d599-dxv7w                                    1/1     Running     0              2m27s
```
Если всё в порядке, двигаемся дальше.

Теперь остался последний шаг. Получаем пароль для Grafana, пробрасываем порт:
```
~ % kubectl get secret --namespace test my-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

export POD_NAME=$(kubectl get pods --namespace test -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=my-grafana" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace test port-forward $POD_NAME 3000
```
Ожидаемый результат — вывод пароля и начало проброса порта:
```
izofun43h2q83YhAm**********************
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
Handling connection for 3000
Handling connection for 3000
```
Теперь идём на ```127.0.0.1:3000```, логинимся с помощью имени пользователя ```admin``` и пароля, полученного из вывода предыдущей команды.

## Проверка

Наслаждаемся результатом и идём смотреть, какие метрики скрейпит наш VictoriaMetrics.

Установлены три дашборда:

```Kubernetes Cluster Monitoring (via Prometheus)```:

<img width="1151" alt="image" src="https://github.com/user-attachments/assets/5b47918b-d570-4928-a6e4-604351d5a314" />

```VictoriaMetrics - cluster```:

<img width="1150" alt="image" src="https://github.com/user-attachments/assets/a9808e54-139a-4bb1-9031-aa771daa2d4c" />

```vmagent```:

<img width="1151" alt="image" src="https://github.com/user-attachments/assets/c91d11bd-7851-4fb4-8576-cedbdec979f9" />

Остальное можно проверить во вкладке ```Explore```, пример:

<img width="1153" alt="image" src="https://github.com/user-attachments/assets/68f34733-896c-4d1f-8e6c-60e9961f6794" />

## За сим кланяюсь, приходите позже, ещё че-нибудь напишу.


