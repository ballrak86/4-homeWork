## Описание файлов в директории
logFileFull.log - полный лог выполнения
used_commands.txt - команды которые использовал

## Кратко по нужным командам
### алгоритм с наилучшим сжатием
```
zpool create zfsCompress /dev/sd[b-e]
zfs create zfsCompress/gzip
zfs set compression=gzip-9 zfsCompress/gzip
for compress in lz4 lzjb zle; do zfs create zfsCompress/$compress; zfs set compression=$compress zfsCompress/$compress;done
```
скопировал файл War_and_Peace.txt на все созданные диски
```
zfs list
```
NAME             |  USED | AVAIL   |  REFER|  MOUNTPOINT        
:----------------|:-----:|:-------:|:-----:|:------------:|
zfsCompress      | 4.75M | 3.62G   |  1.28M|  /zfsCompress
zfsCompress/gzip |  478K | 3.62G   |   478K|  /zfsCompress/gzip
zfsCompress/lz4  |  772K | 3.62G   |   772K|  /zfsCompress/lz4
zfsCompress/lzjb |  920K | 3.62G   |   920K|  /zfsCompress/lzjb
zfsCompress/zle  | 1.18M | 3.62G   |  1.18M|  /zfsCompress/zle

#### Вывод:
Наилучшее сжатие у разных методов такое (от лучшего к худшему):
метод |занимаемое место
:-----|----------------:
gzip  |478K (compression=gzip-9)
lz4	  |772K
lzjb  |920K
zle	  |1.18M

По умолчанию алгоритм сжатия lzjb, или если включен lz4_compress, то используется lz4.

### Определить настройки pool’a
Скачал архив, извлек из него файлы и выполнил комманды ниже
```
zpool import -d ${PWD}/zpoolexport/
zpool import -d ${PWD}/zpoolexport/ otus
[root@server ~]# zpool list
```
NAME        | SIZE|  ALLOC| FREE | CKPOINT|  EXPANDSZ|   FRAG|    CAP|  DEDUP |   HEALTH|  ALTROOT
:-----------|:---:|:-----:|:----:|:------:|:--------:|:-----:|:-----:|:------:|:-------:|:-------:|
otus        | 480M| 2.09M |  478M|      - |        - |    0% |    0% | 1.00x  |  ONLINE | -
zfsCompress |3.75G| 4.82M | 3.75G|      - |        - |    0% |    0% | 1.00x  |  ONLINE | -

Размер хранилища otus SIZE=480M
```
[root@server ~]# zpool status otus
  pool: otus
 state: ONLINE
  scan: none requested
config:

        NAME                         STATE     READ WRITE CKSUM
        otus                         ONLINE       0     0     0
          mirror-0                   ONLINE       0     0     0
            /root/zpoolexport/filea  ONLINE       0     0     0
            /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
Тип pool mirror-0

[root@server ~]# zfs get recordsize | grep otus
otus              recordsize  128K     local
otus/hometask2    recordsize  128K     inherited from otus
```
Значение recordsize=128K
```
[root@server ~]# zfs get compression | grep otus
otus              compression  zle       local
otus/hometask2    compression  zle       inherited from otus
```
Сжатие используется zle
```
[root@server ~]# zfs get checksum
NAME              PROPERTY  VALUE      SOURCE
otus              checksum  sha256     local
otus/hometask2    checksum  sha256     inherited from otus
zfsCompress       checksum  on         default
zfsCompress/gzip  checksum  on         default
zfsCompress/lz4   checksum  on         default
zfsCompress/lzjb  checksum  on         default
zfsCompress/zle   checksum  on         default
```
Контрольная сумма checksum=sha256

### сообщение от преподавателей
Скачал файл снапшота и восстановил файлы из него
```
zfs receive otus/messages < otus_task2.file
zfs list
```
нашел нужный файл через find и прочитал его
```
cat /otus/messages/task1/file_mess/secret_message
https://github.com/sindresorhus/awesome
```

## Подробности по встретившимся параметрам
1. Методы сжатия
Встроенное сжатие ZFS отключено по умолчанию, и система предлагает подключаемые алгоритмы — сейчас среди них LZ4, gzip (1-9), LZJB и ZLE.
* LZ4 — это потоковый алгоритм, предлагающий чрезвычайно быстрое сжатие и декомпрессию и выигрыш в производительности для большинства случаев использования — даже на довольно медленных CPU.
* GZIP — почтенный алгоритм, который знают и любят все пользователи Unix-систем. Он может быть реализован с уровнями сжатия 1-9, с увеличением степени сжатия и использования CPU по мере приближения к уровню 9. Алгоритм хорошо подходит для всех текстовых (или других чрезвычайно сжимаемых) вариантов использования, но в противном случае часто вызывает проблемы c CPU — используйте его с осторожностью, особенно на более высоких уровнях.
* LZJB — оригинальный алгоритм в ZFS. Он устарел и больше не должен использоваться, LZ4 превосходит его по всем показателям.
* ZLE — кодировка нулевого уровня, Zero Level Encoding. Она вообще не трогает нормальные данные, но сжимает большие последовательности нулей. Полезно для полностью несжимаемых наборов данных (например, JPEG, MP4 или других уже сжатых форматов), так как он игнорирует несжимаемые данные, но сжимает неиспользуемое пространство в итоговых записях.

2. Свойство recordsize
Это свойство предназначено исключительно для использования с базами данных, которые обращаются к файлам с записями фиксированного размера. ZFS автоматически регулирует размер блока в соответствии с внутренними алгоритмами, оптимизированными для типичных моделей доступа. Для баз данных, создающих крупные файлы, но обращающихся к файлам в небольших произвольных блоках, эти алгоритмы могут быть близки к оптимальным. Если для свойства recordsize указать значение, превышающее размер записи базы данных или равное ему, производительность может существенно возрасти. Использование этого свойства для универсальных файловых систем достаточно затруднительно и может негативно сказаться на производительности. Можно указать размер не менее 512 и не более 128 КБ, соответствующий степени двойки. Изменение свойства recordsize для файловой системы оказывает влияние только на файлы, созданные после изменения. На существующие файлы изменения не распространяются.

3. checksum
Используется для настройки контрольной суммы в целях проверки целостности данных. По умолчанию используется значение on, задающее автоматический выбор соответствующего алгоритма (в данном случае fletcher2). Возможные значения: on, off, fletcher2, fletcher4 и sha256. Значение off отключает проверку целостности пользовательских данных. Значение off использовать не рекомендуется