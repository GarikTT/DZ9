https://codeberg.org/54210395/otus_linux_hw/src/branch/master/hw_12/readme.md

Занятие 9 - Инициализация системы. Systemd.
Цели занятия - 
	1. писать сценарии автозагрузки демонов;
	2. обращаться с systemctl и journalctl.
Результат - написать свой systemd unit. Необходимо написать сервис, который будет раз в 30 секунд мониторить лог на
предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig.
Потратил два дня решение и тестирование. Довел все действия до автоматизма.

1  Запускаем - script --timing=time_homework_log homework.log
2. Создаем Vagrantfile с Centos/8 - vagrant init Centos/8
4. vagrant up
5. vagrant ssh
6. sudo -i
7. cat > /etc/sysconfig/watchlog
# Создадим файл /etc/sysconfig в котором
# пропишем слово за коим мы будем мониторить.
WORD="ALERT"
LOG=/var/log/watchlog.log
8. Создаем /var/log/watchlog.log - cat > /var/log/watchlog.log:
	Почему меня все не любят?
	Почему меня все бесят?
	Почему я не могу говорить по английски свободно?
	Почему у меня в школе была тройка по русскому?
	ALERT
	ARTLET
	Зец олл
9. Создадим скрипт - cat > /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
	logger "$DATE: Вот фраза в примере домашнего задания непонятна, я напишу от себя - Привед! Кагдила?!!"
else
	exit 0
fi
exit 0
10. Добавим права на запуск файла - chmod +x /opt/watchlog.sh
11. Создаем юнит для сервиса - cat > /lib/systemd/system/watchlog.service:
[Unit]
Description=My watchlog service
[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
12. Cодадим unit для таймера - cat > /lib/systemd/system/watchlog.timer: // Ради спортивного интереса, отойдем от методички и сделаем 36 секунд
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
#OnUnitActiveSec=30
OnCalendar=*:*:0,36
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
13. После изменения перегружаем всех демонов - 
	systemctl daemon-reload
14. Запускаем сервис - 
	systemctl start watchlog.timer
	systemctl start watchlog.service
15. Проверяем каждые 36 секунд лог - cat /var/log/messages или 
	tail -f /var/log/messages
16. Все заработало.
17. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. 
	Имя сервиса должно также называться.
18. Устанавливаем spawn-fcgi и необходимые для него пакеты:
   	sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
	sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
	yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
	раскомментируем строки с переменными в /etc/sysconfig/spawn-fcgi
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.                                                                                               
#                                                                                                                                                                            
# See spawn-fcgi(1) for all possible options.                                                                                                                                
#                                                                                                                                                                            
# Example :                                                                                                                                                                  
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
19. Создадим юнит-файл: cat > /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
20. Убеждаемся что все успешно работает:
	systemctl start spawn-fcgi
	systemctl status spawn-fcgi
21. Для запуска нескольких экземпляров сервиса будем исполþзовать шаблон в
конфигурации файла окружения (/usr/lib/systemd/system/httpd.service ):
	cat > /etc/systemd/system/httpd.service
[Unit]
Description=The Apache HTTP Server
Wants=httpd-init.service

After=network.target remote-fs.target nss-lookup.target httpd-init.service

Documentation=man:httpd.service(8)

[Service]
Type=notify
Environment=LANG=C
EnvironmentFile=/etc/sysconfig/httpd-%i
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
# Send SIGWINCH for graceful stop
KillSignal=SIGWINCH
KillMode=mixed
PrivateTmp=true

[Install]
WantedBy=multi-user.target
22. cat > /etc/sysconfig/httpd-first
# /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf
23. cat > /etc/sysconfig/httpd-second
# /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf
24. Также создадим 2 файла конфигов httpd - first.conf и second.conf для этого скопируем стандартный файл файл конфигурации -
	cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/first.conf
	cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/second.conf
25. В файле /etc/httpd/conf/first.conf меняем 
Listen 80 
на 
Listen 8080
ServerName localhost
PidFile /var/run/httpd-first.pid

	sed -i 's*Listen 80*Listen 8080\nServerName localhost\nPidFile /var/run/httpd-first.pid*g' /etc/httpd/conf/first.conf
25. В файле /etc/httpd/conf/second.conf меняем 
Listen 80 
на 
Listen 80
ServerName localhost
PidFile /var/run/httpd-second.pid

	sed -i 's*Listen 80*Listen 80\nServerName localhost\nPidFile /var/run/httpd-second.pid*g' /etc/httpd/conf/second.conf
#25. # Переименуем юнит-файл в формат, для запуска нескольких экземпляров:
	# mv /usr/lib/systemd/system/httpd.service /usr/lib/systemd/system/httpd-@.service
26. Перечитаем настройки systemd:
	systemctl daemon-reload
27. Запустим экземпляры веб-сервера:
	systemctl start httpd@first
	systemctl start httpd@second
28. Проверяем -
	systemctl status httpd@first
	systemctl status httpd@second
	ss -tnulp | grep httpd
29. Все работает!

30. Получаем файлы time_homework_log и homework.log с описанными действиями - 
	exit
	exit
	exit
	scriptreplay --timing=time_homework_log homework.log -d 20

