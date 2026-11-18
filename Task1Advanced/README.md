# Модульная инфраструктура для нескольких сред

Универсальный модуль Terraform для создания виртуальных машины в Yandex Cloud с тремя подготовленными окружениями: dev, stage, prod. Окружения вызывают один и тот же модуль но с разными параметрами через аргумент `-var-file`

## Структура модуля и окружения

```
/TaskAdvanced1/
  ├── modules/
  │   │
  │   └── vm/
  │       ├── main.tf        <- ВМ, подключаемый диск данных, сетевой интерфейс
  │       ├── variables.tf   <- входные параметры модуля
  │       └── outputs.tf     <- выходные данные
  │
  └── envs/
      ├── dev/               <- main.tf, variables.tf, outputs.tf, dev.tfvars
      ├── stage/             <- main.tf, variables.tf, outputs.tf, dev.tfvars
      └── prod/              <- main.tf, variables.tf, outputs.tf, dev.tfvars
```

Все три окружения имеют одинаковые подготовленные файлы `main.tf`, `variables.tf`, `outputs.tf`, в которых меняется только `*.tfvars`. Модули не содержать захардкоженных значений окружений, всё значения находятся через переменные

- [modules/vm/main.tf](modules/vm/main.tf)
- [modules/vm/variables.tf](modules/vm/variables.tf)
- [modules/vm/outputs.tf](modules/vm/outputs.tf)
- [envs/dev/dev.tfvars](envs/dev/dev.tfvars)
- [envs/stage/stage.tfvars](envs/stage/stage.tfvars)
- [envs/prod/prod.tfvars](envs/prod/prod.tfvars)

## Задача модуля

`modules/vm` создать следующее:

- `yandex_compute_instance` - виртуальную машину с заданным числом ядер, объёмом RAM и долей vCPU, загрузочным диском из образа, сетевым интерфейсом в указанной подсети и SSH-ключом в метаданных  
- `yandex_compute_disk` - отдельный подключаемый (вторичный) диск данных, который присоединяется к ВМ через блок `secondary_disk` с `auto_delete = false`  

## Параметры модуля

**Обязательные (по интерфейсу задания):**

- `image_id` - ID образа загрузочного диска  
- `subnet_id` - ID подсети для сетевого интерфейса  
- `cores` - количество ядер (vCPU)  
- `ssh_key` - содержимое публичного SSH-ключа  
- `name` - имя ВМ и производных ресурсов  
- `memory` - объём RAM в ГБ  
- `attached_disk_size` - размер подключаемого диска в ГБ (тип - `attached_disk_type`)  

**Необязательные (со значениями по умолчанию):**

- `labels` - метки ресурсов, по умолчанию пустая карта  
- `zone` - зона доступности, по умолчанию `ru-central1-a`  
- `core_fraction` - гарантированная доля vCPU (100, 50, 20, 5), по умолчанию 100  
- `nat` - выдавать ли публичный IP, по умолчанию `false`  
- `attached_disk_type` - тип подключаемого диска, по умолчанию `network-ssd`  
- `preemptible` - создавать прерываемую ВМ, по умолчанию `false`  
- `ssh_user` - пользователь ОС для SSH-ключа, по умолчанию `ubuntu`  
- `boot_disk_size` - размер загрузочного диска в ГБ, по умолчанию 20  
- `boot_disk_type` - тип загрузочного диска, по умолчанию `network-hdd`  

## Выходы модуля

- `fqdn` - FQDN виртуальной машины  
- `internal_ip` - внутренний IP-адрес  
- `instance_id` - ID виртуальной машины  
- `instance_name` - имя виртуальной машины  
- `external_ip` - публичный IP-адрес (`null`, если `nat = false`)  
- `boot_disk_id` - ID загрузочного диска  
- `attached_disk_id` - ID подключаемого диска данных  

Те же значения проброшены наружу в `outputs.tf` каждого окружения

## Различия окружений

- dev: 2 ядра, 2 ГБ RAM, `core_fraction = 20`, прерываемая ВМ, публичный IP, диски `network-hdd` по 20 ГБ. Минимальная стоимость  
- stage: 4 ядра, 8 ГБ RAM, `core_fraction = 100`, без публичного IP, диски `network-ssd` (30 и 100 ГБ), зона `ru-central1-b`  
- prod: 8 ядер, 32 ГБ RAM, `core_fraction = 100`, без публичного IP, диски `network-ssd` (50 и 500 ГБ)  

## Запуск

Требуется Terraform >= 1.5 и настроенная аутентификация в Yandex Cloud (переменная окружения `YC_TOKEN` или `yc` CLI / сервисный аккаунт).  
В файлах `*.tfvars` значения `cloud_id`, `folder_id`, `subnet_id`, `image_id` и `ssh_key` - примеры, их нужно заменить на реальные для своей инфраструктуры.  

### dev:

```
cd envs/dev
terraform init
terraform plan  -var-file=dev.tfvars
terraform apply -var-file=dev.tfvars
```

### stage:

```
cd envs/stage
terraform init
terraform apply -var-file=stage.tfvars
```

### prod:

```
cd envs/prod
terraform init
terraform apply -var-file=prod.tfvars
```

## Проверка

- `terraform fmt -recursive -check` - код отформатирован  
- `terraform validate` в `modules/vm` и в каждом окружении `envs/*` - конфигурация валидна  

