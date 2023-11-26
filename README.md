# Дипломная работа по профессии «Системный администратор»  

# ЗАДАЧА  


Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в Yandex Cloud и отвечать минимальным стандартам безопасности.


# ОТВЕТ

![image](https://github.com/SergeyM90/Diplom/assets/84016375/13e62fa8-e798-44e0-ba17-767e87296234)

# Технические детали

Развертывание инфраструктуры автоматизировано с помощью Terraform, Ansible и Gitlab.
Все этапы выполняются в одном пайплайне.
Self-hosted Gitlab работает на локальном домашнем сервере. Раннер запускается в докер-контейнере. Несколько подробнее задокументировал [здесь](https://github.com/SergeyM90/Diplom/blob/main/Details.md)
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
Инфраструктура в Терраформе: ./terraform/
Провижн необходимых сервисов с помощью Ансибл:
Postgres cluster: ./ansible/postgres/ (адаптирован плейбук https://github.com/vitabaks/postgresql_cluster)
./ansible/postgres/inventory
./ansible/postgres/deploy_pgcluster.yml
ELK, Zabbix, nginx: ./ansible/project/
./ansible/project/inventory
./ansible/project/deploy_project.yml
