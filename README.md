[Dockerfile.txt](https://github.com/SergeyM90/Diplom/files/13467030/Dockerfile.txt)# Дипломная работа по профессии «Системный администратор»  

# ЗАДАЧА  


Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в Yandex Cloud и отвечать минимальным стандартам безопасности.


# ОТВЕТ

![image](https://github.com/SergeyM90/Diplom/assets/84016375/13e62fa8-e798-44e0-ba17-767e87296234)

# Технические детали

Развертывание инфраструктуры автоматизировано с помощью Terraform, Ansible и Gitlab.
Все этапы выполняются в одном пайплайне.
Self-hosted Gitlab работает на локальном домашнем сервере. Раннер запускается в докер-контейнере. Несколько подробнее задокументировал https://github.com/SergeyM90/Diplom/blob/main/Details.md
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



