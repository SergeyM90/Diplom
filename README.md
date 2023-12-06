# Дипломная работа по профессии «Системный администратор». Сергей Миронов-SYS-20. 

# ЗАДАЧА  


Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в Yandex Cloud и отвечать минимальным стандартам безопасности.


# ОТВЕТ

![image](https://github.com/SergeyM90/Diplom/assets/84016375/13e62fa8-e798-44e0-ba17-767e87296234)

# Технические детали  

Развертывание инфраструктуры автоматизировано с помощью Terraform, Ansible и Gitlab.
Все этапы выполняются в одном пайплайне.
Self-hosted Gitlab работает на локальном домашнем сервере. Раннер запускается в докер-контейнере. Несколько подробнее задокументировал https://github.com/SergeyM90/Diplom/blob/main/Details.md)
Для запуска Терраформа используется оффициальный образ от гитлаба gitlab-org/terraform-images/stable:latest
Для запуска Ansible образ собирается в начале пайплайна на основе alpine:3.18 (https://github.com/SergeyM90/Diplom/blob/main/Dockerfile.txt) и загружается в Container registry гитлаба

Этапы работы:
- сборка образа для ансибла;
- terraform validate;
- terraform plan;
- terraform apply;
- провижн машин с помощью ансибл;
- плейбук с постгресом;
- плейбук со всеми остальными сервисами;
- terraform destroy (запускается вручную).
  ![image](https://github.com/SergeyM90/Diplom/assets/84016375/0ef81bf0-815a-4ab6-a20d-f7be84e12db9)

Все чувствительные данные передаются в джобы через переменные окружения CI/CD - токен яндекса, имя пользователя, приватный и публичный ключи SSH для доступа к создаваемым машинам. IP адреса создаваемых машин передаются в ансибл через terraform output -> dotenv артефакты

![image](https://github.com/SergeyM90/Diplom/assets/84016375/7aa54132-fd73-4841-b9b4-3f18fea520e6)
![image](https://github.com/SergeyM90/Diplom/assets/84016375/b6c6f464-6c5f-460d-8935-63a89f649f5e)

Для реализации проекта из имеющихся у Яндекса вариантов выбрал дистрибутив AlmaLinux 8, т.к. с убунтой и дебианом работал c начала учебы, и решил, что пора изучать дистрибутивы RHEL, для CentOS больше нет поддержки, а из опенсорс вариантов активно развивается AlmaLinux.
![image](https://github.com/SergeyM90/Diplom/assets/84016375/a97970b7-8753-47d5-a1ec-2356d133b26c)

# Структура проекта  

Пайплайн: [.gitlab-ci.yml](https://github.com/SergeyM90/Diplom/blob/main/gitlab-ci.yml)  
Инфраструктура в Терраформе:  
[./terraform/](https://github.com/SergeyM90/Diplom/tree/main/Terraform)
Провижн необходимых сервисов с помощью Ансибл:  
Postgres cluster:  (https://github.com/SergeyM90/Diplom/tree/main/ansible/postgres) (адаптирован плейбук https://github.com/vitabaks/postgresql_cluster)  
[./ansible/postgres/inventory](https://github.com/SergeyM90/Diplom/blob/main/ansible/postgres/inventory.txt)  
[./ansible/postgres/deploy_pgcluster.yml](https://github.com/SergeyM90/Diplom/blob/main/ansible/postgres/deploy_pgcluster.yml)  
ELK, Zabbix, nginx:  
[./ansible/project/](https://github.com/SergeyM90/Diplom/tree/main/ansible/project)  
[./ansible/project/inventory](https://github.com/SergeyM90/Diplom/blob/main/ansible/project/inventory)  
[./ansible/project/deploy_project.yml](https://github.com/SergeyM90/Diplom/blob/main/ansible/project/deploy_project.yml)  

# Безопасность и сети  
На этапе разворачивания инфраструктуры создаются четыре подсети в одной VPC - условно разделенные на 3 приватных и 1 общедоступную. Открытые порты определяются с помощью групп безопасности и назначаются на сетевые интерфейсы. Дефолтная группа безопасности для всей VPC запрещает любой трафик.
3 приватных сети расположены в трех разных доступных в яндексе зонах - a, b и с. Это сделано для того, чтобы машины с Patroni-Postgres и NGINX физически располагались в разных ДЦ.
1 общедоступная - в зоне а

![image](https://github.com/SergeyM90/Diplom/assets/84016375/f690d051-be59-49af-8aa2-94fb4d9e9100)

Создано три группы безопасности  

в группе private-network разрешен трафик для необходимых портов (постгрес, patroni, etcd, ELK, zabbix, ssh) для всех четырех подсетей, также разрешен исходящий трафик для скачивания обновлений и пакетов и обращения к серверам DNS и NTP  
https://github.com/SergeyM90/Diplom/blob/main/Terraform/sec_private.tf  

группа public-network - это сервисы, которые должны быть доступны извне - внешний балансировщик Яндекса и NGINX. При этом GUI Kibana и Zabbix сидят за обратным прокси nginx.  
https://github.com/SergeyM90/Diplom/blob/main/Terraform/sec_public.tf  

группа bastion-network - доступ SSH извне. Единственная ВМ с этой группой - bastion host.  

https://github.com/SergeyM90/Diplom/blob/main/Terraform/sec_bastion.tf  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/67896a4b-d82c-4467-ba16-fbaabd9a5bf2)

Т.к. реализована концепция Bastion host, дальнейшая установка сервисов на машинах выполняется Ansible подключением jump host  
ansible_ssh_common_args="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ProxyCommand=\"ssh -o StrictHostKeyChecking=no -q \"{{ lookup('ansible.builtin.env', 'TF_VAR_ssh_user') }}\"@\"{{ lookup('ansible.builtin.env', 'EXTERNAL_IP_VM10_BASTION') }}\" -o IdentityFile=\"{{ lookup('ansible.builtin.env', 'ANSIBLE_SSH_PRIVATE_KEY') }}\" -o Port=22 -W %h:22\""  

# Nginx  

https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/nginx  

В яндексе на время работы с проектом зарезервирован один статический IP адрес, для которого у стороннего провайдера добавлена CNAME запись, а в Certificate Manager Яндекса выпущен let's encrypt сертификат для домена - это сделано заранее вручную, и в дальнейшем они просто используются терраформом.  

![image](https://github.com/SergeyM90/Diplom/assets/84016375/6eb293d3-4027-4744-b308-c13bf8aa9cd1)

SSL сертификат терминируется на балансировщике ALB1 Яндекса, балансировщик привязывается к зарезервированому IP адресу, далее трафик передается на http роутер и на бэкэнд, к которому привязана таргет-группа nginx-хостов.  

Добавлен обратный прокси, который перенаправляет на GUI забикса и кибаны.  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/3312750d-6334-4eea-9f21-4a735a523037)
![image](https://github.com/SergeyM90/Diplom/assets/84016375/e26043e4-7b4a-4805-843c-581112ef84bd)
![image](https://github.com/SergeyM90/Diplom/assets/84016375/07d1e012-1282-470d-9d7d-e965fdde847d)

# Кластер Postgres  

Для разворачивания Postgres адаптирован плэйбук по ссылке: https://github.com/vitabaks/postgresql_cluster  
Плэйбук:  https://github.com/SergeyM90/Diplom/blob/main/ansible/postgres/deploy_pgcluster.yml

Кластер постгреса состоит из трех машин, разнесенных по трем разным локациям яндекса - ru-central1-a, ru-central1-b, ru-central1-c  
Устанавливаются и конфигурируются Postgres, Patroni, etcd, pgbouncer.  
Кластер состоит из мастера и двух реплик, управляемых Patroni, работает в режиме Hight Availability, внутренний балансировщик Яндекса определяет мастера, подключения клиентов идут через PGBouncer, режим пула - transaction.  
RestAPI Patroni на порту 8008 используется для определения мастера по запросу к адресу [хост]:8008/master.  
Балансировщик яндекса блокирует опрашиваемые и таргетные порты, поэтому дополнительно устанавливаются правила IPtables для взаимодействия с балансировщиком и отдачи ему информации на недефолтных портах, в итоге после выполнения этого плейбука внутренний балансировщик выглядит так:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/922a5f00-a2f2-49b0-9b42-ed43b9a8c3e9)
![image](https://github.com/SergeyM90/Diplom/assets/84016375/067bee26-f183-44f3-b937-6d0e1f021400)

Подключение с бастиона на адрес балансировщика:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/dee2368a-b905-4413-bd1e-1adef6d2d1d6)

Базу данных в проекте использует Zabbix, поэтому сразу в плейбуке создается пользователь и пустая бд по дефолтному шаблону постгреса.  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/1509c45a-2365-4d2f-a14c-942888b8b70c)

в pg_hba добавлены scram-sha-256 подключения из приватных сетей проекта  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/a2bd6cf7-cfdf-45bb-9fcd-b3997df06e97)

Порты, необходимые для работы этой части инфраструктуры:  
8008 (Patroni)
8007 (Patroni для балансировщика)
2379,2378 (etcd)
5432 postgres
5431 (postgres для балансировщика)
6432 (pgbouncer)
10050 (zabbix agent)
Кластер доступен:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/4637f55a-6bce-4037-8d16-c6b4c2ac22c8)  

Пробное отключение мастера:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/393c757a-1d24-453a-9668-5cbceaff41ea)

Состояние кластера, лидер мгновенно поменялся:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/d8d896ed-53b7-44f2-87ac-fb0031d8e29a)

Вновь подключаемся с бастиона на адрес балансировщика, создаём новую таблицу:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/74ec0f61-1b46-4676-8633-38041a6859a3)

Обратное включение, бывший мастер становится репликой и догоняет нового мастера:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/1c7da43e-bdbf-465e-8bc1-58b53c48530b)

# Zabbix мониторинг  

Роли:  

https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/zabbix_server  
https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/zabbix_front  
https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/zabbix_agent  

Подключен мониторинг хостов заббиксом  
Дополнительно на nginx хостах включен модуль stub_status для мониторинга соединений  
На заббикс сервере после включения авторегистрации агентов сразу начинают поступать данные с хостов:  

![image](https://github.com/SergeyM90/Diplom/assets/84016375/87744ebe-ef06-49f3-9873-5c955d1221c5)
![image](https://github.com/SergeyM90/Diplom/assets/84016375/8aca9222-c73a-4121-8149-00f84aa90e5b)
![image](https://github.com/SergeyM90/Diplom/assets/84016375/140cccd4-b8d0-4bbf-8b92-1b69b61df0d7)

Мониторинг CPU, Memory, Network, NGINX.  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/7d7af62e-b8c5-4b1a-98f3-fc5688db919b)

# ELK

https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/elasticsearch  
https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/kibana  
https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/metricbeat  
https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/logstash  
https://github.com/SergeyM90/Diplom/tree/main/ansible/project/roles/filebeat  

В настоящее время ресурсы elasticsearch недоступны из России с официального репозитория, зеркало яндекса отключено, также недоступен docker registry эластика, в котором хранятся образы логсташа, метрикбита и файлбита.  
В связи с этим необходимые пакеты скачаны через VPN, создан локальный yum репозиторий, в хранилище образов добавлен образ готового диска с ОС и nginx. Для репозитория решил использовать Bastion host (для оптимизации ресурсов). Он сразу создается из заранее подготовленного образа с включенным nginx'ом.  
С помощью плейбука производится  
Установка Elasticsearch  
Генерация enrollment-token для Kibana и её подключение к эластику  
Установка Metricbeat на все хосты (на всех хостах включается модуль System, на хосте с эластиком - модуль для Elasticsearch и Kibana).  
Установка Logstash на хост c Elasticsearch  
Установка Filebeat на nginx хосты  
Т.к. у меня тестовый проект, Logstash отправляет логи в кластер под пользователем elastic, но для использования в продакшене необходимо создать отдельного пользователя с необходимыми ролями, то же касается Metricbeat. Так же из доработок на которые сейчас решил не тратить время - включение API Logstash и его мониторинг.  
Filebeat поставляет логи в логсташ, где настроен один пайплайн, который разбирает логи на разные индексы по document_type, устанавливаемому файлбитом.   
Конвейер Logstash:  
input {  
  beats {  
      port => "5044"  
      host => "192.168.101.31"  
  }  
}  
  
filter {  
  if [fields][document_type] == "nginx-access" {  
    grok {  
      match => {  
        "message" => "%{IPORHOST:ip} - %{DATA:user_name} \[%{HTTPDATE:time}\] \"%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}\" %  
 {NUMBER:response_code} %{NUMBER:body_sent_bytes} \"%{DATA:request_referer}\" \"%{DATA:user_agent}\""  
      }  
    }  
  }  
  else if [fields][document_type] == "nginx-error" {  
    grok {  
      match => {  
        "message" => "%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:loglevel}\] \[%{DATA:error_type}\] %{GREEDYDATA:error_message}"  
      }  
    }  
  }  
}  
  
output {  
  if [fields][document_type] == "nginx-access"  {  
    elasticsearch {  
      index => "nginx-access-%{+YYYY.MM.dd}"  
      hosts => ["https://192.168.101.31:9200"]  
      user => "elastic"  
      password => "IzjkJbO68HHDFc_c2Jf9"   
      cacert => "/etc/elkcert/http_ca.crt"   
    }  
  }  
  else if [fields][document_type] == "nginx-error"  {  
    elasticsearch {  
      index => "nginx-error-%{+YYYY.MM.dd}"  
      hosts => ["https://192.168.101.31:9200"]  
      user => "elastic"  
      password => "IzjkJbO68HHDFc_c2Jf9"   
      cacert => "/etc/elkcert/http_ca.crt"   
    }  
  }  
}  

состояние кластера curl -k -X GET -u elastic:password 'https://localhost:9200/_cluster/health?pretty'  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/edbd1271-0f7c-4cb7-85ce-8e85a067f402)

Состояние индексов curl -k -X GET -u elastic:password  'https://localhost:9200/_cat/indices'  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/56124c06-4849-46fa-929e-3dcbc127790f)  

(Yellow, потому что в кластере эластика только одна мастер-нода)  

Мониторинг самого себя с помощью метрикбит:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/aec9fc1c-2e37-4e05-b978-64a3f8164604)  

Доступные индексы:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/d58daeb2-937d-41ad-8373-870ae5edacc6)  

Хосты (установлен модуль system метрикбита):  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/72d6075c-d38a-4878-81e3-3a681a6e684e)  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/972b0d84-5520-4bb2-9c7c-d6fa065067bc)  

Nginx логи, стастика доступа к страницам:  
![image](https://github.com/SergeyM90/Diplom/assets/84016375/80457c58-6a7d-48c2-80b7-2147cc41f068)  

# Бэкапы

Также настроены ежедневные снапшоты дисков со сроком жизни не более недели:  
https://github.com/SergeyM90/Diplom/blob/main/Terraform/snapshot.tf
