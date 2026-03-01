## 2.0 Основная часть дипломной работы. Этапы выполнения.
### 2.1 Заполнение конфигурационного файла terraform `main.tf` для выполнения задач дипломной работы.
Ссылки на файлы terraform: 

[main.tf](./files/main.tf)

[meta.yaml](/meta.yaml)

[.terraformrc](/terraformrc)

#### По условиям задачи необходимо развернуть через terraform следующий ресурcы:

##### Сайт. Веб-сервера. Nginx.
- Создать две ВМ в разных зонах, установить на них сервер nginx.
- Создать Target Group, включить в неё две созданные ВМ.
- Создать Backend Group, настроить backends на target group, ранее созданную. Настроить healthcheck на корень (/) и порт 80, протокол HTTP.
- Создать HTTP router. Путь указать — /, backend group — созданную ранее.
- Создать Application load balancer для распределения трафика на веб-сервера, созданные ранее. Указать HTTP router, созданный ранее, задать listener тип auto, порт 80.
```terraform
####################
## Nginx-web-1 #####
####################
resource "yandex_compute_instance" "nginx-web-1" {
  name  = "nginx-web-1"
  hostname = "nginx-web-1"
  zone  = yandex_vpc_subnet.a-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8k22q9mlprdarduk0o"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.a-subnet-diplom.id}"
    ipv4 = true
    ip_address = "192.168.10.3"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.nginx-web-security.id, yandex_vpc_security_group.filebeat-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}

####################
## Nginx-web-2 #####
####################
resource "yandex_compute_instance" "nginx-web-2" {
  name  = "nginx-web-2"
  hostname = "nginx-web-2"
  zone  = yandex_vpc_subnet.b-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8k22q9mlprdarduk0o"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.b-subnet-diplom.id}"
    ipv4 = true
    ip_address = "192.168.20.3"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.nginx-web-security.id, yandex_vpc_security_group.filebeat-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}

#################################################################################################################
## Target group ##### https://cloud.yandex.ru/ru/docs/application-load-balancer/operations/target-group-create ##
#################################################################################################################
resource "yandex_alb_target_group" "nginx-target-group"{
  name = "nginx-target-group"

  target {
    subnet_id = "${yandex_vpc_subnet.a-subnet-diplom.id}"
    ip_address   = "${yandex_compute_instance.nginx-web-1.network_interface.0.ip_address}"
  } 

  target {
    subnet_id = "${yandex_vpc_subnet.b-subnet-diplom.id}"
    ip_address   = "${yandex_compute_instance.nginx-web-2.network_interface.0.ip_address}"
  }  
}

###################################################################################################################
## Backend group ##### https://cloud.yandex.ru/ru/docs/application-load-balancer/operations/backend-group-create ##
###################################################################################################################
resource "yandex_alb_backend_group" "nginx-backend-group" {
  name = "nginx-backend-group"
  session_affinity {
    connection {
      source_ip = false
    }
  }

  http_backend {
    name                   = "http-backend"
    weight                 = 1
    port                   = 80
    target_group_ids       = ["${yandex_alb_target_group.nginx-target-group.id}"]
    load_balancing_config {
      panic_threshold      = 90
    }    
    healthcheck {
      timeout              = "10s"
      interval             = "2s"
      healthy_threshold    = 10
      unhealthy_threshold  = 15 
      http_healthcheck {
        path               = "/"
      }
    }
  }
}

###############################################################################################################
## HTTP router ##### https://cloud.yandex.ru/ru/docs/application-load-balancer/operations/http-router-create ##
###############################################################################################################
resource "yandex_alb_http_router" "nginx-tf-router" {
  name   = "nginx-tf-router"
  labels = {
    tf-label    = "tf-label-value"
    empty-label = ""
  }
}

resource "yandex_alb_virtual_host" "nginx-virtual-host" {
  name           = "nginx-virtual-host"
  http_router_id = yandex_alb_http_router.nginx-tf-router.id
  route {
    name = "nginx-route"
    http_route {
      http_route_action {
        backend_group_id = yandex_alb_backend_group.nginx-backend-group.id
        timeout          = "60s"
      }
    }
  }
}    

############################################################################################################################################
## Application load balancer ##### https://cloud.yandex.com/ru/docs/application-load-balancer/operations/application-load-balancer-create ##
############################################################################################################################################
resource "yandex_alb_load_balancer" "nginx-balancer" {
  name        = "nginx-balancer"
  network_id  = "${yandex_vpc_network.network-diplom.id}"

  allocation_policy {
    location {
      zone_id   = "${yandex_vpc_subnet.d-subnet-diplom.zone}"
      subnet_id = yandex_vpc_subnet.d-subnet-diplom.id
    }
  }

  listener {
    name = "nginx-listener"
    endpoint {
      address {
        external_ipv4_address {
        }
      }
      ports = [ 80 ]
    }
    http {
      handler {
        http_router_id = yandex_alb_http_router.nginx-tf-router.id
      }
    }
  }
}
```

##### Мониторинг. Zabbix. Zabbix-agent.
- Создать ВМ, развернуть на ней Zabbix. На каждую ВМ установить Zabbix Agent, настроить агенты на отправление метрик в Zabbix.
```terraform
###############
## Zabbix #####
###############
resource "yandex_compute_instance" "zabbix" {
  name  = "zabbix"
  hostname = "zabbix"
  zone  = yandex_vpc_subnet.d-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8k22q9mlprdarduk0o"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.d-subnet-diplom.id}"
    nat = true
    ipv4 = true
    ip_address = "192.168.30.4"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.zabbix-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}
```

##### Логи. Elasticsearch. Kibana. Filebeat.
- Cоздать ВМ, развернуть на ней Elasticsearch. Установить Filebeat в ВМ к веб-серверам, настроить на отправку access.log, error.log nginx в Elasticsearch.
- Создать ВМ, развернуть на ней Kibana, сконфигурировать соединение с Elasticsearch.
```terraform
######################
## Elasticsearch #####
######################
resource "yandex_compute_instance" "elasticsearch" {
  name  = "elasticsearch"
  hostname = "elasticsearch"
  zone  = yandex_vpc_subnet.a-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 4
  }

  boot_disk {
    initialize_params {
      image_id = "fd8k22q9mlprdarduk0o"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.a-subnet-diplom.id}"
    ipv4 = true
    ip_address = "192.168.10.4"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.elasticsearch-security.id, yandex_vpc_security_group.kibana-security.id, yandex_vpc_security_group.filebeat-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}

###############
## Kibana #####
###############
resource "yandex_compute_instance" "kibana" {
  name  = "kibana"
  hostname = "kibana"
  zone  = yandex_vpc_subnet.d-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8k22q9mlprdarduk0o"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.d-subnet-diplom.id}"
    nat = true
    ipv4 = true
    ip_address = "192.168.30.5"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.elasticsearch-security.id, yandex_vpc_security_group.kibana-security.id, yandex_vpc_security_group.filebeat-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}
```