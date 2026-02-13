## 1.0 Установка и подготовка Terraform и Ansible.
### 1.1 Установка и подготовка Terraform.
Скачиваю и распаковываю стабильную версию с сайта: https://hashicorp-releases.yandexcloud.net/terraform/
```bash
wget https://hashicorp-releases.yandexcloud.net/terraform/1.13.3/terraform_1.13.3_linux_amd64.zip
unzip terraform_1.13.3_linux_amd64.zip
chmod 744 terraform
sudo cp terraform /usr/local/bin/
sudo terraform -version
```
![](screen/screen_1_1.png)

Создаю файл `.terraformrc` и добавляю блок с источником, из которого будет устанавливаться провайдер.
```bash
nano ~/.terraformrc
```
```terraform
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```
![](screen/screen_1_2.png)

Для файла с метаданными, `meta.yaml`, необходим публичный SSH-ключ для доступа к ВМ. Для Yandex Cloud рекомендуется использовать алгоритм Ed25519: сгенерированные по нему ключи — самые безопасные. Ссылка: https://cloud.yandex.ru/ru/docs/glossary/ssh-keygen
```bash
ssh-keygen -t ed25519
```
![](screen/screen_1_3.png)

Создаю файл `meta.yaml` с данными пользователя на создаваемые ВМ.
```bash
nano ~/meta.yaml
```
```terraform
#cloud-config
 users:
  - name: serg
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-ed25519
```
![](screen/screen_1_4.png)
Создаю `playbook Terraform` c блоком провайдера.
```bash
nano ~/main.tf
```
```terraform
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}
```
![](screen/screen_1_5.png)
Инициализирую провайдера.
```bash
terraform-diplom init
```
![](screen/screen_1_6.png)

#### Terraform готов к использованию. Продолжение в основной части.

---
### 1.2 Установка и подготовка Ansible.
Устанавливаю Ansible и проверяю версию.
```bash
sudo apt install ansible
ansible --version
```
![](screen/screen_1_7.png)

Создаю полностью прокомментированный пример `ansible.cfg` и заменяю содержимое файла на необходимые опции. Файл прикреплю в основной части.
```bash
ansible-config init --disabled > ansible.cfg
nano ~/ansible.cfg
```
![](screen/screen_1_8.png)
Создаю файл `hosts` и добавляю в него начальные данные. Файл прикреплю в основной части.

```bash
nano ~/hosts
```
#### Ansible готов к использованию. Продолжение в основной части.

[Ссылка на основную часть дипломной работы.](https://github.com/README.md)
---