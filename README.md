# Windows Server 2025 — Backup Lab

Практическая лабораторная работа по изучению трёх типов резервного копирования в Windows Server 2025 Standard Evaluation:

- **Full Backup** — полное резервное копирование.
- **Incremental Backup** — инкрементное резервное копирование.
- **Differential Backup** — дифференциальное резервное копирование.

В проекте представлены пошаговая инструкция, скриншоты выполнения команд и готовые архивы резервных копий, которые можно скачать и использовать для практики восстановления данных.

## Содержание

1. [Окружение](#окружение)
2. [Подготовка системы](#подготовка-системы)
3. [Создание тестовых данных](#создание-тестовых-данных)
4. [Full Backup](#1-full-backup)
5. [Incremental Backup](#2-incremental-backup)
6. [Differential Backup](#3-differential-backup)
7. [Проверка резервных копий](#проверка-резервных-копий)
8. [Восстановление данных из готового архива](#восстановление-данных-из-готового-архива)

---

## Окружение

- ОС: Windows Server 2025 Standard Evaluation
- Виртуальная машина: Oracle VirtualBox
- Исходные данные: `C:\ImportantData`
- Диск для резервных копий: `E:\BackupFiles`
- Файловая система: NTFS

## Подготовка системы

Перед началом необходимо:

1. Установить Windows Server 2025.
2. Подключить дополнительный виртуальный диск.
3. Инициализировать и отформатировать его в NTFS.
4. Установить компонент Windows Server Backup (если используется в рамках дополнительной практики).
5. Создать папку для резервных копий.

Скриншоты подготовки системы находятся в папке `screenshots/`.

## Создание тестовых данных

Создадим исходную структуру:

```powershell
New-Item -ItemType Directory -Path "C:\ImportantData\Documents" -Force
New-Item -ItemType Directory -Path "C:\ImportantData\Projects" -Force
New-Item -ItemType Directory -Path "C:\ImportantData\Images" -Force

Set-Content "C:\ImportantData\Documents\Instructions.txt" "Backup lab instructions"
Set-Content "C:\ImportantData\Documents\Report.txt" "Initial report"
Set-Content "C:\ImportantData\Projects\Project.txt" "Initial project"
```

## 1. Full Backup

Полное резервное копирование сохраняет все файлы из исходной папки.

```powershell
robocopy "C:\ImportantData" "E:\BackupFiles\Full" /E /COPY:DAT /R:2 /W:2 /TEE
```

Проверить содержимое:

```powershell
Get-ChildItem "E:\BackupFiles\Full" -Recurse
```

## 2. Incremental Backup

Инкрементное резервное копирование сохраняет файлы, отмеченные атрибутом Archive.

Сначала сбросим атрибут Archive:

```powershell
attrib -A C:\ImportantData\* /S
```

Изменим несколько файлов:

```powershell
Add-Content "C:\ImportantData\Documents\Instructions.txt" "Incremental backup test"
Add-Content "C:\ImportantData\Projects\Project.txt" "Incremental backup test"
```

Создадим инкрементную копию:

```powershell
robocopy "C:\ImportantData" "E:\BackupFiles\Incremental\Backup_01" /E /M /COPY:DAT /R:2 /W:2 /TEE
```

Ключ `/M` копирует файлы с атрибутом Archive и сбрасывает его после копирования.

## 3. Differential Backup

Дифференциальная копия должна содержать все изменения относительно исходной полной копии.

Сначала создадим базовую копию:

```powershell
robocopy "E:\BackupFiles\Full" "E:\BackupFiles\Differential\Backup_01" /E /COPY:DAT /R:2 /W:2
```

Затем добавим в неё изменённые файлы из источника:

```powershell
robocopy "C:\ImportantData" "E:\BackupFiles\Differential\Backup_01" /E /XO /COPY:DAT /R:2 /W:2 /TEE
```

В этой лабораторной работе используется имитация дифференциального копирования средствами Robocopy, а не штатный механизм Windows Server Backup.

## Проверка резервных копий

Для проверки целостности можно использовать SHA256:

```powershell
Get-FileHash "C:\ImportantData\Documents\Instructions.txt" -Algorithm SHA256
```

Сравните хеши исходных файлов и файлов из резервной копии.

Для просмотра содержимого:

```powershell
Get-ChildItem "E:\BackupFiles" -Recurse
```

## Восстановление данных из готового архива

В репозитории находится ZIP-архив с тремя видами резервных копий:

`backups/Windows-Server-Backups.zip`

### Шаг 1. Скачать архив

Скачайте `Windows-Server-Backups.zip` из папки `backups` этого репозитория.

### Шаг 2. Распаковать архив

Например, в папку `C:\BackupRestore`:

```powershell
Expand-Archive -Path ".\Windows-Server-Backups.zip" -DestinationPath "C:\BackupRestore" -Force
```

### Шаг 3. Выбрать нужную копию

После распаковки будут доступны:

- `Full/` — полная резервная копия.
- `Incremental/Backup_01/` — инкрементная копия.
- `Differential/Backup_01/` — дифференциальная копия.

**Важно:** для восстановления из инкрементной копии необходима исходная полная копия и все последующие инкрементные копии по порядку. В нашем архиве представлена одна инкрементная копия.

Для дифференциального восстановления необходима исходная полная копия и последняя дифференциальная копия.

### Шаг 4. Восстановить файлы

Для восстановления полной копии:

```powershell
robocopy "C:\BackupRestore\Full" "C:\RestoredData\Full" /E /COPY:DAT /R:2 /W:2
```

Для восстановления инкрементной копии:

Сначала восстановите Full, затем примените Incremental:

```powershell
robocopy "C:\BackupRestore\Full" "C:\RestoredData\Incremental" /E /COPY:DAT /R:2 /W:2

robocopy "C:\BackupRestore\Incremental\Backup_01" "C:\RestoredData\Incremental" /E /COPY:DAT /R:2 /W:2
```

Для восстановления дифференциальной копии:

Сначала восстановите Full, затем примените Differential:

```powershell
robocopy "C:\BackupRestore\Full" "C:\RestoredData\Differential" /E /COPY:DAT /R:2 /W:2

robocopy "C:\BackupRestore\Differential\Backup_01" "C:\RestoredData\Differential" /E /COPY:DAT /R:2 /W:2
```

### Шаг 5. Проверить восстановленные данные

```powershell
Get-ChildItem "C:\RestoredData" -Recurse
```

Для дополнительной проверки сравните SHA256 исходных и восстановленных файлов.

---

## Структура проекта

```text
Windows-Server-Backup-Lab/
│
├── README.md
│
├── backups/
│   └── Windows-Server-Backups.zip
│
└── screenshots/
    ├── 01-windows-server-2025.png
    ├── 02-disk-check.png
    ├── 03-disk-creation-and-formatting.png
    ├── 04-role-installation.png
    ├── 05-source-data.png
    ├── 06-full-backup.png
    ├── 07-incremental-backup.png
    ├── 08-differential-backup.png
    ├── 09-sha256-verification.png
    └── 10-full-restoration.png
```

## Заключение

В ходе лабораторной работы были изучены три подхода к резервному копированию: Full, Incremental и Differential.

Также были подготовлены готовые примеры резервных копий, которые можно скачать, распаковать и использовать для самостоятельной практики восстановления данных.

Проект предназначен для обучения и демонстрации базовых принципов резервного копирования в Windows Server 2025.
