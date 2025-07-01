1. Настройка ВМ
2. 172.18.0.1/24(внешняя сеть)
3. 192.168.1.1/24(внутренняя сеть)
4. на opennebula /etc/sysctl.conf net.ipv4.ip_forward=1
5. sysctl -p
6. iptables -t nat -A POSTROUTING -j MASQUERADE -o ens3 -s 192.168.1.0/24 iptables-save > /root/rules or /etc/iptables/rules.v4
6. crontab -e @reboot /sbin/iptables-restore < /root/rules
7. В VR настраиваем сеть, пишем в /etc/systemd/resolv.conf nameserver 8.8.8.8
8. на всех машинах с дебиан rm /run/systemd/network/10-netplan-all-en.network и apt purge netplan.io -y
9. /etc/systemd/network/wire.network во внешней сети прописываем gateway и dns, во внутренней:
10. [Router]
11. Destination=0.0.0.0/0
12. Gateway=192.168.1.1/24
13. Metric=0
14. на vr прописываем iptables -t nat -A POSTROUTING -s 172.18.0.0/24 -o ens4(или другой) -j MASQUERADE
15. iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o ens4(или другой) -j MASQUERADE
16. На VM3 пишем iptables -t nat -A POSTROUTING -o ens3(или другой интерфейс, не altname) -j MASQUERADE
17. Обновляем пакеты и устанавливаем утилиты на каждую машину
18. Настройка
2. Настройка ssh
Устанавливаем на все машины openssh-server, заходим в /etc/ssh/sshd_config и убираем # с PermitRootLogin yes и PasswordAuthentication yes  
Перезапускаем ssh  
Создаем пользователей user (useradd user, usermod -aG sudo user) и добавляем пользователя в visudo (user ALL=(ALL) NOPASSWD: ALL)  
Пишем ssh-copy-id user@ip-адрес машины  
3. db  
Если не хватает памяти пишем команды growpart /dev/sda 1 и resize2fs /dev/sda1  
3. Настройка Postgresql  
<hr>
apt install postgresql postgresql-contrib su - postgres create user -P --interactive devops n y y psql CREATE DETEBASE test OWNER devops; |

<hr>
    Для доступа к БД необходимо зайти в /etc/postgresql/15/main/postgresql.conf и написать listen_address = “*” и так же зайти в /etc/postgresql/15/main/pg_hba.conf и в самом низу под IPV4 local connections: заменить последнюю на host all all ip webapp и потом все перезагрузить systemctl restart postgresql
4.webapp
apt-get update && apt-get install docker.io
nano web-app.py
#!/usr/bin/env python3

from flask import Flask, request
import tomllib
import psycopg2
import time
import math

app = Flask(__name__)

with open('config.toml', 'rb') as f:
  data = tomllib.load(f)

def _check_postgres():
  try:
    conn = psycopg2.connect(dbname=data['DBNAME'], user=data['DBUSER'], host=data['DBHOST'], password=data['DBPASSWORD'], connect_timeout=1)
    conn.close()
    return True
  except:
    return False

@app.route('/')
def hello():
  return 'hello world'

@app.route('/status')
def status():
  return 'app is ok' if _check_postgres() else 'fail connect to database'

if __name__ == '__main__':
  app.run(host=data['HOST'])

nano config.toml
HOST = '0.0.0.0'
DBNAME = 'test'
DBUSER = 'devops'
DBHOST = 'ip postgre' 
DBPASSWORD = 'devops'

nano Dockerfile
FROM python:3.11-alpine
RUN pip install flask
RUN pip install psycopg2-binary
COPY web-app.py /web-app.py
COPY config.toml /config.toml
CMD ["python3", "web-app.py"]

docker run -p 5001:5000 --name=webapp1 --restart=always -d webapp
docker run -p 5002:5000 --name=webapp2 --restart=always -d webapp

5.балансировщик nginx(router)
apt-get update && apt-get install nginx
nano /etc/nginx/sites-avalible/default
upstream app {
        server ip_webapp:5001;
        server ip_webapp:5002;
}
server {
        listen 80 default_server;
        server_name _;
        location / {
                proxy_pass http://app;
                allow сетка адресов публичных(locust)/24;
                deny all;
        }
}
server {
        listen 5000 default_server;
        server_name _;
        location / {
                proxy_pass http://app;
                allow сетка адресов приватных/24;
                deny all;
        }
}
с vr прописать curl http://приватный ip_vr:5000 




6. Locust
#apt install -y python3 python3-pip python3-venv #pip3 install locust

#locust -f locust.py --headless -u 100 -r 20 --host=http://ip-адрес_ВМ

apt-get install pipx pipx install locust pipx ensurepath nano /root/locustfile.py from locust import HttpUser, task, between

class WebsiteTestUser(HttpUser): wait_time = between(5, 90)

def on_start(self):
    pass
def on_stop(self):
    pass
@task()
def hello_world(self):
    self.client.get("/")
    self.client.get("/status")
locust --headless --users 100 --spawn-rate 10 -H http://публичный ip_vr --run-time 10s

Для доступа к веб-интерфейсу locust пишем в браузере http://ip-адрес_locust:8089

7. Использование Debian образа для создания image в самой OpenNebula, который был заранее загружен в директорию
Используй команду 
find / -name "*.qcow2"

Для поиска файла one_auth используй эту же команду:
find / -name "one_auth"


8. Включаем форвардинг:
echo 1 > /proc/sys/net/ipv4/ip_forward echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.d/98-forward.conf


4. HAproxy
Пишем в /etc/haproxy/haproxy.cfg:
frontend httpin bind *:80 default_backend webservers

backend webservers balance roundrobin option httpchk GET /status server app1 ip-адрес первой ВМ:5000 check server app2 ip-адрес второй ВМ:5000 check backup

Проверка конфига - haproxy -c -f /etc/haproxy/haproxy.cfg

5. Создание скрипта для отказоустойчивости
Заходим в пользователя root и пишем скрипт /usr/local/bin/autoscale.sh, после пишем 

Crontab -e */2 * * * * /usr/local/bin/autoscale.sh >> /var/log/autoscale.log 2>&1
```
