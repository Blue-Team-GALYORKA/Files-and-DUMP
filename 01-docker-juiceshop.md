# Практическая работа №1 (Web). Docker и OWASP Juice Shop

## Цель работы
- Познакомиться с основами WEB-технологий и основными протоколами.
- Научиться разворачивать веб-сервер с помощью Docker.
- Научиться создавать запросы к веб-приложению.

## Задание
1. Установить и настроить Ubuntu 20.04.
2. Настроить подключение к репозиториям Docker.
3. Установить Docker CE.
4. Установить веб-приложение `bkimminich/juice-shop`.
5. Изучить запросы браузера к приложению и форме регистрации.
6. Сформировать запрос к приложению через `curl`.
7. Сформировать запрос через PowerShell.
8. Изучить ответ приложения и распарсить его утилитой `jq`.

> 📷 СКРИНШОТ: каждый шаг.

---

## Ход работы

### 1. Ubuntu 20.04
Установить Ubuntu 20.04 в ВМ. Обновить пакеты:
```bash
sudo apt update && sudo apt upgrade -y
```

### 2–3. Установка Docker CE из официального репозитория
```bash
# зависимости
sudo apt install -y ca-certificates curl gnupg lsb-release
# GPG-ключ Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
# репозиторий
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# установка
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
# проверка
sudo systemctl enable --now docker
sudo docker --version
sudo docker run hello-world
```
📷 СКРИНШОТ версии Docker и hello-world.

### 4. Запуск OWASP Juice Shop
```bash
sudo docker pull bkimminich/juice-shop
sudo docker run -d -p 3000:3000 --name juice bkimminich/juice-shop
sudo docker ps                 # контейнер juice в статусе Up
```
Открыть в браузере: `http://localhost:3000` — загрузится интернет-магазин Juice Shop
(специально уязвимое приложение OWASP для обучения).
📷 СКРИНШОТ главной страницы Juice Shop.

### 5. Изучение запросов браузера (DevTools)
Открыть **F12 → вкладка Network**. Обновить страницу и выполнить регистрацию:
- `GET /rest/products/search?q=` — поиск товаров;
- `GET /api/Products` — список товаров (JSON);
- `POST /api/Users` — регистрация пользователя (тело JSON с email и password).
Посмотреть заголовки запроса/ответа, тело, код состояния (200/201).
📷 СКРИНШОТ вкладки Network с запросом формы регистрации.

### 6. Запрос через curl
Получить список товаров:
```bash
curl http://localhost:3000/api/Products
```
Запрос с заголовками и просмотром кода ответа:
```bash
curl -i http://localhost:3000/rest/products/search?q=apple
```
Регистрация пользователя через curl (POST с JSON):
```bash
curl -X POST http://localhost:3000/api/Users \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"Passw0rd!"}'
```
📷 СКРИНШОТ ответа curl.

### 7. Запрос через PowerShell (на Windows)
```powershell
# простой GET
Invoke-RestMethod -Uri "http://localhost:3000/api/Products" -Method Get

# POST-регистрация
$body = @{ email = "ps@test.com"; password = "Passw0rd!" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:3000/api/Users" -Method Post `
  -ContentType "application/json" -Body $body
```
📷 СКРИНШОТ вывода PowerShell.

### 8. Парсинг ответа утилитой jq
```bash
sudo apt install -y jq
# вывести только названия товаров
curl -s http://localhost:3000/api/Products | jq '.data[].name'
# количество товаров
curl -s http://localhost:3000/api/Products | jq '.data | length'
# id и цена каждого товара
curl -s http://localhost:3000/api/Products | jq '.data[] | {id, price}'
```
`jq` — потоковый обработчик JSON: `.data[]` разворачивает массив, `.name` достаёт поле.
📷 СКРИНШОТ распарсенного вывода jq.

---

## Контрольные вопросы
- **Что такое Docker и контейнер?** — платформа для запуска приложений в изолированных контейнерах со всеми зависимостями.
- **Чем `docker run -d -p 3000:3000` отличается от без флагов?** — `-d` фон (detached), `-p` проброс порта хост:контейнер.
- **Что такое OWASP Juice Shop?** — учебное намеренно уязвимое веб-приложение для тренировки по безопасности.
- **Чем curl отличается от Invoke-RestMethod?** — это аналоги для Linux и PowerShell соответственно; оба шлют HTTP-запросы.
- **Зачем нужен jq?** — удобно фильтровать и извлекать данные из JSON в командной строке.
- **Какой метод у формы регистрации?** — POST на `/api/Users`.

## Вывод
Развёрнут Docker CE на Ubuntu 20.04 и запущено веб-приложение OWASP Juice Shop в
контейнере. Изучены HTTP-запросы браузера к приложению (поиск, список товаров,
регистрация), сформированы запросы через `curl` и PowerShell, ответы в формате JSON
обработаны утилитой `jq`.
