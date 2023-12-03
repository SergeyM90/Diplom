# Дипломная работа по профессии «Системный администратор»  

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
Инфраструктура в Терраформе: [./terraform/](https://github.com/SergeyM90/Diplom/tree/main/Terraform)
Провижн необходимых сервисов с помощью Ансибл:  
Postgres cluster: (https://github.com/SergeyM90/Diplom/tree/main/ansible/postgres) (адаптирован плейбук https://github.com/vitabaks/postgresql_cluster)  
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
https://github.com/SergeyM90/Diplom/blob/main/pg2.jpg  


