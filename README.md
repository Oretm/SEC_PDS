# Установка и настройка Ubuntu
  + Установка Ubuntu Server 14.04 LTS <http://releases.ubuntu.com/trusty/ubuntu-14.04.5-server-amd64.iso>
  + Добавление пользователя cephthree в список исключения sudo (Чтобы не вводить каждый раз пароль):
    В файле /etc/sudoers находим строку '%sudo   ALL=(ALL:ALL) ALL' и заменяем её на '%sudo   ALL=(ALL:ALL) NOPASSWD: ALL'
  + Добавление алиаса ceph='sudo ceph' (для удобства в будущем):
    Открываем файл ~/.bashrs и в конец файла добавляем строку alias ceph='sudo ceph', и перезагружаем оболочку
  + Создание ssh-ключей, чтобы не вводить пароль каждый раз, при подключении к другому хосту по ssh:
    - `ssh-keygen` // Создаем ключ для подключения к хостам с первого без набора пароля
    - `ssh-copy-id cephthree-01` // Копируем ключ на cephthree-01
    - `ssh-copy-id cephthree-02` // Копируем ключ на cephthree-02
    - `ssh-copy-id cephthree-03` // Копируем ключ на cephthree-03

# Развертывание ceph
  + Скачивание ceph-deploy:
    - `sudo apt install ceph-deploy`
  + Разворачиваем Ceph
    - `ceph-deploy new cephthree-01 cephthree-02 cephthree-03` // Создание конфига ceph.conf
    - `ceph-deploy install --release jewel cephthree-01 cephthree-02 cephthree-03 // Установка Ceph 0.80.11 Jewel на cephthree-01, cephthree-02, cephthree-03`
    - `ceph -v` // Посмотреть версию (проверка того, что ceph установился)
  + Установка монитров для ceph:
    `ceph-deploy mon create-initial`
  + Установка OSD:
    - `ceph-deploy disk list cephthree-01` // Просмотреть список доступных дисковых устройств
    - `ceph-deploy disk zap cephthree-01:sdb cephthree-01:sdc cephthree-01:sdd` // Уничтожение как GPT, так и MBR на устройствах sdb, sdc, sdd
    - `ceph-deploy osd create cephthree-01:sdb cephthree-01:sdc cephthree-01:sdd` // Создание OSD на устройствах sdb, sdc, sdd

    Делаем всё то-же самое для cephthree-02, cephthree-03
    -------------------------------------------------------------------------------------------------------------------------------
    - `ceph-deploy disk list cephthree-02`
    - `ceph-deploy disk zap cephthree-02:sdb cephthree-02:sdc cephthree-02:sdd`
    - `ceph-deploy osd create cephthree-02:sdb cephthree-02:sdc cephthree-02:sdd`

    - `ceph-deploy disk list cephthree-03`
    - `ceph-deploy disk zap cephthree-03:sdb cephthree-03:sdc cephthree-03:sdd`
    - `ceph-deploy osd create cephthree-03:sdb cephthree-03:sdc cephthree-03:sdd`
  + Проверяем статус ceph:
    `ceph -s`
  + Должен быть *WARNING*: clock skew. Он появляется из-за того, что время на хостах не синхронизировано. Исправляем:
    - `sudo apt install ntp ntpdate ntp-doc` // Устанавливаем ntpdate и ntp-doc
    - В файле /etc/ntp.conf находим добавляем сервера времени
    - `sudo service ntp restart` // Перезагружаем ntp
    - `ceph -s` //Проверяем, через некоторое время *WARNING* должен исчезнуть

# Установка и настройка iSCSI Target
  + Создание RBD:
    - `sudo rbd create iscsi --size 10240` // Создание образа iscsi размером 10240 МБ
    - `sudo rbd ls` // Проверяем, что образ создан
    - `sudo rbd map --image iscsi` // Добавляем образ iscsi в карту rbd
    - `sudo rbd showmapped` // Проверяем, что образ iscsi добавился в карту
  + Настройка iSCSI Target:
    - `sudo apt install tgt` // Устанавливаем iSCSI Target
    - В ~/ceph.conf обязательно отключаем кеширование rbd. Для этого добавляем в этот файл следующие строки:
       `[client]
       rbd_cache = false`
    - Задаем экспорт по iscsi rbd тома, в файле/etc/tgt/targets.conf:
        `<target iqn.2016-11.rbdstore.iscsi.com:iscsi>
            driver iscsi
            bs-type rbd
            backing-store rbd/iscsi
            initiator-address ALL
        </target>`
    - `sudo service tgt restart` // Перезапускаем iscsi target
    - `sudo tgt-admin -s` // Просматриваем текущую настройку iSCSI Target
    - Подключаем диск в Windows через Инициатор iSCSI, и всё!
