# lab3_cloud
# Лабораторная работа: Создание собственной VPC в AWS

Выполнила: Новак Светлана (студентка группы IA2303).

---

## Шаг 1. Подготовка среды

Я вошла в **AWS Management Console**.  
Проверила, чтобы регион был установлен на **Frankfurt (eu-central-1)**.  
В строке поиска консоли ввела **“VPC”** и открыла сервис **VPC (Virtual Private Cloud)**.

---

## Шаг 2. Создание VPC

В левой панели я выбрала пункт **Your VPCs → Create VPC**.  
Далее заполнила параметры (мой порядковый номер: 21):

- **Name tag:** `student-vpc-k21`
- **IPv4 CIDR block:** `10.21.0.0/16`
- **Tenancy:** `Default`

После этого нажала кнопку **Create VPC**.  
Через несколько секунд появилась моя новая виртуальная сеть.
**Пояснение маски:** `/16` — это 65 536 адресов внутри сети; слишком крупные сети вроде `/8` в учебных задачах и в AWS не используются.

<img width="468" height="152" alt="image" src="https://github.com/user-attachments/assets/f5b6f418-244b-4370-9643-ce7946e88790" />

---

## Шаг 3. Создание Internet Gateway (IGW)

**Internet Gateway (IGW)** — это компонент, который позволяет ресурсам внутри моей VPC получать доступ в Интернет.  
Без него даже наличие публичного IP-адреса не даёт машине (например, EC2-инстансу) выхода во внешнюю сеть.

### Этапы выполнения

1. В левой панели консоли AWS я выбрала **Internet Gateways → Create internet gateway**.  
2. В поле **Name tag** указала:  
   `student-igw-k21`  
3. Нажала **Create internet gateway**, после чего шлюз появился в списке.

<img width="468" height="86" alt="image" src="https://github.com/user-attachments/assets/fc06230c-06a1-4139-8723-4e511e9fb8fd" />

4. Чтобы “присоединить” его к моей сети, я выбрала созданный IGW, нажала **Actions → Attach to VPC**,  
   а затем в списке выбрала свою сеть **student-vpc-k21**.  
5. Подтвердила действие нажатием **Attach internet gateway**.

Теперь мой интернет-шлюз прикреплён к VPC, и в будущем через него можно будет настроить доступ EC2-инстансов к Интернету (через публичную подсеть и маршрутную таблицу).

<img width="468" height="83" alt="image" src="https://github.com/user-attachments/assets/0fc92785-f5c5-42d5-91ff-797ba63ff509" />

---
## Шаг 4. Создание подсетей

**Подсети (Subnets)** — это сегменты внутри VPC, которые позволяют изолировать ресурсы.  
Например, одна подсеть может быть публичной (для веб-серверов с доступом из Интернета), а другая — приватной (для баз данных или внутренних сервисов).  
Такое разделение помогает гибко управлять безопасностью и маршрутизацией трафика.

---

### Шаг 4.1. Создание публичной подсети

Теперь, когда у нас уже есть **VPC** и **Internet Gateway**, создаём первую подсеть — **публичную**.  
Эта подсеть будет использоваться для ресурсов, которым нужен прямой доступ из Интернета (например, EC2-инстансов с веб-сервером).

**Этапы выполнения:**
1. В левой панели консоли AWS выбрала **Subnets → Create subnet**.  
2. Заполнила поля:
   - **VPC ID:** `student-vpc-k21`  
   - **Subnet name:** `public-subnet-k21`  
   - **Availability Zone:** `eu-central-1a`  
   - **IPv4 CIDR block:** `10.21.1.0/24`  
3. Нажала **Create subnet** — теперь подсеть появилась в списке.
   
<img width="468" height="88" alt="image" src="https://github.com/user-attachments/assets/acf10964-059d-44ce-9331-122fe510bd8a" />

**Ответ:**  
На данный момент подсеть **ещё не является публичной**, потому что она **не имеет маршрута в Интернет**.  
Публичной она станет только после того, как в её **таблицу маршрутов (Route Table)** будет добавлен маршрут `0.0.0.0/0` через **Internet Gateway (IGW)**.

---

### Шаг 4.2. Создание приватной подсети

Теперь создаю вторую подсеть — **приватную**.  
Она предназначена для размещения ресурсов, которые **не должны иметь прямого выхода в Интернет** (например, базы данных).

**Этапы выполнения:**
1. Ещё раз нажала **Create subnet**.  
2. Заполнила поля:
   - **VPC ID:** `student-vpc-k21`  
   - **Subnet name:** `private-subnet-k21`  
   - **Availability Zone:** `eu-central-1b`  
   - **IPv4 CIDR block:** `10.21.2.0/24`  
3. Нажала **Create subnet**.

   <img width="468" height="86" alt="image" src="https://github.com/user-attachments/assets/4f4fe8ca-07b7-4fc0-b107-23d2cd425673" />

**Ответ:**  
Подсеть **ещё не является приватной**, потому что в данный момент **её трафик не направляется никуда** — ни в Интернет, ни через NAT.  
Она станет приватной, когда для неё будет настроена **таблица маршрутов**, не содержащая выхода в Интернет, но, например, направленная только на внутренние ресурсы внутри VPC.

---

## Промежуточный итог

 Созданы две подсети:
- **public-subnet-k21** — предназначена для ресурсов с доступом в Интернет;  
- **private-subnet-k21** — предназначена для внутренних сервисов без прямого доступа во внешнюю сеть.  

Обе подсети связаны с одной VPC `student-vpc-k21`, что позволяет им обмениваться данными внутри сети.

## Шаг 5. Создание таблиц маршрутов (Route Tables)

Теперь, когда есть две подсети, я настроила таблицы маршрутов. По умолчанию у VPC есть одна **Main Route Table**, к которой автоматически привязаны новые подсети.  
Для изоляции и понятной структуры создала **две отдельные** таблицы маршрутов: одну для публичной подсети и одну — для приватной.

### Шаг 5.1. Публичная таблица маршрутов

1. **Route Tables → Create route table**  
   - **Name tag:** `public-rt-k21`  
   - **VPC:** `student-vpc-k21`  
   Нажала **Create route table**.

<img width="468" height="212" alt="image" src="https://github.com/user-attachments/assets/d2fc7e52-f4bf-43d2-95b3-dbd266469601" />

2. Открыла созданную таблицу → вкладка **Routes → Edit routes → Add route**:  
   - **Destination:** `0.0.0.0/0` *(весь внешний трафик)*  
   - **Target:** **Internet Gateway** → `student-igw-k21`  
   Сохранила изменения: **Save changes**.

<img width="468" height="138" alt="image" src="https://github.com/user-attachments/assets/8dd1dc9a-6776-45bf-9842-beb6ecc889ca" />

<img width="468" height="95" alt="image" src="https://github.com/user-attachments/assets/2f2a7cc9-8c01-45a0-859d-bf5917f02175" />

3. Вкладка **Subnet associations → Edit subnet associations** → отметила **`public-subnet-k21`** → **Save associations**.

<img width="468" height="133" alt="image" src="https://github.com/user-attachments/assets/79fb4398-6b70-4946-a6c5-9d1b399c8aa1" />

<img width="468" height="133" alt="image" src="https://github.com/user-attachments/assets/05f9f868-3771-479b-99f5-cf779697b5e4" />

**Почему нужно привязывать таблицу маршрутов к подсети?**  
Потому что именно **ассоциация** сообщает AWS, **какие** маршруты применять к трафику **какой** подсети. Без привязки подсеть продолжит использовать **Main Route Table**, и нужные правила могут не примениться.

**Итог:**
- Теперь у публичной подсети есть маршрут `0.0.0.0/0 → IGW`, что и делает её **публичной** (ресурсы с публичными IP смогут выходить в Интернет).

---

### Шаг 5.2. Приватная таблица маршрутов

1. **Create route table** ещё раз:  
   - **Name tag:** `private-rt-k21`  
   - **VPC:** `student-vpc-k21`  
   Нажала **Create route table**.

<img width="468" height="264" alt="image" src="https://github.com/user-attachments/assets/c958c133-12f8-4154-a7fd-f7f22c6e519a" />

2. **Subnet associations → Edit subnet associations** → отметила **`private-subnet-k21`** → **Save associations**.

<img width="468" height="105" alt="image" src="https://github.com/user-attachments/assets/261b3b4f-2dbc-436f-81b0-1fad8514fee1" />

5. На этом этапе **маршрутов наружу не добавляла** (NAT Gateway ещё не создан).

**Итог:**  
Все ресурсы, развёрнутые в `private-subnet-k21`, **пока не имеют доступа в Интернет**. Позже для исходящего доступа (например, для обновлений пакетов) я создам **NAT Gateway** в публичной подсети и добавлю в `private-rt-k21` маршрут `0.0.0.0/0 → NAT Gateway`.

---

## Промежуточный итог

 `public-rt-k21` с маршрутом `0.0.0.0/0 → student-igw-k21`, ассоциирована с `public-subnet-k21` — подсеть стала **публичной**.  
 `private-rt-k21` ассоциирована с `private-subnet-k21` — **исхода в Интернет нет** до создания NAT и добавления маршрута.

## Шаг 6. Создание NAT Gateway

**Зачем нужен NAT Gateway?**  
Чтобы инстансы в **приватной** подсети могли **исходяще** обращаться в Интернет (например, чтобы скачивать обновления), оставаясь **недоступными входящим соединениям** извне.

**Как работает NAT Gateway (кратко):**
- Инстансы из приватной подсети отправляют исходящий трафик на адрес по умолчанию (`0.0.0.0/0`), который в их таблице маршрутов направлен на **NAT Gateway**.
- NAT GW, находясь в **публичной** подсети и имея **Elastic IP (EIP)**, делает **SNAT** (подменяет исходный приватный адрес источника на свой публичный EIP) и отправляет трафик в Интернет через **IGW**.
- Ответы снаружи приходят на EIP NAT GW и перенаправляются обратно инициатору внутри приватной подсети.
- Входящие **неинициированные** соединения извне к ресурсам приватной подсети **не проходят**, поэтому инстансы остаются закрытыми.

> Важно: NAT Gateway должен находиться **в публичной подсети** и иметь маршрут к IGW. Его мы будем использовать как Target для `0.0.0.0/0` в приватной таблице маршрутов.

---

### Шаг 6.1. Создание Elastic IP

1. Открыла **Elastic IPs → Allocate Elastic IP address**.  
2. Нажала **Allocate** — получила новый **EIP** для использования в NAT Gateway.

<img width="468" height="196" alt="image" src="https://github.com/user-attachments/assets/9ad86bdd-6e18-49bd-9513-6ae663b692f8" />

---

### Шаг 6.2. Создание NAT Gateway

1. Перешла в **NAT Gateways → Create NAT gateway**.  
2. Заполнила поля:
   - **Name tag:** `nat-gateway-k21`
   - **Subnet:** `public-subnet-k21` *(NAT всегда создаётся в публичной подсети)*
   - **Connectivity type:** `Public`
   - **Elastic IP allocation ID:** выбрала EIP, созданный на предыдущем шаге
3. Нажала **Create NAT gateway**.  
4. Подождала, пока статус сменится с **Pending** на **Available** (обычно 1–3 минуты).

<img width="468" height="136" alt="image" src="https://github.com/user-attachments/assets/63c8a713-6e8c-4941-81a0-0fec12988edc" />

---

### Шаг 6.3. Изменение приватной таблицы маршрутов

1. Открыла **Route Tables** и выбрала `private-rt-k21`.  
2. Перешла во вкладку **Routes → Edit routes → Add route**:
   - **Destination:** `0.0.0.0/0`
   - **Target:** `NAT Gateway` → `nat-gateway-k21`
3. Нажала **Save changes**.

<img width="468" height="236" alt="image" src="https://github.com/user-attachments/assets/ff7377e9-b815-4ee9-b544-9bfb9715a48e" />

**Итог:**  
Теперь ресурсы в `private-subnet-k21` могут исходяще обращаться в Интернет (обновления, репозитории и т.д.) через **NAT Gateway**, оставаясь недоступными для входящих подключений из Интернета.

---

## Шаг 7. Создание Security Groups (SG)

**Security Group (SG)** — это виртуальный брандмауэр на уровне инстанса (EC2/ENI). Он фильтрует **входящий (Inbound)** и **исходящий (Outbound)** трафик.  
Отличия от NACL: SG — *stateful* (обратные ответы автоматически разрешены), NACL — *stateless*.

### 7.1. Web-сервер: `web-sg-k21`

1. **Security Groups → Create security group**
2. Поля:
   - **Security group name:** `web-sg-k21`
   - **Description:** `Security group for web server`
   - **VPC:** `student-vpc-k21`
3. **Inbound rules → Add rule:**
   - **Type:** `HTTP` | **Protocol:** `TCP` | **Port:** `80` | **Source:** `0.0.0.0/0`
   - **Type:** `HTTPS` | **Protocol:** `TCP` | **Port:** `443` | **Source:** `0.0.0.0/0`
4. **Outbound rules:** оставить по умолчанию `All traffic` → `0.0.0.0/0` (при необходимости ужесточить позже).

<img width="468" height="238" alt="image" src="https://github.com/user-attachments/assets/fe4338f9-4a16-4a07-a5c5-42ae4b2a2c04" />

### 7.2. Bastion host: `bastion-sg-k21`

1. **Create security group** снова
2. Поля:
   - **Security group name:** `bastion-sg-k21`
   - **Description:** `Security group for bastion host (SSH jump)`
   - **VPC:** `student-vpc-k21`
3. **Inbound rules → Add rule:**
   - **Type:** `SSH` | **Protocol:** `TCP` | **Port:** `22` | **Source:** `X.X.X.X/32` *(мой текущий публичный IP)*
4. **Outbound rules:** по умолчанию `All traffic` → `0.0.0.0/0` (нужно для SSH к приватным инстансам из подсети).

<img width="468" height="226" alt="image" src="https://github.com/user-attachments/assets/0afab780-b5e5-48b4-937c-0ac4610f1256" />

### 7.3. База данных: `db-sg-k21`

1. **Create security group** ещё раз
2. Поля:
   - **Security group name:** `db-sg-k21`
   - **Description:** `Security group for database`
   - **VPC:** `student-vpc-k21`
3. **Inbound rules → Add rule:**
   - **Type:** `MySQL/Aurora` | **Protocol:** `TCP` | **Port:** `3306` | **Source:** `web-sg-k21`  
     *(разрешаю доступ к БД только с веб-сервера — указываю **источник как SG**, а не CIDR)*
   - **Type:** `SSH` | **Protocol:** `TCP` | **Port:** `22` | **Source:** `bastion-sg-k21`  
     *(административный доступ к хосту БД — только через bastion)*
4. **Outbound rules:** оставить по умолчанию `All traffic` → `0.0.0.0/0` (можно ужесточить при необходимости: DNS, репозитории и т.п.).

<img width="468" height="233" alt="image" src="https://github.com/user-attachments/assets/30075300-1211-4c77-b489-13042d8065ed" />

<img width="468" height="120" alt="image" src="https://github.com/user-attachments/assets/e903445e-1566-47e3-872e-afd7ccd558b1" />

> **Важно:** В качестве источника правил выбирала **Security Group** (SG reference), а не подсеточный CIDR — так доступ будет предоставлен **конкретным инстансам** с этим SG, даже если их IP меняется.

---

## Что такое Bastion Host и зачем он нужен?

**Bastion Host** — это “прыжковый” (jump) хост в **публичной подсети**, имеющий **публичный IP**, через который администраторы подключаются (обычно по **SSH:22**) к инстансам в **приватных подсетях**.  
Он нужен потому, что:
- Инстансы в приватных подсетях **не имеют публичных IP** и не должны принимать входящие соединения из Интернета.
- Bastion предоставляет **единую точку входа**, которую легко **жёстко ограничить** (например, только с моего IP `/32`) и **мониторить**.
- Дальше подключение выполняется как через “мост”: `мой_ПК → bastion (публичная подсеть) → приватный инстанс` (SSH Agent Forwarding / ProxyJump / SSH туннель).

**Практики безопасности:**
- Разрешать SSH к bastion **только** с конкретных IP (`/32`), использовать **ключи**, а лучше — **AWS Systems Manager Session Manager** вместо открытого SSH.
- Включать **CloudWatch Logs**/Audit для сессий, MFA, ограничивать команды (например, с помощью `sshd_config`/IAM).
- Обновлять bastion, использовать минимальные AMI и автомасштабирование при необходимости.

---
## Шаг 8. Создание EC2-инстансов

На этом этапе я развернула три виртуальные машины (**EC2-инстанса**) с разными ролями:

- **web-server** — публичный веб-сервер, доступный из Интернета по HTTP.
- **db-server** — сервер базы данных, размещённый в приватной подсети.
- **bastion-host** — промежуточный хост (точка доступа) для безопасного подключения к приватным ресурсам.

---

### Подготовка

1. В строке поиска AWS Console ввела **EC2** и открыла консоль.
2. Нажала **Launch instance** (создать инстанс).
3. Для всех инстансов выбрала:
   - **AMI:** `Amazon Linux 2 AMI (HVM), SSD Volume Type`
   - **Тип:** `t3.micro` (бесплатный уровень)
   - **Key Pair:** создала новый ключ `student-key-k21` и скачала файл `.pem` для SSH-подключений.
   - **Хранилище:** по умолчанию (8 ГБ SSD).
   - **Теги:** Name — `web-server`, `db-server`, `bastion-host` (в зависимости от инстанса).

---

### 8.1. Веб-сервер (**web-server-k21**)

1. **Network Settings:**
   - VPC: `student-vpc-k21`
   - Subnet: `public-subnet-k21`
   - Auto-assign Public IP: `Enable`
   - Security Group: `web-sg-k21`

<img width="468" height="283" alt="image" src="https://github.com/user-attachments/assets/ce4b92f0-47f1-4579-a5be-22a9b06c5ae4" />

2. **User data (скрипт установки веб-сервера):**
   ```bash
   #!/bin/bash
   dnf install -y httpd php
   echo "<?php phpinfo(); ?>" > /var/www/html/index.php
   systemctl enable httpd
   systemctl start httpd
   ```
   <img width="468" height="86" alt="image" src="https://github.com/user-attachments/assets/864dbe26-712b-49f8-9434-dc84c1730a07" />

3. После запуска веб-сервер стал доступен по HTTP через публичный IP-адрес (порт 80).
   В браузере открыла `http://<Public-IP>` и убедилась, что отображается страница `phpinfo()` — значит, сервер работает.

<img width="468" height="351" alt="image" src="https://github.com/user-attachments/assets/0741ae4e-1e10-4e4a-a108-7a60ba37f667" />

---

### 8.2. Сервер базы данных (**db-server-k21**)

1. **Network Settings:**
   - VPC: `student-vpc-k21`
   - Subnet: `private-subnet-k21`
   - Auto-assign Public IP: `Disable`
   - Security Group: `db-sg-k21`
2. **User data (установка MariaDB):**
   ```bash
   #!/bin/bash
   dnf install -y mariadb105-server
   systemctl enable mariadb
   systemctl start mariadb
   mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'StrongPassword123!'; FLUSH PRIVILEGES;"
   ```
3. Инстанс не имеет публичного IP — к нему можно подключиться только через **bastion-host** по SSH, а затем изнутри выполнить запросы к MySQL.

---

### 8.3. Bastion Host (**bastion-host-k21**)

1. **Network Settings:**
   - VPC: `student-vpc-k21`
   - Subnet: `public-subnet-k21`
   - Auto-assign Public IP: `Enable`
   - Security Group: `bastion-sg-k21`
2. **User data (установка клиента MySQL):**
   ```bash
   #!/bin/bash
   dnf install -y mariadb105
   ```

3. Bastion-хост используется как “мост” для подключения к приватным серверам.
   С моего компьютера подключаюсь по SSH к bastion, затем с него — к `db-server` (используя его приватный IP).

---

### Проверка работы архитектуры

✅ **web-server-k21** — доступен из Интернета по HTTP и HTTPS.  
✅ **db-server-k21** — доступен только из приватной сети (через Bastion или web-server).  
✅ **bastion-host-k21** — позволяет безопасно подключаться к приватным инстансам.

---

## Итог

Я развернула три уровня инфраструктуры:
- **Публичный уровень:** bastion и web-server.
- **Приватный уровень:** база данных.
- **Безопасность обеспечивается:** VPC изолирует сеть, NAT Gateway управляет исходящим трафиком, а Security Groups — контролируют доступ между уровнями.

Теперь можно протестировать соединения:
- `мой_ПК → bastion-host (SSH)`
- `bastion-host → db-server (MySQL)`
- `web-server → db-server (MySQL)`

## Шаг 9. Проверка работы инфраструктуры

На этот момент у меня готовы:
- **VPC** `student-vpc-k21`
- Подсети: `public-subnet-k21` и `private-subnet-k21`
- **IGW** и **NAT Gateway**
- Таблицы маршрутов: `public-rt-k21` и `private-rt-k21`
- Три EC2: `web-server-k21`, `db-server-k21`, `bastion-host-k21`
- Три **Security Group**: `web-sg-k21`, `db-sg-k21`, `bastion-sg-k21`

Задача — убедиться, что веб доступен из Интернета, приватная подсеть изолирована, а исходящий трафик из приватной подсети работает через NAT.

---

### 9.1. Дождаться запуска инстансов

В консоли **EC2 → Instances** убедилась, что все три инстанса в статусе **Running** и **Status checks: 2/2 passed**.

<img width="468" height="94" alt="image" src="https://github.com/user-attachments/assets/d63ecbe9-0f13-40e6-8c8c-f44a227d4fbe" />

---

### 9.2. Проверка веб-сервера (HTTP)

1. В списке инстансов открыла карточку **web-server-k21** и скопировала **Public IPv4 address**.
2. В браузере открыла:  
   `http://<WEB_PUBLIC_IP>/`  
   Должна отобразиться страница **phpinfo()** — это подтверждает, что:
   - работает Apache (httpd) и PHP,
   - корректно настроена **публичная подсеть** и маршрут `0.0.0.0/0 → IGW`,
   - SG `web-sg-k21` разрешает **HTTP (80)** из `0.0.0.0/0`.

<img width="468" height="351" alt="image" src="https://github.com/user-attachments/assets/abf6cfa6-6c88-44e3-a13f-276f50f06419" />

> Альтернатива из терминала (на bastion или локально):  
> `curl -I http://<WEB_PUBLIC_IP>/` — ожидаю `HTTP/1.1 200 OK`.

---

### 9.3. Подключение к bastion-host по SSH

На своём компьютере выполнила подключение к **bastion-host-k21** (используя загруженный ключ `student-key-k21.pem`):

```bash
ssh -i student-key-k21.pem ec2-user@<BASTION_PUBLIC_IP>
```

<img width="468" height="164" alt="image" src="https://github.com/user-attachments/assets/3903833f-3f0a-4965-beda-b9564de03d9d" />

Если подключение прошло — значит SG `bastion-sg-k21` корректно настроен и пускает **SSH (22)** только с моего IP (`/32`).

---

### 9.4. Проверка выхода в Интернет с bastion

На bastion выполнила:

```bash
ping -c 4 google.com
```

<img width="468" height="178" alt="image" src="https://github.com/user-attachments/assets/79b2f1f8-2cea-4c82-9137-24f696e9b5c2" />

Ожидаю ответы (time ~XX ms). Это подтверждает:
- Публичная подсеть имеет маршрут через **IGW**,
- Исходящий трафик открыт (Outbound по умолчанию у SG),
- DNS резолвится (при необходимости можно проверить `dig google.com` после `dnf install -y bind-utils`).

---

### 9.5. Подключение с bastion к db-server (приватный IP)

1. В консоли EC2 посмотрела **Private IPv4 address** инстанса **db-server-k21**.
2. С bastion выполнила подключение к MySQL на БД по приватному IP:

```bash
mysql -h <DB_PRIVATE_IP> -u root -p
# ввести пароль: StrongPassword123!
```

<img width="468" height="113" alt="image" src="https://github.com/user-attachments/assets/1d5a5100-3316-4bf0-8b09-a77cb593651c" />

Успешное подключение подтверждает:
- Работает маршрут из приватной подсети через **NAT Gateway** для исходящих (если БД тянет обновления/репозитории),
- SG `db-sg-k21` разрешает **3306** **только** источнику `web-sg-k21` (доступ к MySQL с bastion не обязателен для продакшена — можно временно добавить правило для админ-доступа, как в шаге 7),
- Приватная подсеть **не имеет прямого входа из Интернета** (нет Public IP и нет маршрута на IGW).

> Проверка недоступности из Интернета: попытка подключиться к БД напрямую по публичному IP **невозможна**, потому что у инстанса его нет; а попытка по приватному IP снаружи тоже невозможна — адреса RFC1918 не маршрутизируются в Интернет.

---

### 9.6. Выход из сессий

```bash
exit  # выйти из MySQL
exit  # выйти с bastion на мой локальный ПК
```

---

После выполнения всех шагов, я удалила созданные ресурсы в AWS. 

### Вывод
В этой лабораторной работе я развернула защищённую облачную инфраструктуру в AWS: создала VPC с публичной и приватной подсетями, подключила IGW и NAT Gateway, настроила отдельные таблицы маршрутов для каждой подсети, определила правила доступа через три Security Group и запустила три EC2-инстанса (web, db и bastion) с нужными скриптами и параметрами сети. Я проверила доступность веб-сервера по публичному IP, подключение по SSH к bastion и доступ к базе данных только из приватной сети, подтвердив корректность маршрутизации и изоляции. Итогом стало понимание на практике принципов сетевой сегментации, stateful-безопасности SG, роли IGW/NAT, а также умение строить минимально необходимую, но расширяемую архитектуру “web–app/db–bastion” в AWS.
