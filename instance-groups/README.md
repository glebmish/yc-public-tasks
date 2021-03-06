# Работа с Группами виртуальных машин


### Подготовка окружения
* Зайдите в консоль облака https://console.cloud.yandex.ru и создайте себе Каталог (folder)
* В терминале рабочей станции инициируйте `yc init`
* Выберите созданный вами folder

### Создание сети для группы виртуальных машин

Создадим сеть и подсети в трех зонах доступности для работы группы виртуальных машин

```
yc vpc network create --name yc-auto-network

zones=(a b c)

for i in ${!zones[@]}; do
  echo "Creating subnet yc-auto-subnet-$i"
  yc vpc subnet create --name yc-auto-subnet-$i \
  --zone ru-central1-${zones[$i]} \
  --range 192.168.$i.0/24 \
  --network-name yc-auto-network
done
```



### Создание группы виртуальных машин
Создаем группу инстансов, которая интегрируется с целевой группой

Создадим файл конфигурации для группы VM
```
cat > nginx.yaml <<EOF
name: nginx
instance_template:
  platform_id: standard-v1
  resources_spec:
    memory: 4294967296
    cores: 4
    core_fraction: 100
  metadata:
    user-data: |-
      #cloud-config
      apt:
        preserve_sources_list: true
      package_update: true
      runcmd:
        - [ sh, -c, "until ping -c1 www.centos.org &>/dev/null; do :; done" ]
        - [ sh, -c, "until ping -c1 www.docker.com &>/dev/null; do :; done" ]
        - [ sh, -c, "until ping -c1 www.google.com &>/dev/null; do :; done" ]
        - [ sh, -c, "apt install -y nginx" ]
        - [ systemctl, daemon-reload ]
        - [ systemctl, enable,  nginx.service ]
        - [ systemctl, start, --no-block, nginx.service ]
        - [ sh, -c, "echo \$(hostname | cut -d '.' -f 1 ) > /var/www/html/index.html" ]
  boot_disk_spec:
    mode: READ_WRITE
    disk_spec:
      type_id: network-nvme
      size: 21474836480
      image_id: fd8o145rut84nlussc8a
  network_interface_specs:
  - network_id: $(yc vpc network get --name yc-auto-network --format=json | jq .id | tr -d '"')
    primary_v4_address_spec: { one_to_one_nat_spec: { ip_version: IPV4 }}
scale_policy:
  fixed_scale:
    size: 3
deploy_policy:
  max_unavailable: 1
  starting_duration: 0s
allocation_policy:
  zones:
  - zone_id: ru-central1-a
  - zone_id: ru-central1-b
  - zone_id: ru-central1-c
load_balancer_spec:
    target_group_spec:
        name: nginx

EOF
```

```
cat nginx.yaml
yc compute instance-group create --file=nginx.yaml
rm -rf nginx.yaml
```

Зайдите в консоль облака в ваш фолдер, далее выберите Сompute и перейдите в раздел "Группы виртуальных машин" и посмотрите на созданную вами группу



### Подключим балансировщик у группе виртуальных машин

Создадим балансировщик, к которому подключена таргет группа
* Зайдите в созданный вами каталог
* Выберите Load balancer
* Нажмите создать
* Добавьте порт 80 для обработчика
* Добавьте созданную целевую группу с проверкой доступности работающей по HTTP на 80 порту, делающей запросы на url '/'

### Проверим балансировщик

Зайдите в консоль облака в ваш фолдер, далее выберите Сompute и перейдите в раздел "Группы виртуальных машин" и посмотрите на созданную вами группу.
Далее зайдите в раздел load balancer , посмотрите на созданный балансирощик, нажмите на него - посмотрети что все ВМ в статусе Healthy

Команды ниже сделают 30 HTTP запросов на адрес балансировщика. Веб сервера за балансирощиком ответят своими именами хостов.

```
EXTERNAL_IP=$(yc load-balancer network-load-balancer get nginx --format=json | jq .listeners[0].address | tr -d '"')

for i in {1..30}
do
  curl --silent $EXTERNAL_IP
done
```

### Удалим инфраструктуру

Удалим Балансировщик и целевую группу
```
yc load-balancer network-load-balancer delete --name nginx

yc compute instance-group delete --name nginx
```

Удалим Виртуальные машины

```
for i in ${!zones[@]}; do
  echo "Deleting subnet yc-auto-subnet-$i"
  yc vpc subnet delete --name yc-auto-subnet-$i
done

echo "Deleting network yc-auto-network"
yc vpc network delete --name yc-auto-network

```
