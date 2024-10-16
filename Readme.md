## Здесь приведена инструкция для работы с технологическими журналами 1С при помощи Opensearch и fluent-bit.

Требования:

Платформа 1С:Предприятие:
    8.3.25 и выше 
	
Операционная система:
    Linux (в качестве примера, в статье используем Almalinux 9.1) 
	
Используемые инструменты:
    fluentbit
    OpenSearch
    OpenSearch Dashboards 

fluentbit и сервер 1С:Предприятие должны быть установлены на одном сервере там же где формируется ТЖ.
Инструменты OpenSearch и OpenSearch Dashboards будут установлены на другом сервере. 

1. настройка сервера 1с и ТЖ:
два варианта полная инфа или только выборочно:
  1.1 вариант с полным ТЖ:
  
  /opt/1cv8/x86_64/8.3.25.1336/conf/logcfg.xml
  
  ```
<?xml version='1.0' encoding='UTF-8'?>
<config xmlns="http://v8.1c.ru/v8/tech-log">
  <log location="/1c/logs" history="4" placement="plain" format="json">
    <event>
      <ne property="Name" value=""/>
    </event>
    <property name="all">
    </property>
  </log>
</config>
```

  1.2 ТЖ с фильтрацией только необходимого:

```
  <?xml version="1.0"?>
<config xmlns="http://v8.1c.ru/v8/tech-log">
        <dump create="false"/>
        <log location="/mnt/nfsshare/tj/1c-al01/logs"  history="4" placement="plain" format="json">
                <event>
                        <eq property="name" value="DBPOSTGRS"/>
                        <ge property="Durationus" value="100000"/>
                </event>
                <property name="Context"/>
                <property name="p:processName"/>
        </log>
           <log location="/mnt/nfsshare/tj/1c-al01/logs"  history="4" placement="plain" format="json">
                <event>
                        <eq property="name" value="TLOCK"/>
                </event>
                <event>
                        <eq property="name" value="TTIMEOUT"/>
                </event>
                <event>
                        <eq property="name" value="TDEADLOCK"/>
                </event>
                <property name="t:connectID"/>
                <property name="Regions"/>
                <property name="Locks"/>
                <property name="WaitConnections"/>
                <property name="DeadlockConnectionIntersections"/>
                <property name="alreadyLocked"/>
                <property name="escalating"/>
                <property name="Context"/>
                <property name="p:processName"/>
        </log>
            <log location="/mnt/nfsshare/tj/1c-al01/logs" history="4" placement="plain" format="json">
                <event>
                        <eq property="name" value="CALL"/>
                        <ge property="Duration" value="10000"/>
                </event>
                <property name="Usr"/>
                <property name="Context"/>
                <property name="SessionID"/>
                <property name="Memory"/>
                <property name="MemoryPeak"/>
                <property name="InBytes"/>
                <property name="OutBytes"/>
                <property name="p:processName"/>
        </log>
</config>
```
Внимание на поля placement и format. 
В данном случае placement=plain означает, что файлы будут расположены в указанной директории 
без создания промежуточных каталогов с именами процессов,
 а format=json определяет содержимое логов в формате JSON  

2. Применяем владельца
```
   sudo chown usr1cv8:grp1cv8 /opt/1cv8/x86_64/8.3.25.1336/conf/logcfg.xml
```
3. Теперь главное:
 установка Opensearch )
используется мануал https://opensearch.org/docs/latest/install-and-configure/install-opensearch/index/
смотрим по совместимости версии Java
для установки последней версии надо такие: 

OpenSearch Version 	Compatible Java Versions 	Bundled Java Version
2.12.0+ 	               11, 17, 21 	         21.0.4+7

Если установлен в другое место не по умолчанию, не забываем переменные:
```
export OPENSEARCH_JAVA_HOME=/path/to/opensearch-2.17.0/jdk
```
4. не забываем про порты при использовании firewalld  в нашем случае
Port number 	OpenSearch component
443 	OpenSearch Dashboards in AWS OpenSearch Service with encryption in transit (TLS)
5601 	OpenSearch Dashboards
9200 	OpenSearch REST API
9300 	Node communication and transport (internal), cross cluster search
9600 	Performance Analyzer

5. важные моменты перед установкой: проверяем параметры

```
cat /proc/sys/vm/max_map_count
```
 ставим следующее значение /etc/sysctl.conf:

```
vm.max_map_count=262144
```
применяем 

```
sudo sysctl -p
```
6. Установка Java:

```
java --version
```

```
wget https://download.oracle.com/java/23/latest/jdk-23_linux-aarch64_bin.rpm
```
7. Есть несколько вариантов установки opensearch:
мы выберем стандартный вариант с версией 2.17
Делаем локальную репу
```
sudo curl -SL https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/opensearch-2.x.repo -o /etc/yum.repos.d/opensearch-2.x.repo
```
вычищаем кеш:
```
sudo yum clean all
```
проверяем:
```
 sudo yum repolist
```
смотрим доступные версии:
```
sudo yum list opensearch --showduplicates
```
можем поставить последнюю версию так:
```
sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD=<custom-admin-password> yum install opensearch
```
либо выбрать вот так:
```
sudo env OPENSEARCH_INITIAL_ADMIN_PASSWORD=<custom-admin-password> yum install 'opensearch-2.17.0'
```
выполняем запуск и проверку:
```
sudo systemctl start opensearch
```
```
sudo systemctl status opensearch
```
сразу оговоримся что ставится демо пакет и tls
их надо отключить

и еще момент: 
при установке не стоит ставить слабые и стандартные пароли - установка будет вываливаться в ошибку

Проверяем доступность сервиса
 ```
 curl -X GET http://localhost:9200 -u 'admin:pass --insecure
 ```
 где pass ваш пароль примененный при установке custom-admin-password
На запрос должен быть примерно такой ответ:

{
  "name" : "os-rs",
  "cluster_name" : "opensearch",
  "cluster_uuid" : "OPEqarFTRNS6ZMkbLDcTow",
  "version" : {
    "distribution" : "opensearch",
    "number" : "2.17.0",
    "build_type" : "rpm",
    "build_hash" : "8586481dc99b1740ca3c7c966aee15ad0fc7b412",
    "build_date" : "2024-09-13T01:03:56.097566655Z",
    "build_snapshot" : false,
    "lucene_version" : "9.11.1",
    "minimum_wire_compatibility_version" : "7.10.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}

проверяем плагины
```
curl -X GET http://localhost:9200/_cat/plugins?v -u admin:pass --insecure
```
name  component                            version
os-rs opensearch-alerting                  2.17.0.0
os-rs opensearch-anomaly-detection         2.17.0.0
os-rs opensearch-asynchronous-search       2.17.0.0
os-rs opensearch-cross-cluster-replication 2.17.0.0
os-rs opensearch-custom-codecs             2.17.0.0
os-rs opensearch-flow-framework            2.17.0.0
os-rs opensearch-geospatial                2.17.0.0
os-rs opensearch-index-management          2.17.0.0
os-rs opensearch-job-scheduler             2.17.0.0
os-rs opensearch-knn                       2.17.0.0
os-rs opensearch-ml                        2.17.0.0
os-rs opensearch-neural-search             2.17.0.0
os-rs opensearch-notifications             2.17.0.0
os-rs opensearch-notifications-core        2.17.0.0
os-rs opensearch-observability             2.17.0.0
os-rs opensearch-performance-analyzer      2.17.0.0
os-rs opensearch-reports-scheduler         2.17.0.0
os-rs opensearch-security                  2.17.0.0
os-rs opensearch-security-analytics        2.17.0.0
os-rs opensearch-skills                    2.17.0.0
os-rs opensearch-sql                       2.17.0.0
os-rs opensearch-system-templates          2.17.0.0
os-rs query-insights                       2.17.0.0

Приступаем к настройке параметров opensearch.yml:
```
sudo nano /etc/opensearch/opensearch.yml
```
выставляем такие параметры:
```
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["10.15.22.42"]
cluster.initial_cluster_manager_nodes: ["os-rs"]
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.http.enabled: false
plugins.security.system_indices.enabled: false
```
меняем настройки Java
 ```
 nano /etc/opensearch/jvm.options
```
```
-Xms4g
-Xmx4g
```
Если надо настроить супер пупер секьюрность то в раздел Configure TLS
```
 https://opensearch.org/docs/latest/install-and-configure/install-opensearch/rpm/#configure-tls
```
Рестартим:
```
sudo systemctl restart opensearch
```
8. Аналогично ставим opensearch-dashboards (kibana):

Ставим локальную репу:
```
sudo curl -SL https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.x/opensearch-dashboards-2.x.repo -o /etc/yum.repos.d/opensearch-dashboards-2.x.repo
```
```
sudo yum repolist
```
```
sudo yum clean all
```
```
sudo yum list opensearch-dashboards --showduplicates
```
```
sudo yum install 'opensearch-dashboards-2.17.0'
```
Ставим переменную:
```
export NODE_HOME=/usr/share/opensearch-dashboards/node 
```
Запускаем:  
```
sudo systemctl start opensearch-dashboards
```
Проверяем:
```
ss -tulpn
```
если не работает смотреть установлен ли node.js
```
npm -v
```
```
node -v
```
```
export NODE_HOME=/usr/share/opensearch-dashboards/node/
```
проверка:
```
curl -X GET http://localhost:5601 -u admin:pass --insecure
```
проверим переменные:
```
nano /etc/profile
```
```
#----------NODE config
export NODE_HOME=/usr/share/opensearch-dashboards/node
```
```
export PATH=$PATH:$NODE_HOME/bin
```
```
export NODE_PATH=$NODE_HOME/lib/node_modules
```
применяем:
```
source /etc/profile
```
9. если появился дашборд продолжаем настраивать сборщик fluent-bit для Алмалинукс 9.1

установку производим по https://docs.fluentbit.io/manual/installation/getting-started-with-fluent-bit

ставим скрипт:
```
curl https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh
```
конфигурируем репу fluent-bit.repo in /etc/yum.repos.d/
```
[fluent-bit]
name = Fluent Bit
baseurl = https://packages.fluentbit.io/centos/$releasever/
gpgcheck=1
gpgkey=https://packages.fluentbit.io/fluentbit.key
repo_gpgcheck=1
enabled=1
```
ставим сборщик
```
sudo yum install fluent-bit
```
Поместить архив fluentbit_config_1c.tar.gz (скачать https://its.1c.ru/db/files/1CITS/EXE/Scalability/i8106019/i8106019.zip ) в директорию /tmp. 

на астралинукс не ставится
я паковал сам из исходников сборку на астру
но вы можете скачать для debian 10 пакет и попробовать его поставить

создаем каталог скриптов
```
sudo mkdir -p /1c/config/fluent-bit
```
увеличиваем лимит по открытым файлам в системе
```
ulimit -n
```
Если покажет значение 1024
ставим:
```
ulimit -n 65535
```
заливаем туда из /tmp
/1c/config/fluent-log/techlog.conf

приводим конфигурацию techlog.conf к след виду
```
[SERVICE]
    Mem_Buf_Limit  500MB
    Log_Level info											# уровень детализации сообщений
    Parsers_File /1c/config/fluent-log/techlog_parser.conf  # путь до парсера
[INPUT]
    Name   tail
    Path   /mnt/tj/logs/*.log     # путь ТЖ
    Buffer_Chunk_Size 20MB
    Buffer_Max_Size 100MB
    Parser techlog
[FILTER]
    Name lua
    Match *
    Script /1c/config/fluent-bit/transform_techlog.lua
    Call transform_techlog
[OUTPUT]
    Name  opensearch
    Match *
    Host localhost
    Port 9200
    Http_User admin
    Http_Passwd pass
    Index techlog_index_1                 # 1 это номер логируемого сервера 
    Buffer_Size  15MB
    Type  _doc
    logstash_format true
    Logstash_Prefix techlog_index_1
    Suppress_Type_Name true
```
в конфиг парсера по дефолту добавляем 
```
/etc/fluent-bit/parser.conf
```
```
[PARSER]
    Name techlog
    Format json
```
конфигурация настроек по умолчанию
/etc/fluent-bit/fluent-bit.conf
```
    flush        5
   daemon       Off
   log_level    info
   parsers_file parsers.conf
   plugins_file plugins.conf
   http_server  Off
   http_listen  0.0.0.0
   http_port    2020
   storage.metrics on
[INPUT]
    name cpu
    tag  cpu.local

    # Read interval (sec) Default: 1
    interval_sec 1

[OUTPUT]
    name  stdout
    match *
```
В файле techlog.conf, который входит в состав дистрибутива распакованных ранее скриптов, содержатся основные параметры fluentbit для сбора данных. 
В большинстве случаев требуется изменить только несколько параметров:
Path в секции [INPUT]. Этодиректория настроек технологического журнала в параметре location с добавлением маски файлов *.log. Например: /mnt/1c/tj/*.log;
script в секции [FILTER]. Это полный путь к файлу transform_techlog.lua. Например: /1c/config/fluent-bit/transform_techlog.lua;
параметры Host и Port в секции [OUTPUT] это имя хоста и порта установленного  OpenSearch ;
параметры Http_User и Http_Passwd в секции [OUTPUT] это логин и пароль к доступу OpenSearch.
В OpenSearch автоматически создается пользователь admin с паролем pass. 
При желании можно создать нового пользователя и указать его значения в этих параметрах;
Для указания наименования индекса в OpenSearch, по которому будут сохраняться документы 
(структурированные сообщения) OpenSearch, задайте параметр Index в секции [OUTPUT]. 
Если не менять значение, то индекс по умолчанию будет techlog_index. 

Заходим в диеркторию,куда установлен или помещен fluent-bit
```
cd /opt/fluent-bit/bin
```
запускаем сборщик
```
sudo ./fluent-bit --config /1c/config/fluent-bit/techlog.conf
```
смотрим ошибки при запуске

при ошибках
проверить доступность
```
sudo curl -X GET http://<IP>:9200/_cat/plugins?v -u admin:pass --insecure
```
иногда просит установить библиотеку 
```
sudo apt install libyaml-dev
```
Снова запускаем
```
cd /opt/fluent-bit/bin
```
```
sudo ./fluent-bit --config /1c/config/techlog.conf --parser /etc/fluent-bit/parsers.conf
```
Далее проверяем и заводим паттерны:
согласно https://its.1c.ru/db/metod8dev/content/6019/hdoc



