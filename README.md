# Split DNS
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/split-dns-hw23.git`
В текущей директории появится папка с именем репозитория. В данном случае split-dns-hw23. Ознакомимся с содержимым:
```
cd split-dns-hw23
ls -l
README.md
provisioning
Vagrantfile
```
Здесь:
- provisioning - папка с плэйбуками
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ:
```
vagrant up
```

## Добавить еще один сервер client2
Для этого потребуется добавить запись Vagrantfile и playbook.yml:
```
# Vagrantfile
  config.vm.define "client2" do |client2|
    client2.vm.network "private_network", ip: "192.168.50.16", virtualbox__intnet: "dns"
    client2.vm.hostname = "client2"
  end
# playbook.yml
- hosts: client1,client2
  become: yes
  tasks:
  - name: copy resolv.conf to the client1,client2
    copy: src=client-resolv.conf dest=/etc/resolv.conf owner=root group=root mode=0644
  - name: copy rndc conf file
    copy: src=rndc.conf dest=/home/vagrant/rndc.conf owner=vagrant group=vagrant mode=0644
  - name: copy motd to the client1,client2
    copy: src=client-motd dest=/etc/motd owner=root group=root mode=0644
```
## Завести в зоне dns.lab имена web1 и web2
Для этого потребуется добавить запись named.dns.lab:
```
# named.dns.lab
;Web
web1            IN      A       192.168.50.15
web2            IN      A       192.168.50.16
```
# Завести еще одну зону newdns.lab и добавить в ней запись
Для этого потребуется добавить создать файл named.newdns.lab:
```
# named.newdns.lab
$TTL 3600
$ORIGIN newdns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201007 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11

;WWW
www             IN      A       192.168.50.15
www             IN      A       192.168.50.16
```
## Настроить split-dns
### клиент1 - видит обе зоны, но в зоне dns.lab только web1
Для этого нужно создать ключ, далее добавить acl запись, запись view для каждого клиента:
```
# master-named.conf (для ns01)
key "client1" {
        algorithm hmac-sha512;
        secret "4oq3IGhf30vOd6inoWMEfqiKokZJ9Amx7n/t9ZagtZe8uiZyusASZqhPmgFYbiXAiJ6YbryhSM6mCWfAk2Maxw==";
};
key "client2" {
        algorithm hmac-sha512;
        secret "jY2QCsiPSY1G9ydS70IapS0QPBzimrBuQ6UZ5Xl/Yt9g3V+V5IPKfdr+8q9Gjf8eo/9O0/yIoI2b0mRFp8lsqQ==";
};

acl client1 { !key client2; key client1; 192.168.50.15; };
acl client2 { !key client1; key client2; 192.168.50.16; };

// Настройка первого view 
view "client1" {
    // Кому из клиентов разрешено подключаться, нужно указать имя access-листа
    match-clients { client1; };

    // Описание зоны dns.lab для client
    zone "dns.lab" {
        // Тип сервера — мастер
        type master;
        // Добавляем ссылку на файл зоны, который создали в прошлом пункте
        file "/etc/named/named.dns.lab.client1";
        // Адрес хостов, которым будет отправлена информация об изменении зоны
        also-notify { 192.168.50.11 key client1; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "/etc/named/named.newdns.lab";
        also-notify { 192.168.50.11 key client1; };
    };
};

// Описание view для client2
view "client2" {
    match-clients { client2; };

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "/etc/named/named.dns.lab";
        also-notify { 192.168.50.11 key client2; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "/etc/named/named.dns.lab.rev";
        also-notify { 192.168.50.11 key client2; };
    };
};
# slave-named.conf (для ns02)
key "client1" {
        algorithm hmac-sha512;
        secret "4oq3IGhf30vOd6inoWMEfqiKokZJ9Amx7n/t9ZagtZe8uiZyusASZqhPmgFYbiXAiJ6YbryhSM6mCWfAk2Maxw==";
};
key "client2" {
        algorithm hmac-sha512;
        secret "jY2QCsiPSY1G9ydS70IapS0QPBzimrBuQ6UZ5Xl/Yt9g3V+V5IPKfdr+8q9Gjf8eo/9O0/yIoI2b0mRFp8lsqQ==";
};

acl client1 { !key client2; key client1; 192.168.50.15; };
acl client2 { !key client1; key client2; 192.168.50.16; };

view "client1" {
    match-clients { client1; };
    allow-query { any; };

    // dns.lab zone
    zone "dns.lab" {
        // Тип сервера — slave
        type slave;
        // Будет забирать информацию с сервера 192.168.50.10
        masters { 192.168.50.10 key client1; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type slave;
        masters { 192.168.50.10 key client1; };
    };
};

view "client2" {
    match-clients { client2; };

    // dns.lab zone
    zone "dns.lab" {
        type slave;
        masters { 192.168.50.10 key client2; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type slave;
        masters { 192.168.50.10 key client2; };
    };
};
```