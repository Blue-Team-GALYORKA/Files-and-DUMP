# Практическая работа. Атака на Windows, дамп .evtx и анализ логов (Hayabusa)

## Цель работы
- Развернуть виртуальную машину с Windows и настроить аудит безопасности.
- Провести на неё набор атак (post-exploitation), оставляющих следы в журналах событий.
- Выгрузить журналы событий в формате `.evtx`.
- Проанализировать журналы Sigma-движком **Hayabusa** и получить HTML-отчёт с обнаружениями.

## Задача
Получить итоговый HTML-отчёт Hayabusa с обнаружениями (как в примере): General Overview,
Results Summary, обнаружения по уровням опасности (high/medium/low), список хостов,
разбивка по тактикам MITRE ATT&CK.

> ⚠️ Все атаки проводятся **только** на собственной изолированной лабораторной ВМ.
> 📷 СКРИНШОТ: каждый ключевой шаг.

---

## Схема стенда
```
[ Kali Linux — атакующий ]  ──локальная сеть (Host-Only)──►  [ Windows 10/11 или Server — цель ]
                                                              + Sysmon + включённый аудит
                                               ▼
                                   экспорт .evtx  ──►  анализ Hayabusa  ──►  HTML-отчёт
```
Сеть между ВМ — **Host-Only / Internal** (изолированная, без выхода в интернет), чтобы
атаки не ушли наружу.

---

## Часть 1. Развёртывание Windows-ВМ

1. В VirtualBox/VMware создать ВМ с Windows 10/11 (или Windows Server 2019/2022).
2. Сетевой адаптер — **Host-Only Adapter** (та же сеть, что у Kali).
3. Сделать **snapshot** чистой системы (чтобы откатиться после атаки).
4. Узнать IP машины:
   ```cmd
   ipconfig
   ```
📷 СКРИНШОТ установленной Windows и её IP.

---

## Часть 2. Настройка аудита и логирования

Без включённого аудита в журналах не будет нужных событий, и Hayabusa ничего не найдёт.
Выполнять в **PowerShell от администратора**.

### 2.1. Включить расширенный аудит (локальная политика)
```cmd
:: аудит управления учётными записями (создание/изменение пользователей, смена паролей)
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
:: аудит членства в группах
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
:: аудит входов и отказов входа (brute-force)
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Account Lockout" /failure:enable
:: аудит изменения системного времени
auditpol /set /subcategory:"Security State Change" /success:enable /failure:enable
:: проверить
auditpol /get /category:*
```

### 2.2. Включить логирование PowerShell (Script Block + Module)
Через `gpedit.msc` → Computer Configuration → Administrative Templates → Windows Components
→ Windows PowerShell → включить **Turn on Module Logging** и **Turn on PowerShell Script
Block Logging**. Или через реестр:
```powershell
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name EnableScriptBlockLogging -Value 1
```

### 2.3. Установить Sysmon (детальная телеметрия процессов/сети)
```powershell
# скачать Sysmon из Sysinternals и готовый конфиг (напр. SwiftOnSecurity/sysmon-modular)
# установка с конфигом:
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
# проверка, что служба работает:
Get-Service Sysmon64
```
Журнал Sysmon появится в `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`.
📷 СКРИНШОТ включённого аудита и работающего Sysmon.

---

## Часть 3. Проведение атаки

Ниже — набор действий атакующего, каждое из которых порождает конкретное событие в
журнале. Именно эти события затем «подсветит» Hayabusa (сопоставление с обнаружениями из
примера приведено в таблице).

### 3.1. Brute-force по SMB/RDP с Kali (отказы входа)
С атакующей машины Kali подобрать пароль (генерирует события **4625** — отказ входа):
```bash
# по SMB
crackmapexec smb 192.168.56.20 -u users.txt -p passwords.txt
# или hydra по RDP
hydra -L users.txt -P passwords.txt rdp://192.168.56.20
```
Порождает: *Logon Failure (Wrong Password)* и *Logon Failure (User Does Not Exist)*.

### 3.2. Пост-эксплуатация на Windows (от имени администратора)
Выполнить на цели в cmd/PowerShell — имитация действий закрепившегося атакующего:
```cmd
:: создать пользователя и добавить в локальные администраторы  (4720, 4732)
net user hacker P@ssw0rd123 /add
net localgroup administrators hacker /add

:: сбросить пароль существующему пользователю  (4724)
net user victim NewP@ss456 /add
net user victim ResetByAdmin!

:: добавить в глобальную группу безопасности (на контроллере домена)  (4728)
net group "Domain Admins" hacker /add /domain

:: перечисление сохранённых учётных данных (Credential Manager)
vaultcmd /listcreds:"Windows Credentials"
cmdkey /list

:: установка подозрительной службы с бинарём во временной папке  (7045, 4697)
sc create EvilSvc binPath= "C:\Users\Public\evil.exe" start= auto
sc start EvilSvc

:: изменение системного времени  (4616)
w32tm /config /manualpeerlist:1.1.1.1
Set-Date -Date (Get-Date).AddHours(-3)

:: аварийная остановка службы / некорректное завершение  (7034, 7023)
sc stop EvilSvc

:: внеплановое завершение работы  (1074 / 6008)
shutdown /r /t 0 /c "test"
```
> Для чистоты эксперимента часть команд можно заменить на **Atomic Red Team** — готовые
> тесты техник MITRE ATT&CK, которые безопасно генерируют телеметрию.

### Соответствие действий и обнаружений (как в примере)
| Действие атакующего | Event ID | Обнаружение в отчёте |
|---------------------|----------|----------------------|
| `net localgroup administrators hacker /add` | 4732 | User Added To Local Admin Grp |
| `sc create EvilSvc binPath=...public\evil.exe` | 7045 | Suspicious Service Path / Rare Service Installations |
| `net user victim ResetByAdmin!` | 4724 | Password Reset By Admin |
| `vaultcmd /listcreds`, `cmdkey /list` | 4798/Sysmon | Credential Manager Enumerated / Accessed |
| `net group "Domain Admins" ... /add` | 4728 | Member Added to Security-Enabled Global Group |
| brute-force (несущ. юзер) | 4625 | Logon Failure (User Does Not Exist) |
| brute-force (неверный пароль) | 4625 | Logon Failure (Wrong Password) |
| `net user hacker ... /add` | 4720 | Local User Account Created |
| `w32tm` / `Set-Date` | 4616 | Unauthorized System Time Modification |
| `sc stop` / сбой службы | 7034/7023 | Service Crashed / Terminated With Error |
📷 СКРИНШОТ выполнения команд атаки.

---

## Часть 4. Дамп (экспорт) журналов в .evtx

Файлы журналов физически лежат в `C:\Windows\System32\winevt\Logs\*.evtx`. Но правильнее
экспортировать их утилитой `wevtutil` (сохранит целостность и метаданные).

### 4.1. Экспорт отдельных журналов (`wevtutil epl`)
Выполнять в cmd/PowerShell от администратора:
```cmd
mkdir C:\logs

:: основные журналы
wevtutil epl Security      C:\logs\Security.evtx
wevtutil epl System        C:\logs\System.evtx
wevtutil epl Application    C:\logs\Application.evtx

:: журнал Sysmon
wevtutil epl "Microsoft-Windows-Sysmon/Operational" C:\logs\Sysmon.evtx

:: журнал PowerShell
wevtutil epl "Microsoft-Windows-PowerShell/Operational" C:\logs\PowerShell.evtx
```

### 4.2. Экспорт ВСЕХ журналов разом (PowerShell)
```powershell
mkdir C:\logs -Force
wevtutil el | ForEach-Object {
    $name = $_ -replace '[\\/]', '-'
    try { wevtutil epl "$_" "C:\logs\$name.evtx" 2>$null } catch {}
}
```

### 4.3. Просто скопировать файлы журналов
```powershell
Copy-Item C:\Windows\System32\winevt\Logs\*.evtx C:\logs\ -Force
```

### 4.4. Передать .evtx на машину с анализатором
Скопировать папку `C:\logs` на машину с Hayabusa (через общую папку ВМ, SMB или `scp`).
Hayabusa работает и под Windows, поэтому анализировать можно прямо на целевой ВМ.
📷 СКРИНШОТ папки с выгруженными .evtx.

---

## Часть 5. Анализ в Hayabusa

Hayabusa — быстрый Sigma-движок на Rust; именно его HTML-отчёт показан в примере задания.

### 5.1. Установка
Скачать релиз с https://github.com/Yamato-Security/hayabusa (есть сборки под Windows и
Linux), распаковать. Обновить правила:
```bash
./hayabusa update-rules
```
(на Windows — `hayabusa.exe update-rules`).

### 5.2. Запуск (команда как в примере)
Пример из задания: `hayabusa dfir-timeline -d <папка логов> -p verbose -o timeline.csv -H report.html -w`
```bash
# построение таймлайна с HTML-отчётом
./hayabusa csv-timeline -d /home/kali/logs -o timeline.csv -H report.html -p verbose
```
- `-d` — папка с `.evtx`;
- `-o timeline.csv` — CSV-таймлайн всех обнаружений;
- `-H report.html` — HTML-отчёт (именно он на картинке примера);
- `-p verbose` — подробный профиль полей;
- `-w` — обновить правила перед запуском.

Для одного файла — `-f`:
```bash
./hayabusa csv-timeline -f /home/kali/logs/Security.evtx -o timeline.csv -H report.html
```
📷 СКРИНШОТ запуска Hayabusa в терминале (прогресс-бар и итоговая сводка).

### 5.3. Что содержит HTML-отчёт (разделы с примера)
- **General Overview** — командная строка запуска, время старта, размер и число
  проанализированных событий, статистика правил (stable/test/noisy и т. д.).
- **Results Summary** — всего и уникальных обнаружений по уровням
  (critical / high / medium / low / informational), с процентами.
- **Dates with most total detections** — даты с наибольшим числом сработок по уровням.
- **Computers with most … detections** — хосты с наибольшим числом обнаружений
  (в примере — `lab`, `WIN-9P7VNATCK9`).
- **All high / medium / low alerts** — конкретные обнаружения с указанием авторов правил:
  *User Added To Local Admin Grp*, *Suspicious Service Path*, *Password Reset By Admin*,
  *Credential Manager Enumerated*, *Rare Service Installations*, *Logon Failure (User Does
  Not Exist)*, *Logon Failure (Wrong Password)*, *Local User Account Created*,
  *Unauthorized System Time Modification*, *Windows Service Terminated With Error* и др.
- **MITRE ATT&CK Tactics** — разбивка обнаружений по тактикам: Persistence, Privilege
  Escalation, Credential Access, Impact — по каждому хосту.
📷 СКРИНШОТ HTML-отчёта Hayabusa (должен совпадать по структуре с примером преподавателя).

### 5.4. Открыть отчёт
Открыть `report.html` в браузере. Для интерактивного просмотра таймлайна CSV можно
загрузить `timeline.csv` в Timeline Explorer или Excel.

---

## Контрольные вопросы
- **Что такое .evtx?** — бинарный формат файлов журналов событий Windows.
- **Где лежат журналы?** — `C:\Windows\System32\winevt\Logs\`.
- **Чем экспортировать журнал?** — `wevtutil epl <журнал> <файл.evtx>`.
- **Зачем включать аудит и Sysmon перед атакой?** — иначе нужные события не попадут в журнал и анализ ничего не найдёт.
- **Что делает Hayabusa?** — применяет Sigma-правила к .evtx и выдаёт обнаружения атак с уровнями и тегами MITRE ATT&CK.
- **Что такое Sigma?** — универсальный формат правил детектирования для логов (SIEM-независимый).
- **Какой Event ID у добавления в локальные администраторы?** — 4732.
- **Какой Event ID у отказа входа?** — 4625.
- **Что показывает раздел MITRE ATT&CK в отчёте?** — к каким тактикам/техникам относятся обнаруженные события.

## Вывод
Развёрнута Windows-ВМ с включённым аудитом безопасности и Sysmon. Проведён набор атак
(brute-force входа, создание пользователя и добавление в администраторы, сброс пароля,
установка подозрительной службы, перечисление Credential Manager, изменение времени),
каждая из которых оставила следы в журналах. Журналы выгружены в `.evtx` утилитой
`wevtutil` и проанализированы Sigma-движком Hayabusa. Получен итоговый HTML-отчёт с
обнаружениями по уровням опасности и разбивкой по тактикам MITRE ATT&CK, совпадающий по
структуре с примером.
