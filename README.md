# Домашнее задание к занятию "`Отказоустойчивость в облаке`" - `Голуб Сергей`


### Инструкция по выполнению домашнего задания

   1. Сделайте `fork` данного репозитория к себе в Github и переименуйте его по названию или номеру занятия, например, https://github.com/имя-вашего-репозитория/git-hw или  https://github.com/имя-вашего-репозитория/7-1-ansible-hw).
   2. Выполните клонирование данного репозитория к себе на ПК с помощью команды `git clone`.
   3. Выполните домашнее задание и заполните у себя локально этот файл README.md:
      - впишите вверху название занятия и вашу фамилию и имя
      - в каждом задании добавьте решение в требуемом виде (текст/код/скриншоты/ссылка)
      - для корректного добавления скриншотов воспользуйтесь [инструкцией "Как вставить скриншот в шаблон с решением](https://github.com/netology-code/sys-pattern-homework/blob/main/screen-instruction.md)
      - при оформлении используйте возможности языка разметки md (коротко об этом можно посмотреть в [инструкции  по MarkDown](https://github.com/netology-code/sys-pattern-homework/blob/main/md-instruction.md))
   4. После завершения работы над домашним заданием сделайте коммит (`git commit -m "comment"`) и отправьте его на Github (`git push origin`);
   5. В личном кабинете прикрепите и отправьте ссылку на решение в виде md-файла в вашем Github.
   6. Любые вопросы по выполнению заданий спрашивайте в разделе “Вопросы по заданию” в личном кабинете.
   
Желаем успехов в выполнении домашнего задания!
   
### Дополнительные материалы, которые могут быть полезны для выполнения задания

1. [Руководство по оформлению Markdown файлов](https://gist.github.com/Jekins/2bf2d0638163f1294637#Code)

---

### Задание 1

`Возьмите за основу решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке».`

1. `Теперь вместо одной виртуальной машины сделайте terraform playbook, который:`
`- создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;`
`- создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;`
`- создаст сетевой балансировщик нагрузки, который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.`
`Рекомендуем изучить документацию сетевого балансировщика нагрузки для того, чтобы было понятно, что вы сделали.`
2. `Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.`
3. `Перейдите в веб-консоль Yandex Cloud и убедитесь, что:`
`- созданный балансировщик находится в статусе Active,`
`- обе виртуальные машины в целевой группе находятся в состоянии healthy.`
4. `Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.`
`В качестве результата пришлите:`
`1. Terraform Playbook.`
`2. Скриншот статуса балансировщика и целевой группы.`
`3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.`

---

### Решение 1

*1. Terraform Playbook.*
main.tf:
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

# --- Переменные ---
variable "yc_token" {
  type      = string
  sensitive = true
}

variable "yc_cloud_id" {
  type = string
}

variable "yc_folder_id" {
  type = string
}

# --- Провайдер ---
provider "yandex" {
  token     = var.yc_token
  cloud_id  = var.yc_cloud_id
  folder_id = var.yc_folder_id
  zone      = "ru-central1-b"
}

# --- Сеть ---
resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network-1.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

# --- Виртуальные машины ---
resource "yandex_compute_instance" "vm" {
  count = 2
  name  = "vm${count.index}"

  resources {
    core_fraction = 20
    cores         = 2
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8a67rb91j689dqp60h"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }

  metadata = {
    user-data = file("./meta.yaml")
  }
}

# --- Таргет-группа ---
resource "yandex_lb_target_group" "target-1" {
  name = "target-1"

  dynamic "target" {
    for_each = yandex_compute_instance.vm
    content {
      subnet_id = yandex_vpc_subnet.subnet-1.id
      address   = target.value.network_interface[0].ip_address
    }
  }
}

# --- Сетевой балансировщик ---
resource "yandex_lb_network_load_balancer" "lb-1" {
  name = "lb1"

  listener {
    name = "listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.target-1.id
    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

# --- Outputs ---
output "internal_ip_address_vm-0" {
  value = yandex_compute_instance.vm[0].network_interface[0].ip_address
}
output "external_ip_address_vm-0" {
  value = yandex_compute_instance.vm[0].network_interface[0].nat_ip_address
}
output "internal_ip_address_vm-1" {
  value = yandex_compute_instance.vm[1].network_interface[0].ip_address
}
output "external_ip_address_vm-1" {
  value = yandex_compute_instance.vm[1].network_interface[0].nat_ip_address
}
```


`2. Скриншот статуса балансировщика и целевой группы.`

![изображение](https://github.com/sGolubn/8-05-hw/blob/main/1.jpg) 

![изображение](https://github.com/sGolubn/8-05-hw/blob/main/2.jpg) 

![изображение](https://github.com/sGolubn/8-05-hw/blob/main/3.jpg) 

3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.

![изображение](https://github.com/sGolubn/8-05-hw/blob/main/4.jpg) 

```

