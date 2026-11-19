# Практическая работа №4 (Web). Web Application Firewall (ModSecurity 3)

## Цель работы
- Получить знания по установке, настройке и работе Web Application Firewall.
- Научиться самостоятельно разворачивать WAF в базовой конфигурации.

## Задание
1. Используя стенд первой практической работы, изменить инфраструктуру по схеме: машина
   с WAF должна быть **прокси** для веб-сервера из первой ПР.
2. Установить WAF (ModSecurity 3) на машину WAF; показать, что сайт открывается при
   подключении к машине WAF.
3. Настроить WAF с детектом OWASP Top 10.
4. Провести 3 атаки из списка OWASP Top 10 и показать срабатывание правил WAF.

> Схема: `Клиент → [WAF: nginx + ModSecurity 3] → [Web-сервер из ПР №1]`
> 📷 СКРИНШОТ: каждый шаг и каждое срабатывание правила.

---

## Теория
WAF (Web Application Firewall) — межсетевой экран уровня приложений: анализирует HTTP(S)
трафик и блокирует запросы с признаками атак (SQLi, XSS и др.). ModSecurity — популярный
open-source WAF; версия 3 (libmodsecurity) работает с nginx через коннектор. Набор правил
детектирования — **OWASP Core Rule Set (CRS)**, покрывающий OWASP Top 10.

---

## Ход работы

### 1. Подготовка машины WAF (Debian/Ubuntu)
```bash
sudo apt update
sudo apt install -y nginx git build-essential libpcre3-dev zlib1g-dev libxml2-dev \
  libyajl-dev liblmdb-dev libgeoip-dev libcurl4-openssl-dev automake libtool pkg-config
```

### 2. Сборка libmodsecurity (ModSecurity 3)
```bash
cd /opt
sudo git clone --depth 1 https://github.com/SpiderLabs/ModSecurity
cd ModSecurity
sudo git submodule init && sudo git submodule update
sudo ./build.sh
sudo ./configure
sudo make
sudo make install
```

### 3. Сборка коннектора ModSecurity-nginx
```bash
cd /opt
sudo git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx
# узнать версию nginx
nginx -v
# скачать исходники той же версии nginx и собрать динамический модуль
wget http://nginx.org/download/nginx-<версия>.tar.gz
tar -xzf nginx-<версия>.tar.gz && cd nginx-<версия>
sudo ./configure --with-compat --add-dynamic-module=/opt/ModSecurity-nginx
sudo make modules
sudo cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules/
```

### 4. Подключить модуль в nginx
В начало `/etc/nginx/nginx.conf`:
```nginx
load_module modules/ngx_http_modsecurity_module.so;
```

### 5. Базовая конфигурация ModSecurity
```bash
sudo mkdir -p /etc/nginx/modsec
sudo cp /opt/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf
sudo cp /opt/ModSecurity/unicode.mapping /etc/nginx/modsec/
```
Включить блокировку (а не только детект) — в `/etc/nginx/modsec/modsecurity.conf`:
```
SecRuleEngine On
```
Создать главный файл правил `/etc/nginx/modsec/main.conf`:
```
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/coreruleset/crs-setup.conf
Include /etc/nginx/modsec/coreruleset/rules/*.conf
```

### 6. Установить OWASP Core Rule Set (детект OWASP Top 10)
```bash
cd /etc/nginx/modsec
sudo git clone --depth 1 https://github.com/coreruleset/coreruleset
sudo mv coreruleset/crs-setup.conf.example coreruleset/crs-setup.conf
```

### 7. Настроить nginx как реверс-прокси к веб-серверу из ПР №1
В `/etc/nginx/conf.d/waf.conf`:
```nginx
server {
    listen 80;
    server_name _;

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    location / {
        proxy_pass http://192.168.1.50;        # IP веб-сервера из ПР №1
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
Проверить и перезапустить:
```bash
sudo nginx -t
sudo systemctl restart nginx
```

### 8. Проверка, что сайт открывается через WAF
В браузере открыть `http://<IP-машины-WAF>/` — должен отобразиться сайт, физически
работающий на сервере из ПР №1. Трафик идёт через WAF.
📷 СКРИНШОТ сайта, открытого через адрес WAF.

### 9. Три атаки из OWASP Top 10 и срабатывание WAF
Выполнять с клиента по адресу WAF. Логи срабатываний — `/var/log/modsec_audit.log` и
`/var/log/nginx/error.log`.

**Атака 1 — SQL-инъекция (A03 Injection):**
```bash
curl "http://<IP-WAF>/?id=1' OR '1'='1"
```
Ожидаемо: WAF вернёт `403 Forbidden`, в логе — правило группы `942xxx` (SQLi).

**Атака 2 — XSS (A03 Injection):**
```bash
curl "http://<IP-WAF>/?search=<script>alert(1)</script>"
```
Ожидаемо: `403 Forbidden`, правило группы `941xxx` (XSS).

**Атака 3 — Path traversal (A01 Broken Access Control):**
```bash
curl "http://<IP-WAF>/?file=../../../../etc/passwd"
```
Ожидаемо: `403 Forbidden`, правило группы `930xxx` (LFI/path traversal).

Посмотреть сработки:
```bash
sudo tail -f /var/log/modsec_audit.log
```
📷 СКРИНШОТ ответов 403 и записей в modsec_audit.log для всех трёх атак.

### 10. Проверка, что легитимный трафик проходит
```bash
curl -I "http://<IP-WAF>/"        # обычный запрос → 200 OK
```

---

## Контрольные вопросы
- **Что такое WAF?** — межсетевой экран уровня приложений, фильтрующий HTTP-трафик по признакам атак.
- **Что такое OWASP CRS?** — Core Rule Set, готовый набор правил ModSecurity против OWASP Top 10.
- **Разница SecRuleEngine On / DetectionOnly?** — `On` блокирует запросы, `DetectionOnly` только логирует.
- **Что означает ответ 403 от WAF?** — запрос распознан как атака и заблокирован.
- **Где смотреть сработки?** — `/var/log/modsec_audit.log`.
- **Зачем WAF ставить проксёй?** — весь трафик к веб-серверу проходит фильтрацию до попадания на приложение.

## Вывод
Развёрнут WAF ModSecurity 3 на nginx в режиме реверс-прокси перед веб-сервером из ПР №1.
Подключён OWASP Core Rule Set для детекта OWASP Top 10, включён режим блокировки. Тремя
атаками (SQL-инъекция, XSS, path traversal) продемонстрировано срабатывание правил WAF
(ответы 403 и записи в audit-логе), при этом легитимные запросы проходят нормально.
