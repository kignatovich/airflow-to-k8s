# airflow-to-k8s

Добавляем хельм
```shell
helm repo add apache-airflow https://airflow.apache.org
```
Смотрим каие версии существуют

```shell
helm search repo apache-airflow/airflow --versions
```

Примерный вывод

```shell
NAME                  	CHART VERSION	APP VERSION	DESCRIPTION                                       
apache-airflow/airflow	1.15.0       	2.9.3      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.14.0       	2.9.2      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.13.1       	2.8.3      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.13.0       	2.8.2      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.12.0       	2.8.1      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.11.0       	2.7.1      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.10.0       	2.6.2      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.9.0        	2.5.3      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.8.0        	2.5.1      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.7.0        	2.4.1      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.6.0        	2.3.0      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.5.0        	2.2.4      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.4.0        	2.2.3      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.3.0        	2.2.1      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.2.0        	2.1.4      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.1.0        	2.1.2      	The official Helm chart to deploy Apache Airflo...
apache-airflow/airflow	1.0.0        	2.0.2      	Helm chart to deploy Apache Airflow, a platform...
```

Скачиваем хельм с нужной версией 

```shell
helm pull apache-airflow/airflow --version 1.15.0 --untar
```
В текущем каталоге появится каталог airflow

Для начала генерируем `webserverSecretKey`

```python
python3 -c 'import secrets; print(secrets.token_hex(16))'
```

Добовляем его в файле `values.yaml`:

```shell
webserverSecretKey: a9e90fdf97dc742b3d46bb228600bcfe
```

Так же меняем тип `executor` на нужный в файле `values.yaml`:

```shell
executor: "KubernetesExecutor"
```
Если необходимо, в этом же файле отключаем Postgres, чтобы чарт не развертывал свой собственный контейнер Postgres:

```shell
postgresql:
  enabled: false
```

Настройки подключения к внешней bd находятся так же в этом файле.

```shell
  metadataConnection:
    user: postgres
    pass: postgres
    protocol: postgresql
    host: ~
    port: 5432
    db: postgres
    sslmode: disable
```

Запускаем наш чарт:

```shell
helm install my-airflow ./airflow --namespace airflow --create-namespace
```

После старта, чтобы попасть в web интерфейс выполните (логин пароль по умолчанию admin:admin):

```shell
kubectl port-forward svc/my-airflow-webserver 8080:8080 --namespace airflow
```

Пароль меняется так же в файле `values.yaml`:
```shell
  defaultUser:
    enabled: true
    role: Admin
    username: admin
    email: admin@example.com
    firstName: admin
    lastName: user
    password: admin
```

Если вы меняете что-то в чартах, то для обновления выполните команду:

```shell
helm upgrade my-airflow ./airflow --namespace airflow
```

# Добавление Dag

Создаем репозиторий для Dag

```shell
mkdir my-airflow-project && cd my-airflow-project
mkdir dags  # закидываем даги сюда
cat <<EOM > Dockerfile
FROM apache/airflow
COPY ./dags .
EOM
```

Далее билдим image

```shell
docker build --pull --tag my-dags:0.0.1 .
```

И обновляем наш хельм

```shell
helm upgrade my-airflow ./airflow --namespace airflow \
    --set images.airflow.repository=my-dags \
    --set images.airflow.tag=0.0.1
```

Посмотреть поды:

```shell
kubectl get pods -n airflow 
```

Подключится к поду:

```shell
kubectl exec -n airflow -it my-airflow-webserver-d9589f9b8-mj5zj -- /bin/bash
```
После этой команды можно посмотреть тип `executor`:

```shell
airflow@my-airflow-webserver-d9589f9b8-mj5zj:/opt/airflow$ airflow config get-value core executor
/home/airflow/.local/lib/python3.12/site-packages/airflow/metrics/statsd_logger.py:184 RemovedInAirflow3Warning: The basic metric validator will be deprecated in the future in favor of pattern-matching.  You can try this now by setting config option metrics_use_pattern_match to True.
KubernetesExecutor
airflow@my-airflow-webserver-d9589f9b8-mj5zj:/opt/airflow$ 
```