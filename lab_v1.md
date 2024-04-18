# Лабораторная работа 

## Задание 

> Пусть есть сервер с адресом 11.03.02.09, 
использующий порты SSH-22 (управление серверов) 
и HTTP-80 (общедоступное API) и HTTP-9000 (метрики), 
все подключения к которому проходит через МСЭ. 
Администратор подключается с 11.10.138.15 для управления сервером по SSH.

![Топология сети](./screenshoots/Scheme.drawio.png)

Необходимо сделать:
-	Чтобы злоумышленник не мог использовать сканер безопасности, читать метрики и управлять сервером, необходимо для всех пользователей, кроме администратора, был доступен толь порт 80;
- Для избежания флуд атаки, необходимо ограничить количество пакетов, посылаемых на сервер;
- Запретить передачи пакетов с содержимым `block-payload`.

## Подготовка к лабораторной работы
### Установка
В лабораторной работе будет использовать технологии [`docker`](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) и [`containerlab`](https://containerlab.dev/install/#install-script).
Для их установки необходимо ввести:
```bash
$ sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
$ curl -sL https://get.containerlab.dev | sudo bash
```
### Создание конфигурационного файла
Конфигурационный файл должен быть назван `lab.clab.yml`:

[Файл конфигурации](./lab.clab.yml)
```yaml
name: lab

topology:
  nodes:
    # сервер, который нету защищать
    victim:
      image: wbitt/network-multitool:alpine-extra
      kind: linux
      exec:
        - ip addr add 2.2.1.21/24 dev eth1
        - ip r a 2.2.2.0/24 via 2.2.1.32
        - ip r a 2.2.3.0/24 via 2.2.1.32
        - ip r a 2.2.4.0/24 via 2.2.1.32
    # МСЭ, которым мы управляем
    firewall:
      # image: wbitt/network-multitool:alpine-extra
      image: vrnetlab/vr-routeros:6.47.9
      kind: linux
      exec:
        - ip addr add 2.2.2.31/24 dev eth1
        - ip addr add 2.2.1.32/24 dev eth2
        - ip r a 2.2.4.0/24 via 2.2.2.41
        - ip r a 2.2.3.0/24 via 2.2.2.41
        - ip r a 2.2.1.21 dev eth2

    # реализация внешней сети
    internet:
      image: wbitt/network-multitool:alpine-extra
      kind: linux
      exec:
        - ip addr add 2.2.2.41/24 dev eth1
        - ip addr add 2.2.3.42/24 dev eth2
        - ip addr add 2.2.4.43/24 dev eth3
        - ip r add 2.2.1.0/24 via 2.2.2.31
        # - ip r a 2.2.1.0/24 dev eth1
        # - ip r a 2.2.2.31/32 via 2.2.2.41
    # потенциально атакующий
    atacker:
      image: wbitt/network-multitool:alpine-extra
      kind: linux
      exec:
        - ip addr add 2.2.4.51/24 dev eth1
        - ip r a 2.2.3.0/24 via 2.2.4.43
        - ip r a 2.2.2.0/24 via 2.2.4.43
        - ip r a 2.2.1.0/24 via 2.2.4.43
    # администратор
    administator:
      image: wbitt/network-multitool:alpine-extra
      kind: linux
      exec:
        - ip addr add 2.2.3.61/24 dev eth1
        - ip r a 2.2.4.0/24 via 2.2.3.42
        - ip r a 2.2.2.0/24 via 2.2.3.42
        - ip r a 2.2.1.0/24 via 2.2.3.42
  links:
    - endpoints: ["firewall:eth1", "internet:eth1"]
    - endpoints: ["firewall:eth2", "victim:eth1"]
    - endpoints: ["administator:eth1", "internet:eth2"]
    - endpoints: ["atacker:eth1", "internet:eth3"]
```

### Запуск стенда
Необходимо вызвать команду:
```bash
$ containerlab deploy
```

В результате должна появиться следующая таблица:

| # |         Name          | Container ID |              Image               | Kind  |  State  |  IPv4 Address  |     IPv6 Address     |
|---|-----------------------|--------------|----------------------------------|-------|---------|----------------|----------------------|
| 1 | clab-lab-administator | e9b2914da2ea | ilyakharev/graduate-lab:admin    | linux | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
| 2 | clab-lab-atacker      | 9605fcf17d66 | ilyakharev/graduate-lab:atacker  | linux | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
| 3 | clab-lab-firewall     | 329d7c0f6719 | ilyakharev/graduate-lab:firewall | linux | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
| 4 | clab-lab-internet     | 9489cdf7b4fd | ilyakharev/graduate-lab:internet | linux | running | 172.20.20.5/24 | 2001:172:20:20::5/64 |
| 5 | clab-lab-victim       | 082bb94acb88 | ilyakharev/graduate-lab:victim   | linux | running | 172.20.20.6/24 | 2001:172:20:20::6/64 |

Управление межсетевым экраном осуществляется по адресу `172.20.20.5`.

## Выполнение лабораторной работы

Для добавления правил в МСЭ необходимо нажать на `IP -> Firewall`.
![image](https://github.com/ilyakharev/graduate-work/assets/38153753/dd745cde-fd93-480f-bd47-abb9bb9972dd)
Затем нажать `Add New`, ввести необходимые поля и нажать `OK`.

1. пользователей, кроме администратора, был доступен толь порт 80
```bash
iptables  -A FORWARD  -s 2.2.3.61/24  -j ACCEPT
iptables  -A FORWARD  --protocol tcp --dst 2.2.1.21/24 --dport 80 --jump ACCEPT
iptables  -A FORWARD  -d 2.2.1.21/24  -j DROP
```
![image](https://github.com/ilyakharev/graduate-work/assets/38153753/5112bc7b-768b-4562-89e3-2844805d7c81)
![image](https://github.com/ilyakharev/graduate-work/assets/38153753/dbff8945-9195-4c51-9a71-9474b89ccc10)
![image](https://github.com/ilyakharev/graduate-work/assets/38153753/85d19c8f-77aa-499c-9340-aab42c4cff86)

А так же добавить

![image](https://github.com/ilyakharev/graduate-work/assets/38153753/85897893-4863-41e1-8856-5a633fb0a380)

2. Для избежания флуд атаки, необходимо ограничить количество пакетов, посылаемых на сервер;
```bash
iptables -A INPUT -m limit --limit 3/sec -j ACCEPT
```
![image](https://github.com/ilyakharev/graduate-work/assets/38153753/f5a23b2d-6933-4cf5-ad5e-4d256645fe8b)

3. Запретить пакеты с определенным payload
```bash
iptables  -A FORWARD  -m search --algo kmp --hex-string "block-payload"  -j DROP
```
![image](https://github.com/ilyakharev/graduate-work/assets/38153753/f827ee6f-a96d-4fd8-aa12-5397b94e0afa)

4. Запретить фрагментацию пакетов

В результате должен появиться такой список

![image](https://github.com/ilyakharev/graduate-work/assets/38153753/cdf73331-6662-4fd3-8184-d41f18bfa4a6)

Перейти на сайт и начать тест

![image](https://github.com/ilyakharev/graduate-work/assets/38153753/6ee89b01-5a94-4dac-be6a-a806a7577bb4)

В результате должно быть

![image](https://github.com/ilyakharev/graduate-work/assets/38153753/5ad5609a-3730-40ad-bee1-6a054e4ca1e1)
