# 1. Настройка ВМ
1.	Настройка сети (выдаем ip-адреса)
2.	Создание image (для этого пишем в машине ls /var/tmp/Debian-12-nocloud-amd64-20231210-1591.qcow2)
3.  В VR настраиваем сеть, пишем в /etc/systemd/resolv.conf nameserver 8.8.8.8
4.  На основной машине (на которой установлена OpenNebula) пишем правила для iptables:
```
    iptables -t nat -A POSTROUTING -s ip-адрес сетевого интерфейса/24 -o ens3 -j MASQUERADE
    iptables -t nat -A POSTROUTING -s 172.16.100.0/24 – ens192 -j MASQUERADE
    apt install iptables-persistent
    iptables-save > /etc/iptables/rules.v4
```
5.  На VM3 пишем iptables -t nat -A POSTROUTING -o ens3(или другой интерфейс, не altname) -j MASQUERADE
6.  Обновляем пакеты и устанавливаем утилиты на каждую машину
7.  Настройка
<hr>

# 2. Настройка ssh
1.  Устанавливаем на все машины openssh-server, заходим в /etc/ssh/sshd_config и убираем # с PermitRootLogin yes и PasswordAuthentication yes
2.  Перезапускаем ssh
3.  Создаем пользователей user (useradd user, usermod -aG sudo user) и добавляем пользователя в visudo (user ALL=(ALL) NOPASSWD: ALL)
4.  Пишем ssh-copy-id user@ip-адрес машины
<hr>

# 3. Доп штуки
1.  Если не хватает памяти пишем команды growpart /dev/sda 1 и resize2fs /dev/sda1
2.  Запуск докера 
```
    docker build -t myapp . 
    docker run -d -p 5000:5000 –name app myapp
```
3. Настройка Postgresql
<hr>
```
    sudo -u postgres psql -c “CREATE USER myuser WITH PASSWORD ‘student’;”
    sudo -u postgres psql -с “CREATE DATABASE mydb OWNER myuser;”
```
<hr>
    Для доступа к БД необходимо зайти в /etc/postgresql/15/main/postgresql.conf и написать listen_address = “*” и так же зайти в /etc/postgresql/15/main/pg_hba.conf и в самом низу под IPV4 local connections: написать host all myuser 172.18.100.0/24 md5 и потом все перезагрузить systemctl restart postgresql

4. HAproxy
Пишем в /etc/haproxy/haproxy.cfg:
```
frontend httpin
    bind *:80
    default_backend webservers

backend webservers
    balance     roundrobin
    option      httpchk GET /status
    server      app1 ip-адрес первой ВМ:5000 check
    server      app2 ip-адрес второй ВМ:5000 check backup
```
Проверка конфига - haproxy -c -f /etc/haproxy/haproxy.cfg

5. Создание скрипта для отказоустойчивости
Заходим в пользователя root и пишем скрипт /usr/local/bin/autoscale.sh, после пишем 

```
Crontab -e
*/2 * * * * /usr/local/bin/autoscale.sh >> /var/log/autoscale.log 2>&1
```

6. Locust
```
apt install -y python3 python3-pip python3-venv
pip3 install locust

locust -f locust.py --headless -u 100 -r 20 --host=http://ip-адрес_ВМ

```
Для доступа к веб-интерфейсу locust пишем в браузере http://ip-адрес_ВМ:8089

7. Использование Debian образа для создания image в самой OpenNebula, который был заранее загружен в директорию
Используй команду 
```
find / -name "*.qcow2"
```
Для поиска файла one_auth используй эту же команду:
```
find / -name "one_auth"
```

8. Включаем форвардинг:
```
echo 1 > /proc/sys/net/ipv4/ip_forward
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.d/98-forward.conf
```
