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