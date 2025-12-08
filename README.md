# Лабораторная работа: Создание виртуальной сети (VPC) в AWS

## **Цель моей работы**

В рамках данной лабораторной работы я вручную создала виртуальную сеть (VPC) в AWS и настроила полноценную сетевую инфраструктуру, включающую:

* виртуальную частную сеть (VPC),
* публичные и приватные подсети,
* интернет-шлюз (Internet Gateway),
* NAT Gateway,
* отдельные таблицы маршрутов,
* группы безопасности (Security Groups),
* три EC2-инстанса: веб-сервер, bastion-host и сервер базы данных.

В процессе выполнения я научилась:

* понимать принципы сетевой изоляции в AWS,
* настраивать маршрутизацию между подсетями,
* управлять входящими и исходящими соединениями через Security Groups,
* пользоваться bastion-хостом для доступа в приватную подсеть,
* развертывать EC2-инстансы с автоматической установкой ПО.


#  **Описание созданной мной архитектуры**

В результате работы мной была развёрнута следующая сеть:

* **VPC**, внутри которой изолированы все ресурсы.
* **Публичная подсеть** — для веб-сервера и bastion-host.
* **Приватная подсеть** — для сервера базы данных.
* **Internet Gateway** — обеспечивает доступ в интернет для публичной подсети.
* **NAT Gateway** — даёт приватной подсети исходящий доступ в интернет.
* **Две таблицы маршрутов** — отдельная для публичной и приватной подсети.
* **Security Groups** — три группы с разными уровнями разрешений.
* **EC2-инстансы**:

  * Web-server (публичная подсеть)
  * Bastion-host (публичная подсеть)
  * DB-server (приватная подсеть)


#  **Ход выполнения работы**


# 1. Подготовка среды

Я вошла в AWS Management Console, переключила регион на **Frankfurt (eu-central-1)** и открыла сервис **VPC**.

# 2. Создание VPC

Я создала VPC со следующими параметрами:

* **Name:** `student-vpc-k13`
* **CIDR блок:** `10.13.0.0/16`
* **Tenancy:** Default
<img width="1280" height="388" alt="Image" src="https://github.com/user-attachments/assets/a7f4b3f2-b904-4ca5-a4c7-81f113d13b18" />

### Ответ на контрольный вопрос

**Маска /16** означает, что сеть содержит ~65 536 адресов.
Использовать `/8` нельзя — это слишком большая сеть и выходит за общепринятые рекомендации AWS.


# 3. Создание Internet Gateway (IGW)

Я создала IGW `student-igw-k13` и прикрепила его к моей VPC.
С этого момента подсети внутри VPC могут получать доступ в интернет (если маршрутизация настроена правильно).

<img width="953" height="664" alt="Image" src="https://github.com/user-attachments/assets/15a3e640-c17b-4d3c-aef4-37115e23a761" />

# 4. Создание подсетей

## 4.1 Публичная подсеть

* **Name:** `public-subnet-k13`
* **CIDR:** `10.13.1.0/24`
* **AZ:** eu-central-1a
<img width="1280" height="245" alt="Image" src="https://github.com/user-attachments/assets/29ee0d99-88f9-4ae6-b839-3d66a743bd58" />
###  Ответ на контрольный вопрос

На данном этапе подсеть НЕ является публичной, потому что маршруты ещё не направлены в Internet Gateway.

## 4.2 Приватная подсеть

* **Name:** `private-subnet-k13`
* **CIDR:** `10.13.2.0/24`
* **AZ:** eu-central-1b
<img width="891" height="783" alt="Image" src="https://github.com/user-attachments/assets/f49b7125-b123-47cf-9bda-94843436178a" />

### Ответ на контрольный вопрос

Приватная подсеть пока тоже не приватная. Она станет приватной только после настройки таблицы маршрутов и отсутствия публичных IP на инстансах.

# 5. Создание таблиц маршрутов

## Публичная таблица маршрутов

* **Name:** `public-rt-k13`
* Я добавила маршрут:
<img width="955" height="737" alt="Image" src="https://github.com/user-attachments/assets/c22a4825-cea0-404a-a9e8-558b2dbbc953" />
  * `0.0.0.0/0` → **Internet Gateway**
* Ассоциировала таблицу с `public-subnet-kXX`

<img width="935" height="558" alt="Image" src="https://github.com/user-attachments/assets/7d009b7f-fe37-4a60-941d-1179966f0806" />

Теперь публичная подсеть получила выход в интернет.

### Контрольный вопрос

Я привязываю таблицу маршрутов к подсети, чтобы подсеть использовала заданные маршруты, а не маршруты по умолчанию.

## Приватная таблица маршрутов
<img width="955" height="737" alt="Image" src="https://github.com/user-attachments/assets/7e99ec31-7b9b-457c-b1ae-26d7b311df79" />
* **Name:** `private-rt-k13`
* Ассоциирована с `private-subnet-k13`
<img width="951" height="599" alt="Image" src="https://github.com/user-attachments/assets/96ef880b-f577-46a4-8cfe-4a41c1bfb545" />
Пока что приватная подсеть не имеет доступа в интернет — NAT Gateway ещё не создан.

# 6. Создание NAT Gateway

## 6.1 Создание Elastic IP

Я создала новый статический публичный IP (Elastic IP).

<img width="926" height="780" alt="Image" src="https://github.com/user-attachments/assets/4978ea54-09b4-4d7b-a599-6121d26b629c" />

## 6.2 Создание NAT Gateway

* **Name:** `nat-gateway-k13`
* Размещён в публичной подсети
* Выбрана режим "Zonal"
* К NAT привязан Elastic IP

<img width="913" height="785" alt="Image" src="https://github.com/user-attachments/assets/bb3b5738-6b48-4226-a430-d4bed2d0a946" />
После нескольких минут NAT стал доступным.

## 6.3 Настройка маршрутов приватной таблицы

В `private-rt-k13` я добавила:

* `0.0.0.0/0` → **NAT Gateway**

<img width="923" height="752" alt="Image" src="https://github.com/user-attachments/assets/4b07adee-024f-49c8-bd60-9335958be428" />
Теперь приватная подсеть имеет доступ в интернет, но остаётся недоступной извне.

### Контрольный вопрос: как работает NAT Gateway?

Он подменяет исходящие запросы приватных инстансов своим публичным IP, скрывая внутренние адреса и запрещая внешние подключения.

# 7. Создание Security Groups

Я создала три группы безопасности:

### **web-sg-k13:**

Разрешила вход:

* HTTP (80) — всему миру
* HTTPS (443) — всему миру
<img width="894" height="754" alt="Image" src="https://github.com/user-attachments/assets/05abba98-96fc-4319-ac3e-5f77287b13d8" />
### **bastion-sg-k13:**

Разрешила вход:

* SSH (22) — **только с моего IP**

### **db-sg-k13:**

Разрешила:

* MySQL (3306) — от web-sg-k13
* MySQL (3306) — от bastion-sg-k13
* SSH — только от bastion-host-к13

###  Контрольный вопрос: кто такой Bastion Host?

Это промежуточный защищённый сервер, через который можно подключаться к ресурсам в приватной подсети.

# 8. Создание EC2-инстансов

## Web-server

* Размещён в публичной подсети
* Получил публичный IP
* Применена группа безопасности web-sg
* В User Data установился Apache и PHP-файл `phpinfo()`
<img width="881" height="763" alt="Image" src="https://github.com/user-attachments/assets/99b575f8-1cf0-442d-801a-df0143bd4964" />
<img width="860" height="541" alt="Image" src="https://github.com/user-attachments/assets/da231739-a0a4-4b77-b821-a26f10f2bb63" />
## DB-server

* Размещён в приватной подсети
* Публичного IP нет
* Установлена MariaDB
* root-пароль был изменён

<img width="861" height="789" alt="Image" src="https://github.com/user-attachments/assets/47b50eae-ee8b-4c83-86cf-29ee7dc46ca8" />

<img width="854" height="308" alt="Image" src="https://github.com/user-attachments/assets/a42d7abd-d312-49f2-8369-6f1e59850cc0" />

## Bastion-host

* Размещён в публичной подсети
* Использует группу безопасности с SSH-доступом
* Через него я подключалась в приватную подсеть
<img width="867" height="782" alt="Image" src="https://github.com/user-attachments/assets/70666771-a054-47d2-96d4-185f029f5a04" />

<img width="867" height="258" alt="Image" src="https://github.com/user-attachments/assets/ed32b19f-e753-4d92-9ee5-e2e6bbe44b0a" />
# 9. Проверка работы

Я выполнила:

###  Проверку веб-сервера

В браузере открылась страница phpinfo().
<img width="948" height="1045" alt="Image" src="https://github.com/user-attachments/assets/e4343520-028d-4db1-b623-86210604da89" />

###  Подключение к bastion-host по SSH

Работает корректно.
<img width="948" height="494" alt="Image" src="https://github.com/user-attachments/assets/6d832d18-1a15-4757-b348-5f585b659455" />
###  Проверка интернета внутри bastion

Команда `ping google.com` возвращала ответы — значит IGW и маршруты настроены правильно.

<img width="963" height="353" alt="Image" src="https://github.com/user-attachments/assets/508ebe21-39bd-4a2c-9823-317305934733" />

###  Подключение к базе данных с bastion

Команда:

```bash
mysql -h <private-ip-db> -u root -p
```

позволила подключиться к MariaDB, что подтверждает правильность конфигурации приватной подсети и NAT Gateway.

---

# 10. Завершение работы и удаление ресурсов

Чтобы предотвратить расходы, я удалила:

* все EC2-инстансы,
* NAT Gateway,
* Elastic IP.

После этого я вручную удалила:

* ассоциации таблиц маршрутов,
   <img width="660" height="472" alt="Image" src="https://github.com/user-attachments/assets/b5844e45-5dba-4706-9752-611741ecf52e" />
* сами таблицы маршрутов,
  <img width="620" height="489" alt="Image" src="https://github.com/user-attachments/assets/eaaf4027-d0e1-4b54-8ada-e9a782c4d06d" />
* подсети,
* Security Groups,
  <img width="632" height="654" alt="Image" src="https://github.com/user-attachments/assets/66e982c7-070d-4fb1-ad20-d4c463f1dd01" />
* Internet Gateway,
  <img width="581" height="340" alt="Image" src="https://github.com/user-attachments/assets/348fd5c8-a9f8-4eac-9800-fd8b0539a1eb" />
* VPC.
  <img width="604" height="387" alt="Image" src="https://github.com/user-attachments/assets/086a8b12-b003-4681-88b3-986eb11b1d89" />

Все ресурсы были успешно удалены в корректном порядке.

#  Итоги

По завершению работы я:

* успешно создала и настроила изолированную архитектуру AWS VPC,
* настроила публичные и приватные подсети,
* обеспечила доступ в интернет с помощью IGW и NAT,
* настроила доступ к приватной подсети через bastion-host,
* развернула и проверила взаимодействие EC2-инстансов,
* корректно освободила ресурсы.


