**Секции systemd**
Description	Краткое описание юнита.
Documentation	Список ссылок на документацию.
Before, After	Порядок запуска юнитов.
Requires	Если этот сервис активируется, перечисленные здесь юниты тоже будут активированы. Если один из перечисленных юнитов останавливается или падает, этот сервис тоже будет остановлен.
Wants	Устанавливает более слабые зависимости, чем Requires. Если один из перечисленных юнитов не может успешно запуститься, это не повлияет на запуск данного сервиса. Это рекомендуемый способ установления зависимостей.
Conflicts	Если установлено что данный сервис конфликтует с другим юнитом, то запуск последнего остановит этот сервис и наоборот.
Alias	Дополнительные имена сервиса разделенные пробелами. Большинство команд в systemctl, за исключением systemctl enable, могут использовать альтернативные имена сервисов.
RequiredBy, WantedBy	Данный сервис будет запущен при запуске перечисленных сервисов. Для более подробной информации смотрите описание опций Wants и Requires в секции [Unit].
Also	Определяет список юнитов, которые также будут активированы или дезактивированы вместе с данным сервисом при выполнении команд systemctl enable или systemctl disable.
Type	Настраивает тип запуска процесса. Один из:
simple (по умолчанию) — запускает сервис мгновенно. Предполагается, что основной процесс сервиса задан в ExecStart.
forking — считает сервис запущенным после того, как родительский процесс создает процесс-потомка, а сам завершится.
oneshot — аналогичен типу simple, но предполагается, что процесс должен завершиться до того, как systemd начнет отслеживать состояния юнитов (удобно для скриптов, которые выполняют разовую работу и завершаются). Возможно вы также захотите использовать RemainAfterExit=yes, чтобы systemd продолжал считать сервис активным и после завершения процесса.
dbus — аналогичен типу simple, но считает сервис запущенным после того, как основной процесс получает имя на шине D-Bus.
notify — аналогичен типу simple, но считает сервис запущенным после того, как он отправляет systemd специальный сигнал.
idle — аналогичен типу simple, но фактический запуск исполняемого файла сервиса откладывается, пока не будут выполнены все задачи.
ExecStart	Команды вместе с аргументами, которые будут выполнены при старте сервиса. Опция Type=oneshot позволяет указывать несколько команд, которые будут выполняться последовательно. Опции ExecStartPre и ExecStartPost могут задавать дополнительные команды, которые будут выполнены до или после ExecStart.
ExecStop	Команды, которые будут выполнены для остановки сервиса запущенного с помощью ExecStart.
ExecReload	Команды, которые будут выполнены чтобы сообщить сервису о необходимости перечитать конфигурационные файлы.
Restart	Если эта опция активирована, сервис будет перезапущен если процесс прекращен или достигнут timeout, за исключением случая нормальной остановки сервиса с помощью команды systemctl stop
RemainAfterExit	Если установлена в значение True, сервис будет считаться запущенным даже если сам процесс завершен. Полезен с Type=oneshot. Значение по умолчанию False.



**Systemd запускает сервисы описанные в его конфигурации.**
Конфигурация состоит из множества файлов, которые по-модному называют юнитами.

Все эти юниты разложены в трех каталогах:

/usr/lib/systemd/system/ – юниты из установленных пакетов RPM — всякие nginx, apache, mysql и прочее
/run/systemd/system/ — юниты, созданные в рантайме — тоже, наверное, нужная штука
/etc/systemd/system/ — юниты, созданные системным администратором — а вот сюда мы и положим свой юнит.

Юнит представляет из себя текстовый файл с форматом, похожим на файлы .ini Microsoft Windows.

[Название секции в квадратных скобках]
имя_переменной = значение


**Для создания простейшего юнита надо описать три секции: [Unit], [Service], [Install]**

В секции Unit описываем, что это за юнит:
Названия переменных достаточно говорящие:

Описание юнита:
Description=MyUnit

Далее следует блок переменных, которые влияют на порядок загрузки сервисов:

Запускать юнит после какого-либо сервиса или группы сервисов (например network.target):
After=syslog.target
After=network.target
After=nginx.service
After=mysql.service

Для запуска сервиса необходим запущенный сервис mysql:
Requires=mysql.service

Для запуска сервиса желателен запущенный сервис redis:
Wants=redis.service

В итоге переменная Wants получается чисто описательной.
Если сервис есть в Requires, но нет в After, то наш сервис будет запущен параллельно с требуемым сервисом, а не после успешной загрузки требуемого сервиса

В секции Service указываем какими командами и под каким пользователем надо запускать сервис:

Тип сервиса:
Type=simple
(по умолчанию): systemd предполагает, что служба будет запущена незамедлительно. Процесс при этом не должен разветвляться. Не используйте этот тип, если другие службы зависят от очередности при запуске данной службы.

Type=forking
systemd предполагает, что служба запускается однократно и процесс разветвляется с завершением родительского процесса. Данный тип используется для запуска классических демонов.

Также следует определить PIDFile=, чтобы systemd могла отслеживать основной процесс:
PIDFile=/work/www/myunit/shared/tmp/pids/service.pid

Рабочий каталог, он делается текущим перед запуском стартап команд:
WorkingDirectory=/work/www/myunit/current

Пользователь и группа, под которым надо стартовать сервис:
User=myunit
Group=myunit


Переменные окружения:
Environment=RACK_ENV=production

Запрет на убийство сервиса вследствие нехватки памяти и срабатывания механизма OOM:
-1000 полный запрет (такой у sshd стоит), -100 понизим вероятность.
OOMScoreAdjust=-100

Команды на старт/стоп и релоад сервиса

ExecStart=/usr/local/bin/bundle exec service -C /work/www/myunit/shared/config/service.rb --daemon
ExecStop=/usr/local/bin/bundle exec service -S /work/www/myunit/shared/tmp/pids/service.state stop
ExecReload=/usr/local/bin/bundle exec service -S /work/www/myunit/shared/tmp/pids/service.state restart

Тут есть тонкость — systemd настаивает, чтобы команда указывала на конкретный исполняемый файл. Надо указывать полный путь.

Таймаут в секундах, сколько ждать system отработки старт/стоп команд.
TimeoutSec=300


Попросим systemd автоматически рестартовать наш сервис, если он вдруг перестанет работать.
Контроль ведется по наличию процесса из PID файла
Restart=always


В секции [Install] опишем, в каком уровне запуска должен стартовать сервис

Уровень запуска:
WantedBy=multi-user.target

multi-user.target или runlevel3.target соответствует нашему привычному runlevel=3 «Многопользовательский режим без графики. Пользователи, как правило, входят в систему при помощи множества консолей или через сеть»

Вот и готов простейший стартап скрипт, он же unit для systemd:
myunit.service

[Unit]
Description=MyUnit
After=syslog.target
After=network.target
After=nginx.service
After=mysql.service
Requires=mysql.service
Wants=redis.service

[Service]
Type=forking
PIDFile=/work/www/myunit/shared/tmp/pids/service.pid
WorkingDirectory=/work/www/myunit/current

User=myunit
Group=myunit

Environment=RACK_ENV=production

OOMScoreAdjust=-1000

ExecStart=/usr/local/bin/bundle exec service -C /work/www/myunit/shared/config/service.rb --daemon
ExecStop=/usr/local/bin/bundle exec service -S /work/www/myunit/shared/tmp/pids/service.state stop
ExecReload=/usr/local/bin/bundle exec service -S /work/www/myunit/shared/tmp/pids/service.state restart
TimeoutSec=300

[Install]
WantedBy=multi-user.target 

Кладем этот файл в каталог /etc/systemd/system/

Смотрим его статус systemctl status myunit

myunit.service - MyUnit
   Loaded: loaded (/etc/systemd/system/myunit.service; disabled)
   Active: inactive (dead)

Видим, что он disabled — разрешаем его
systemctl enable myunit
systemctl -l status myunit

Если нет никаких ошибок в юните — то вывод будет вот такой:

myunit.service - MyUnit
   Loaded: loaded (/etc/systemd/system/myunit.service; enabled)
   Active: inactive (dead)


Запускаем сервис
systemctl start myunit

Смотрим красивый статус:
systemctl -l status myunit

Если есть ошибки — читаем вывод в статусе, исправляем, не забываем после исправлений в юните перегружать демон systemd

systemctl daemon-reload
.service: Данный юнит описывает, как управлять службой или приложением на сервере. Он в себе будет включать запуск или остановку службу, при каких обстоятельствах она должна автоматически запускаться, а также информацию о зависимости для соответствующего программного обеспечения.







**Системные таймеры и чтение информации о них**
Когда система РЕД ОС устанавливается на компьютер, создается несколько системных таймеров, являющихся частью процедур обслуживания системы.

Чтобы увидеть, какие таймеры работают на данный момент, введите команду:

 systemctl status *timer 
 
 ● dnf-makecache.timer - dnf makecache --timer
     Loaded: loaded (/usr/lib/systemd/system/dnf-makecache.timer; enabled; vendo
     Active: active (waiting) since Thu 2022-06-23 10:03:35 MSK; 4h 37min ago
    Trigger: Thu 2022-06-23 15:18:12 MSK; 37min left
   Triggers: ● dnf-makecache.service

 июн 23 10:03:35 localhost.localdomain systemd[1]: Started dnf makecache --timer.
 
 ● sysstat-summary.timer - Generate summary of yesterday's process accounting
     Loaded: loaded (/usr/lib/systemd/system/sysstat-summary.timer; enabled; ven
     Active: active (waiting) since Thu 2022-06-23 10:03:35 MSK; 4h 37min ago
    Trigger: Fri 2022-06-24 00:07:00 MSK; 9h left
   Triggers: ● sysstat-summary.service

 июн 23 10:03:35 localhost.localdomain systemd[1]: Started Generate summary of ye
 
 ● systemd-tmpfiles-clean.timer - Daily Cleanup of Temporary Directories
     Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-clean.timer; stati
     Active: active (waiting) since Thu 2022-06-23 10:03:35 MSK; 4h 37min ago
    Trigger: Fri 2022-06-24 10:19:11 MSK; 19h left
   Triggers: ● systemd-tmpfiles-clean.service
       Docs: man:tmpfiles.d(5)
             man:systemd-tmpfiles(8)

 июн 23 10:03:35 localhost.localdomain systemd[1]: Started Daily Cleanup of Tempo
 
 ● sysstat-collect.timer - Run system activity accounting tool every 10 minutes
     Loaded: loaded (/usr/lib/systemd/system/sysstat-collect.timer; enabled; ven
     Active: active (waiting) since Thu 2022-06-23 10:03:35 MSK; 4h 37min ago
    Trigger: Thu 2022-06-23 14:50:00 MSK; 9min left
   Triggers: ● sysstat-collect.service

 июн 23 10:03:35 localhost.localdomain systemd[1]: Started Run system activity ac


С каждым таймером связано, по меньшей мере, шесть строк, содержащих сведения о нём:

Имя файла таймера и короткое описание цели существования этого таймера (но не всегда).

Сведения о состоянии таймера - загружен ли таймер, полный путь к файлу таймера, состояние vendor preset (disabled или enabled).

Сведения об активности таймера, куда входят данные о том, когда именно таймер был активирован.

Дата и время следующего запуска таймера и примерное время, оставшееся до его запуска.

Имя сервиса или события, вызываемого таймером.

Некоторые (но не все) таймеры содержат указатели на документацию.


Примеры использования таймеров
Самым простым примером использования таймеров можно считать системный таймер systemd-tmpfiles-clean. Как следует из названия, данный таймер раз в некоторое время производит очистку временных файлов системы.

 ● systemd-tmpfiles-clean.timer - Daily Cleanup of Temporary Directories
     Loaded: loaded (/usr/lib/systemd/system/systemd-tmpfiles-clean.timer; stati
     Active: active (waiting) since Thu 2022-06-23 10:03:35 MSK; 4h 37min ago
    Trigger: Fri 2022-06-24 10:19:11 MSK; 19h left
   Triggers: ● systemd-tmpfiles-clean.service
       Docs: man:tmpfiles.d(5)
             man:systemd-tmpfiles(8)
Данный таймер является системным и включен по умолчанию.



Создание таймеров
В системе РЕД ОС юниты systemd располагаются в следующих директориях:

/usr/lib/systemd/system – юниты, поставляемые вместе с системой и устанавливаемыми приложениями;

/etc/systemd/system – юниты системного администратора.

Для примера будет создан простой файл конфигурации сервиса, запускающий команду free (утилита, выводящая информацию об использовании оперативной памяти). Это может пригодиться, если необходимо регулярно отслеживать объём свободной памяти.

Создайте unit-файл с именем ramMonitor.service в папке /etc/systemd/system.

 nano /etc/systemd/system/ramMonitor.service
И добавьте в него следующие строки:

 [Unit]
 Description=Monitors RAM # Описание сервиса
 Wants=ramMonitor.timer # Зависимость, отсылается к таймеру (будет создан позже)
 
 [Service]
 Type=oneshot # Тип службы, означающий выполнение одного задания и завершение
 ExecStart=/usr/bin/free # Указывается полный путь к исполняемому файлу программы
 
 [Install]
 WantedBy=multi-user.target # Указывается, на каком уровне происходит запуск сервиса. Параметр multi-user.target указывает на запуск в многопользовательском режиме без графики 
Данный файл можно назвать простейшим файлом конфигурации сервиса.

После проведенных манипуляций, проверьте состояние вашего сервиса командой:

 systemctl status ramMonitor.service 

● ramMonitor.service - Monitors RAM
     Loaded: loaded (/etc/systemd/system/ramMonitor.service; disabled; vendor pr
     Active: inactive (dead)
Сервис существует, но не активен. Запустите сервис и проверьте его статус:

systemctl start ramMonitor.service
systemctl status ramMonitor.service 

 ● ramMonitor.service - Monitors RAM
      Loaded: loaded (/etc/systemd/system/ramMonitor.service; disabled; vendor pr
      Active: inactive (dead)
 
 июн 23 15:08:44 localhost.localdomain systemd[1]: Starting Monitors RAM...
 июн 23 15:08:44 localhost.localdomain systemd[1]: ramMonitor.service: Succeeded.
 июн 23 15:08:44 localhost.localdomain systemd[1]: Finished Monitors RAM.
 июн 23 15:08:44 localhost.localdomain free[3914]:               total        use
 июн 23 15:08:44 localhost.localdomain free[3914]: Mem:         995784      53659
 июн 23 15:08:44 localhost.localdomain free[3914]: Swap:       2097148        927
При включении сервиса в терминале не выводится никаких сообщений. Это происходит, потому что по умолчанию стандартный вывод (stdout) от программ, запускаемых systemd, перенаправляется в журнал systemd.

Чтобы проверить этот журнал, воспользуйтесь командой:

 journalctl -u ramMonitor.service 
 
 -- Logs begin at Thu 2022-04-21 11:49:59 MSK, end at Thu 2022-06-23 15:12:30 MSK
 июн 23 15:08:44 localhost.localdomain systemd[1]: Starting Monitors RAM...
 июн 23 15:08:44 localhost.localdomain systemd[1]: ramMonitor.service: Succeeded.
 июн 23 15:08:44 localhost.localdomain systemd[1]: Finished Monitors RAM.
 июн 23 15:08:44 localhost.localdomain free[3914]:               total        use
 июн 23 15:08:44 localhost.localdomain free[3914]: Mem:         995784      53659
 июн 23 15:08:44 localhost.localdomain free[3914]: Swap:       2097148        927
С помощью ключа -S вы можете указать временной период, за который утилита journalctl будет искать записи. Например,  journalctl -S today -u ramMonitor.service  выведет записи, сделанные за сегодня.



Убедившись в работоспособности сервиса, создайте для него таймер. Для этого необходимо создать в той же папке файл с названием ramMonitor.timer.

nano /etc/systemd/system/ramMonitor.timer
В файл добавьте следующее содержимое:

 [Unit]
 Description=Monitors RAM 
 Requires=ramMonitor.service # Указание строгой зависимости
 
 [Timer]
 Unit=ramMonitor.service # Ссылка на сервис
 OnCalendar=*-*-* *:*:00 # Срабатывание таймера по условию календаря. В данном случае – каждую минуту
 
 [Install]
 WantedBy=timers.target 
В секции [Timer] указываются условия запуска. Таймеры могут быть двух типов: событийные и монотонные. Первые активируются по событию, вторые выполняются периодически. Из событий таймеров можно выделить OnBootSec, срабатывающий через определенное время после запуска системы. Из монотонных следует выделить:

OnUnitActiveSec - срабатывает через указанное время после активации целевого юнита;

OnUnitInactiveSec — срабатывает так же, как OnUnitActiveSec, только время отсчитывается с момента прекращения работы целевого юнита, хорошо подходит для «длительных» задач, например, бекапов.

OnCalendar - срабатывает по условию календаря.

В качестве формата даты для календаря используется формат:

 DOW YYYY-MM-DD HH:MM:SS  
(где DOW – Day Of Week – день недели, необязательный параметр; за ним следует указание года, месяца, дня через дефис и часы, минуты и секунды через двоеточие. Для указания любого значения используется «*», перечисления делаются через запятую, а диапазоны через «..»).

После завершения настройки нужно запустить таймер и проверить его статус:

 systemctl start ramMonitor.timer
 systemctl status ramMonitor.timer 
 
 ● ramMonitor.timer - Monitors RAM
     Loaded: loaded (/etc/systemd/system/ramMonitor.timer; disabled; vendor pres
     Active: active (waiting) since Thu 2022-06-23 15:16:22 MSK; 7s ago
    Trigger: Thu 2022-06-23 15:17:00 MSK; 29s left
   Triggers: ● ramMonitor.service

июн 23 15:16:22 localhost.localdomain systemd[1]: Started Monitors RAM.
Если все настроено верно, таймер будет иметь состояние active (waiting), а ниже будет указано время до его запуска.



Преимущества и недостатки работы с таймерами
Основные преимущества:

Задачи могут быть легко запущены сразу, независимо от их таймеров - это упрощает отладку.

Каждая задача может быть настроена для работы в определенной среде.

Задачи могут быть настроены в зависимости от других юнитов systemd.

Задачи регистрируются в журнале systemd.

Единственным недостатком можно считать сложность создания таких таймеров: чтобы настроить задачу, запускаемую в определенное время при помощи systemd, нужно создать два файла и использовать команды systemctl. По сравнению с добавлением одной строки в crontab. Кроме того в systemd отсутствует встроенный аналог cron’овского MAILTO для отправки писем при сбоях.
