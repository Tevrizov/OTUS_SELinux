
Что нужно сделать?

1. Запустить nginx на нестандартном порту 3-мя разными способами:
переключатели setsebool;
добавление нестандартного порта в имеющийся тип;
формирование и установка модуля SELinux.
К сдаче:
README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux.
развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
выяснить причину неработоспособности механизма обновления зоны (см. README);
предложить решение (или решения) для данной проблемы;
выбрать одно из решений для реализации, предварительно обосновав выбор;
реализовать выбранное решение и продемонстрировать его работоспособность.




1. Запустить nginx на нестандартном порту 3-мя разными способами:

Требуется предварительно установленный и работоспособный 
Hashicorp Vagrant  и Oracle VirtualBox 
    tep@tep-HYM-WXX:~$ vagrant -v
    Vagrant 2.4.1

    VirtualBox 7.0.16_Ubuntu r162802

1.1 Cоздаю файл Vagrantfile следующего содержания:
        tep@tep-HYM-WXX:~/OTUS/18.Selinux/Otus 1.1$ cat Vagrantfile 
        MACHINES = {
        :selinux => {
                :box_name => "centos/7",
                :box_version => "2004.01",
                #:provision => "test.sh",       
        },
        }


        Vagrant.configure("2") do |config|
        MACHINES.each do |boxname, boxconfig|
            config.vm.define boxname do |box|
                box.vm.box = boxconfig[:box_name]
                box.vm.box_version = boxconfig[:box_version]


                box.vm.host_name = "selinux"
                box.vm.network "forwarded_port", guest: 4881, host: 4881


                box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "1024"]
                    needsController = false
                end


                box.vm.provision "shell", inline: <<-SHELL
                #install epel-release
                yum install -y epel-release
                #install nginx
                yum install -y nginx
                #change nginx port
                sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
                sed -i 's/listen       80;/listen       4881;/' /etc/nginx/nginx.conf
                #disable SELinux
                #setenforce 0
                #start nginx
                systemctl start nginx
                systemctl status nginx
                #check nginx port
                ss -tlpn | grep 4881
                SHELL
            end
        end
        end
        tep@tep-HYM-WXX:~/OTUS/18.Selinux/Otus 1.1$ 


 1.2 tep@tep-HYM-WXX:~/OTUS/18.Selinux/Otus 1.1$ vagrant up
    Bringing machine 'selinux' up with 'virtualbox' provider...
    ==> selinux: Box 'centos/7' could not be found. Attempting to find and install...
        selinux: Box Provider: virtualbox
        selinux: Box Version: 2004.01
    ==> selinux: Loading metadata for box 'centos/7'   
    ....


    Проверяю:
    tep@tep-HYM-WXX:~/OTUS/18.Selinux/Otus 1.1$ vagrant status
    Current machine states:

    selinux                   running (virtualbox)

    The VM is running. To stop this VM, you can run `vagrant halt` to
    shut it down forcefully, or you can run `vagrant suspend` to simply
    suspend the virtual machine. In either case, to restart it again,
    simply run `vagrant up`.
    tep@tep-HYM-WXX:~/OTUS/18.Selinux/Otus 1.1$ 

1.3 Результатом выполнения команды vagrant up станет созданная виртуальная машина с установленным nginx,
    который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.

    Проверка:
    1.tep@tep-HYM-WXX:~$ netcat -vz localhost 4881
    Connection to localhost (127.0.0.1) 4881 port [tcp/*] succeeded!
    tep@tep-HYM-WXX:~$ 
    2. Но nginx не стартовал 

    Во время развёртывания стенда попытка запустить nginx завершится с ошибкой:
    [vagrant@selinux ~]$ systemctl  status nginx
    ● nginx.service - The nginx HTTP and reverse proxy server
    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    Active: failed (Result: exit-code) since Mon 2024-05-13 12:10:30 UTC; 14min ago
    [vagrant@selinux ~]$ 



1.4 Проверим, что в ОС отключен файервол

    [root@selinux vagrant]# systemctl status firewalld
    ● firewalld.service - firewalld - dynamic firewall daemon
    Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
    Active: inactive (dead)
        Docs: man:firewalld(1)
    [root@selinux vagrant]# 

    Проверяю, что конфигурация nginx настроена без ошибок:
    [root@selinux vagrant]# nginx -t
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    [root@selinux vagrant]# 



1.5 Далее проверим режим работы SELinux:
    [root@selinux vagrant]# getenforce 
    Enforcing
    [root@selinux vagrant]# 

    Данный режим означает, что SELinux будет блокировать запрещенную активность.

    

1.6 СПОСОБ №1 - Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

    Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта:
    [root@selinux vagrant]# cat /var/log/audit/audit.log | grep 4881 | audit2why
    type=AVC msg=audit(1715602230.552:902): avc:  denied  { name_bind } for  pid=3081 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly. 
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
    [root@selinux vagrant]# 
    Утилита audit2why покажет почему трафик блокируется. 
    Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled.

1.7 Включим параметр nis_enabled и перезапустим nginx:

    [root@selinux vagrant]# setsebool -P nis_enabled on
    [root@selinux vagrant]# systemctl restart nginx
    [root@selinux vagrant]# systemctl status nginx
    ● nginx.service - The nginx HTTP and reverse proxy server
    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    Active: active (running) since Mon 2024-05-13 12:38:34 UTC; 7s ago
    Process: 3525 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
    Process: 3522 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 3521 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Main PID: 3527 (nginx)
    CGroup: /system.slice/nginx.service
            ├─3527 nginx: master process /usr/sbin/nginx
            └─3529 nginx: worker process

    May 13 12:38:33 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    May 13 12:38:33 selinux nginx[3522]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    May 13 12:38:34 selinux nginx[3522]: nginx: configuration file /etc/nginx/nginx.conf test is successful
    May 13 12:38:34 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
    [root@selinux vagrant]# 

    Проверка c хостовой машины:
    tep@tep-HYM-WXX:~$  curl -I  127.0.0.1:4881
    HTTP/1.1 200 OK
    Server: nginx/1.20.1
    Date: Mon, 13 May 2024 12:41:10 GMT
    Content-Type: text/html
    Content-Length: 4833
    Last-Modified: Fri, 16 May 2014 15:12:48 GMT
    Connection: keep-alive
    ETag: "53762af0-12e1"
    Accept-Ranges: bytes

    tep@tep-HYM-WXX:~$ 


1.8 Проверить статус параметра можно с помощью команды: 
    root@selinux vagrant]# getsebool -a | grep nis_enabled
    nis_enabled --> on
    [root@selinux vagrant]# 


1.9 Вернём запрет работы nginx на порту 4881 обратно

    [root@selinux vagrant]# setsebool -P nis_enabled off
    [root@selinux vagrant]# getsebool -a | grep nis_enabled
    nis_enabled --> off
    [root@selinux vagrant]# 


-------

1.10 СПОСОБ №2 - Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:

    [root@selinux vagrant]# semanage port -l | grep http
    http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
    http_cache_port_t              udp      3130
    http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
    pegasus_http_port_t            tcp      5988
    pegasus_https_port_t           tcp      5989

1.11 Добавим порт в тип http_port_t: emanage port -a -t http_port_t -p tcp 4881

    [root@selinux vagrant]# semanage port -a -t http_port_t -p tcp 4881
    [root@selinux vagrant]# semanage port -l | grep http
    http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
    http_cache_port_t              udp      3130
    http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
    pegasus_http_port_t            tcp      5988
    pegasus_https_port_t           tcp      5989
    [root@selinux vagrant]# 


1.12 Теперь перезапустим службу nginx и проверим её работу: systemctl restart nginx

    [root@selinux vagrant]# systemctl restart nginx
    [root@selinux vagrant]# semanage port -l | grep http^C
    [root@selinux vagrant]# systemctl restart nginx
    [root@selinux vagrant]# netstat -ltunp | grep 4881
    tcp        0      0 0.0.0.0:4881            0.0.0.0:*               LISTEN      3680/nginx: master  
    tcp6       0      0 :::4881                 :::*                    LISTEN      3680/nginx: master  
    [root@selinux vagrant]# 

    tep@tep-HYM-WXX:~$ curl  -I 127.0.0.1:4881
    HTTP/1.1 200 OK
    Server: nginx/1.20.1
    Date: Mon, 13 May 2024 13:23:05 GMT
    Content-Type: text/html
    Content-Length: 4833
    Last-Modified: Fri, 16 May 2014 15:12:48 GMT
    Connection: keep-alive
    ETag: "53762af0-12e1"
    Accept-Ranges: bytes
    tep@tep-HYM-WXX:~$ 

1.13 Удалить нестандартный порт из имеющегося типа можно с помощью команды: semanage port -d -t http_port_t -p tcp 4881

    [root@selinux vagrant]# semanage port -d -t http_port_t -p tcp 4881 
    [root@selinux vagrant]# semanage port -l | grep  http_port_t
    http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
    pegasus_http_port_t            tcp      5988
    [root@selinux vagrant]# 


    [root@selinux vagrant]# systemctl restart nginx
    Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
    [root@selinux vagrant]# 



1.14 СПОСОБ №3 - Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:

    Попробуем снова запустить nginx
    [root@selinux vagrant]# systemctl start  nginx
    Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
    [root@selinux vagrant]# 

    Nginx не запуститься, так как SELinux продолжает его блокировать.


1.15 Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу
     nginx на нестандартном порту: 

     [root@selinux vagrant]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
     ******************** IMPORTANT ***********************
     To make this policy package active, execute:

    semodule -i nginx.pp

    [root@selinux vagrant]#  

    Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль: semodule -i nginx.pp


1.16 Применяю данный модуль:

    [root@selinux vagrant]# semodule -i nginx.pp
    [root@selinux vagrant]# systemctl start nginx
    [root@selinux vagrant]# systemctl status nginx
    ● nginx.service - The nginx HTTP and reverse proxy server
    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    Active: active (running) since Mon 2024-05-13 13:35:28 UTC; 9s ago
    Process: 3819 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
    Process: 3817 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 3816 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Main PID: 3821 (nginx)
    CGroup: /system.slice/nginx.service
            ├─3821 nginx: master process /usr/sbin/nginx
            └─3823 nginx: worker process

    May 13 13:35:28 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    May 13 13:35:28 selinux nginx[3817]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    May 13 13:35:28 selinux nginx[3817]: nginx: configuration file /etc/nginx/nginx.conf test is successful
    May 13 13:35:28 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
    [root@selinux vagrant]# 

    После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки. 

    Просмотр всех установленных модулей: semodule -l


1.17 Для удаления модуля воспользуемся командой: semodule -r nginx

    [root@selinux vagrant]# semodule -r nginx
    libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
    [root@selinux vagrant]# 



---------------------------------------------------------------------------------------------------------------------------------------------

2.Обеспечение работоспособности приложения при включенном SEL


2.1 Для того, чтобы развернуть стенд потребуется хост, с установленным git и ansible.
    tep@tep-HYM-WXX:~/OTUS/18.Selinux$ git -v
    git version 2.43.0
    
    tep@tep-HYM-WXX:~/OTUS/18.Selinux$ ansible --version
    ansible [core 2.16.3]
    config file = None
    configured module search path = ['/home/tep/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
    ansible python module location = /usr/lib/python3/dist-packages/ansible
    ansible collection location = /home/tep/.ansible/collections:/usr/share/ansible/collections
    executable location = /usr/bin/ansible
    python version = 3.12.3 (main, Apr 10 2024, 05:33:47) [GCC 13.2.0] (/usr/bin/python3)
    jinja version = 3.1.2
    libyaml = True



2.2 Выполняем клонирование репозитория: git clone https://github.com/mbfx/otus-linux-adm.git
    tep@tep-HYM-WXX:~/OTUS/18.Selinux$ git clone https://github.com/mbfx/otus-linux-adm.git
    Клонирование в «otus-linux-adm»...
    remote: Enumerating objects: 558, done.
    remote: Counting objects: 100% (456/456), done.
    remote: Compressing objects: 100% (303/303), done.
    remote: Total 558 (delta 125), reused 396 (delta 74), pack-reused 102
    Получение объектов: 100% (558/558), 1.38 МиБ | 181.00 КиБ/с, готово.
    Определение изменений: 100% (140/140), готово.
    tep@tep-HYM-WXX:~/OTUS/18.Selinux$ 



2.3 перехожу в каталог cd otus-linux-adm/selinux_dns_problems

    tep@tep-HYM-WXX:~/OTUS/18.Selinux/otus-linux-adm/selinux_dns_problems$ vagrant  up
    Bringing machine 'ns01' up with 'virtualbox' provider...
    Bringing machine 'client' up with 'virtualbox' provider...
    ==> ns01: Importing base box 'centos/7'...
    ==> ns01: Matching MAC address for NAT networking...
    ==> ns01: Checking if box 'centos/7' version '2004.01' is up to date...
    ==> ns01: Setting the name of the VM: selinux_dns_problems_ns01_1715359078806_42443
    ==> ns01: Clearing any previously set network interfaces...
    ==> ns01: Preparing network interfaces based on configuration...
        ns01: Adapter 1: nat
        ns01: Adapter 2: intnet
    ==> ns01: Forwarding ports...

    tep@tep-HYM-WXX:~/OTUS/18.Selinux/otus-linux-adm/selinux_dns_problems$ vagrant status
    Current machine states:

    ns01                      running (virtualbox)
    client                    running (virtualbox)

    This environment represents multiple VMs. The VMs are all listed
    above with their current state. For more information about a specific
    VM, run `vagrant status NAME`.
    tep@tep-HYM-WXX:~/OTUS/18.Selinux/otus-linux-adm/selinux_dns_problems$ 


2.4 Подключимся к клиенту: vagrant ssh client

    ep@tep-HYM-WXX:~/OTUS/18.Selinux/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
    Last login: Fri May 10 16:48:32 2024 from 10.0.2.2
    ###############################
    ### Welcome to the DNS lab! ###
    ###############################

    - Use this client to test the enviroment
    - with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

    - nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

    - rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

    ###############################
    ### Enjoy! ####################
    ###############################t
    [vagrant@client ~]$  


2.5 Попробуем внести изменения в зону: nsupdate -k /etc/named.zonetransfer.key

    [vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
    > 
    > ^C[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
    > ^C[vagrant@client ~]$ 
    [vagrant@client ~]$ 
    [vagrant@client ~]$ 
    [vagrant@client ~]$ 
    [vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
    > server 192.168.50.10
    > zone ddns.lab
    > update add www.ddns.lab. 60 A 192.168.50.15
    > send
    update failed: SERVFAIL
    > quit
    [vagrant@client ~]$ 

2.6 Смотрим логи SELinux
    Для этого воспользуемся утилитой audit2why (audit2why — это утилита в Linux, которая определяет из сообщения аудита SELinux причину запрета доступа.)
    [vagrant@client ~]$ sudo -i
    [root@client ~]# 
    [root@client ~]# 
    [root@client ~]# cat /var/log/audit/audit.log | audit2why
    [root@client ~]# 

    подключимся к серверу ns01 и проверим логи SELinux:

    tep@tep-HYM-WXX:~/OTUS/18.Selinux/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
    Last login: Fri May 10 16:42:26 2024 from 10.0.2.2
    [vagrant@ns01 ~]$ sudo -i
    [root@ns01 ~]# cat /var/log/audit/audit.log  | audi
    audispd      audit2allow  audit2why    auditctl     auditd       
    [root@ns01 ~]# cat /var/log/audit/audit.log  | audit2why 
    type=AVC msg=audit(1715360949.197:2000): avc:  denied  { create } for  pid=5392 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" 
    scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

    [root@ns01 ~]# 

    В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.
    Проверяем:

    [root@ns01 ~]# ls -laZ /etc/named
    drw-rwx---. root named system_u:object_r:etc_t:s0       .
    drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
    drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
    -rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
    -rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
    -rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
    -rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
    [root@ns01 ~]# 

    Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. 
    осмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись 
    правильные политики SELinux можно с помощью команды: sudo semanage fcontext -l | grep named

    [root@ns01 ~]# sudo semanage fcontext -l | grep named
    /etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
    /var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
    /etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 


2.7 Изменим тип контекста безопасности для каталога /etc/named: sudo chcon -R -t named_zone_t /etc/named

    [root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
    [root@ns01 ~]# 

2.8 Попробуем снова внести изменения с клиента: 
    [root@client ~]# nsupdate -k /etc/named.zonetransfer.key
    > server 192.168.50.10
    > zone ddns.lab
    > update add www.ddns.lab. 60 A 192.168.50.15
    > send
    > quit
    [root@client ~]# dig www.ddns.lab

    ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37811
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 4096
    ;; QUESTION SECTION:
    ;www.ddns.lab.                  IN      A

    ;; ANSWER SECTION:
    www.ddns.lab.           60      IN      A       192.168.50.15

    ;; AUTHORITY SECTION:
    ddns.lab.               3600    IN      NS      ns01.dns.lab.

    ;; ADDITIONAL SECTION:
    ns01.dns.lab.           3600    IN      A       192.168.50.10

    ;; Query time: 1 msec
    ;; SERVER: 192.168.50.10#53(192.168.50.10)
    ;; WHEN: Fri May 10 17:31:47 UTC 2024
    ;; MSG SIZE  rcvd: 96

    [root@client ~]# 

    изменения применились


2.9 Попробуем перезагрузить хосты и ещё раз сделать запрос с помощью dig: 


    [root@client named]# dig @192.168.50.10 www.ddns.lab

    ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 www.ddns.lab
    ; (1 server found)
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58231
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 4096
    ;; QUESTION SECTION:
    ;www.ddns.lab.                  IN      A

    ;; ANSWER SECTION:
    www.ddns.lab.           60      IN      A       192.168.50.15

    ;; AUTHORITY SECTION:
    ddns.lab.               3600    IN      NS      ns01.dns.lab.

    ;; ADDITIONAL SECTION:
    ns01.dns.lab.           3600    IN      A       192.168.50.10

    ;; Query time: 0 msec
    ;; SERVER: 192.168.50.10#53(192.168.50.10)
    ;; WHEN: Fri May 10 18:00:16 UTC 2024
    ;; MSG SIZE  rcvd: 96

    [root@client named]# 


    Всё правильно. После перезагрузки настройки сохранились. 
    Для того, чтобы вернуть правила обратно, можно ввести команду:
    
    restorecon -v -R /etc/named


------------------------------------------------------------------------------------------------------------