# Лабораторная работа №2

**Тема:** Знакомство с LXC-контейнерами в Proxmox  
**Выполнил студент**: Ряднов Вениамин Сергеевич  

---

## Шаги выполнения работы

### 1. Проверка сетевого интерфейса
- Перейти на ноду **proxmox**.  
- Открыть раздел **System -> Network**.  
- Проверить текущий сетевой интерфейс (например, **enp3s0** или **eth0**).
![Скриншот_1](images/1.png) 

### 2. Создание сетевого моста (Linux Bridge)
- Перейти в раздел **System -> Network**.  
- Нажать **Create -> Linux Bridge**.  
- Указать:
  - **IP-адрес**: 10.0.2.15/24.  
  - **Шлюз (Gateway)**: 10.0.2.2 (проверить через `ip r`).  
  - **Порт бриджа**: имя основного интерфейса (например, **enp0s3**).
![Скриншот_1](images/2.png)
![Скриншот_1](images/3.png)
- Нажать **Apply configuration** (перед этим перепроверить параметры).  

⚠️ **Важно:** Ошибка на этом шаге может привести к потере доступа к Proxmox, требуя повторной настройки с нуля!

### 3. Загрузка шаблона контейнера (Turnkey Redis 18.0)
- Перейти в раздел **local (storage)** в меню слева на вкладке ноды **proxmox**.  
- Выбрать вкладку **CT Templates**.  
- Нажать **Templates**.  
- Найти и скачать шаблон **Turnkey Redis 18.0**.  
  Если шаблона нет, загрузить его вручную через консоль Proxmox:
  ```bash
  pveam update
  pveam available
  pveam download local debian-12-turnkey-redis_18.0-1_amd64.tar.gz
  ```
  ![Скриншот_1](images/4.png)
- Убедиться, что шаблон появился в списке **CT Templates**.

### 4. Создание контейнера с Redis
- Нажать **Create CT**.  
- Настройки по умолчанию, кроме:
  - **IPv4**: выбрать **DHCP**.  
  - Выбрать шаблон **Turnkey Redis 18.0**.  
- Записать **Hostname** и **пароль**.  
![Скриншот_1](images/5.png)
![Скриншот_1](images/6.png)
![Скриншот_1](images/7.png)
![Скриншот_1](images/8.png)

### 5. Инициализация контейнера
- Если создание завершилось с **TASK OK**, войти в контейнер через **Console**.  
- Логин: **root**, пароль – указанный при создании.  
![Скриншот_1](images/9.png)

### 6. Настройка Redis
- Придумать пароль для админки Redis (можно любой).  
- Выбрать интерфейс **all**.  
- Выключить **Protected-mode**.  
- Пропустить остальные шаги (используя **Tab** и стрелки).
- Сохранить финальный экран с адресами и портами Redis. 
![Скриншот_1](images/10.png)
![Скриншот_1](images/11.png) 

### 7. Загрузка шаблона Nextcloud
- Скачать образ Nextcloud в **CT Templates**.
![Скриншот_1](images/12.png)

### 8. Создание контейнера Nextcloud через CLI
- Подключиться по SSH или через консоль VirtualBox.  
- Выполнить команду:
```bash
pvesh create /nodes/proxmox/lxc -vmid 101 -hostname nextcloud \
-unprivileged true -storage local -password "пароль" \
-net0 "name=eth0,bridge=vmbr0,ip=dhcp,firewall=yes" \
-ostemplate local:vztmpl/debian-12-turnkey-nextcloud_18.1-1_amd64.tar.gz \
-memory 512
```
![Скриншот_1](images/13.png)

### 9. Проверка создания контейнера
- Войти в веб-интерфейс Proxmox и проверить наличие нового контейнера.  
- Подключиться к нему через **Console**.
![Скриншот_1](images/14.png)

### 10. Настройка Nextcloud
- Выполнить начальную настройку как в шаге 6.
- На экране с Redis выбрать **Skip**.  
- В качестве **Nextcloud Domain** указать **localhost**.
![Скриншот_1](images/15.png)
![Скриншот_1](images/16.png)

### 11. Проброс портов
- Добавить проброс портов:  
  - **127.0.0.1:443 -> nextcloud:443**.  
- Проверить доступность Nextcloud по адресу:  
```
https://localhost/
```
- Ожидаемая ошибка сервиса – настраиваем Redis позднее.
![Скриншот_1](images/17.png)

### 12. Настройка подключения к Redis
- Открыть файл конфигурации Nextcloud:
```
nano /var/www/nextcloud/config/config.php
```
- Обновить IP и порт Redis (из шага 6).
- Отключить **Memcache**:
```
Удалить строку 'memcache.local'.
```
![Скриншот_1](images/18.png)

### 13. Проверка Nextcloud
- Перейти по адресу:  
```
https://localhost/
```
- Убедиться, что форма входа в Nextcloud открывается без ошибок.  
![Скриншот_1](images/19.png)

---

## Вопросы для размышления

1. **Почему неправильная конфигурация на шаге 2 приведет (вероятнее всего) к полной потере сетевой доступности?**
   - Неправильный IP-адрес или шлюз в бридже может привести к конфликтам в сети или потере маршрутизации. В результате Proxmox потеряет соединение с сетью, включая доступ к веб-интерфейсу. Исправить это можно только вручную через консоль, если есть физический доступ.

2. **Почему адрес шлюза (default gateway) виртуальной машины выглядит как 10.0.2.2, а не 10.0.2.1?**
   - Адрес 10.0.2.2 – это шлюз, созданный VirtualBox в режиме NAT. Он обрабатывает трафик между виртуальной машиной и внешней сетью, а 10.0.2.1 часто зарезервирован для других целей (например, маршрутизации внутри VirtualBox).

---

## Вывод
Все шаги по настройке контейнеров в Proxmox успешно выполнены. В результате были созданы и настроены контейнеры с Redis и Nextcloud. Приложения доступны для дальнейшей работы и тестирования.
