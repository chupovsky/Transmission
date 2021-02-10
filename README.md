# Скрипты обработчики для Transmission-daemon и Transmission-remote
 
 ![GitHub](https://img.shields.io/github/license/GregoryGost/Transmission)
 ![GitHub](https://img.shields.io/github/repo-size/GregoryGost/Transmission)
 ![GitHub](https://img.shields.io/github/languages/code-size/GregoryGost/Transmission)

# Скрипт torrentdone.sh
 Данный скрипт выполняется после завершения скачивания каждого торрента в Transmission daemon  
 Для его работы он должен быть настроен в файле конфигурации через `script-torrent-done-enabled` и `script-torrent-done-filename`  
 >
 Для более подробной информации изучите статью [Домашний Сервер: Часть 4 – Настройка Transmission daemon в контейнере LXC Proxmox-VE](https://gregory-gost.ru/domashnij-server-chast-4-nastrojka-transmission-daemon-v-kontejnere-lxc-proxmox-ve/)
 >
 История версий:
 * v1.2.3 - (11.02.2021) Поправлены regex_film и regex_film_dir для определения фильмов. Bash не хочет понимать \d, хотя норм понимает \s. Странности.
 * v1.2.2 - (08.02.2021) Поправлено определение фильм или сериал после определения файл или дирректория. Теперь коллекции необходимо корректно именовать.
 * v1.2.1 - (26.01.2021) Удален параметр CLEARFLAG т.к. не используется.
 * v1.2.0 - (19.01.2021) Доработаны функции обработки файлов сериалов и фильмов. Поправлена работа с именем файла.
 * v1.1.0 - (09.01.2021) Переработан алгоритм и функции для работы с торрентами в которых загружаются несколько файлов сразу (т.е. они загружаются папкой с файлами)
 * v1.0.0 - (31.03.2020) Исправлена ошибка с кириллическими символами в имени файла при переходе в функцию
 * v0.9.16 - (24.03.2020) Улучшено комментирование кода
 * v0.9.15 - (21.03.2020) Добавлена обработка торрентов с несколькими файлами (папками). Изменен принцип логирования
 * v0.0.8 - (18.04.2018) Изменено регулярное выражение regex_film
 * v0.0.7 - (18.04.2018) Улучшено структурирование и комментирование кода
 * v0.0.6 - (17.04.2018) Улучшено комментирование кода
 * v0.0.5 - (17.04.2018) Изменена версия на корректную
 * v0.0.4 - (17.04.2018) Поправлено регулярное выражение regex_film
 * v0.0.3 - (17.04.2018) Улучшено комментирование кода
 * v0.0.2 - (17.04.2018) Улучшено комментирование кода
 * v0.0.1 - (17.04.2018) Исправлены условия определяющие корректно ли перемещен файл
 * NV - (17.04.2018) Первая версия
 
 В скрипт по умолчанию передаются следующие переменные: 
 * $TR_APP_VERSION: версия Transmission
 * $TR_TORRENT_ID: идентификатор (ID) торрента
 * $TR_TORRENT_NAME: имя торента в том виде, как оно отображается в интерфейсе Transmission Remote GUI
 * $TR_TORRENT_DIR: текущая папка торрента
 * $TR_TORRENT_HASH: хэш торрента
 * $TR_TIME_LOCALTIME: дата и время запуска скрипта

## Установка
 Данный скрипт необходимо поместить в папку `/etc/transmission-daemon`  
 Далее все команды от пользователя **root**
 ```
 cd /etc/transmission-daemon
 wget https://raw.githubusercontent.com/GregoryGost/Transmission/master/torrentdone.sh
 ```
 После скачивания указываем пользователя для файла и разрешаем файл на выполнение.
 ```
 chown debian-transmission:debian-transmission /etc/transmission-daemon/torrentdone.sh
 chmod +x /etc/transmission-daemon/torrentdone.sh
 ```

## Алгоритм обработки торрентов
 ```
 ── Это файл или Папка ?
   ├─ ЭТО ФАЙЛ
   | └─ Это Сериал или Фильм ?
   |   ├─ ЭТО СЕРИАЛ => serialprocess()
   |   └─ ЭТО ФИЛЬМ
   |     └─ Фильм 2D или 3D ?
   |       ├─ 2D => filmprocess()
   |       └─ 3D => filmprocess()
   ├─ ЭТО ПАПКА
     └─ Это Сериал или Фильм ? (мы не можем изменять названия файлов в папке)
       ├─ ЭТО СЕРИАЛ => serialprocess()
       └─ ЭТО ФИЛЬМ
         └─ Фильм 2D или 3D ?
           ├─ 2D => filmprocess()
           └─ 3D => filmprocess()
 ```

## Правила именования торрентов для корректной работы скрипта
 Нельзя просто так добавлять торренты в Transmission remote GUI или кидать торрент файлы в папку отслеживания с данным скриптом.  
 Если вы хотите, чтобы парсинг файлов и папок выполнялся корректно, необходимо соблюдать простые правила именования.  
 Некоторые примеры описаны в самом скрипте, чтобы не бегать на github если забыли.  
 
 **Сериалы**  
 Сериалы обрабатываются с помощью регулярных выражений `regex_ser`  
 Корректный пример:
 ```
 The.Mandalorian.S02E07.1080p.rus.LostFilm.TV.mkv
 ```
 Если сериал скачивается целым сезоном, то файлы будут лежать в папке и мы можем изменять только название папки.  
 Соответственно необходимо иметь в названии необходимые части.  
 Корректные примеры:
 ```
 The Mandalorian S02 - LostFilm.TV [1080p]
 Шерлок Холмс S01 Serial WEB-DL (1080p)
 ```
 
 **Фильмы**  
 Фильмы обрабатываются с помощью трех регулярных выражений `regex_film`, `regex_film_dir` и `regex_3d`  
 При добавлении одиночного фильма, необходимо его корректно назвать. Год может обрамляться скобками `(2021)`, нижними подчеркиваниями `_2021_`, точками `.2021.` и просто пробелами ` 2021 ` и комбинациями этих обрамлений.  
 Т.е. любой файл, где год обрамлен этими знаками будет корректно вычленен из имени файла.  
 Корректные примеры:
 ```
 Бегущий по лезвию (2019).mkv
 Аватар 3D (2009).mkv
 ```
 Если фильмы скачиваются трилогиями, дилогиями и т.д., то необходимо проверять внутренние файлы на наличие в них года. Иначе файл не будет скопирован т.к. не определится год.  
 Скачиваемый торрент также необходимо его корректно назвать. В названии должны присутствовать года затрагиваемые фильмами.  
 Года может обрамляться скобками `(2012-2014)`, нижними подчеркиваниями `_2012-2014_`, точками `.2012-2014.` и просто пробелами ` 2012-2014 ` и комбинациями этих обрамлений.  
 Разделитель между годами также может быть дефисом `(2012-2014)`, нижним подчеркиванием `(2012_2014)`, точкой `(2012.2014)` и просто пробелом `(2012 2014)`
 Пример корректных названий файлов в трилогиях:
 ```
 The Hobbit - The Battle of the Five Armies (2014) Extended 2160p HDR.mkv
 The Hobbit_The Battle Of The Five Armies_2014_Extended.Cut_BDRip_1080p_[HEVC]_10bit.mkv
 ```
 Пример корректного именования самого торрента с трилогией:
 ```
 Хоббит трилогия (2012-2014)
 Хоббит трилогия _2012_2014_
 Хоббит трилогия .2012.2014.
 Хоббит трилогия .2012-2014.
 ```
 
 
# Скрипт torrentclear
Скрипт необходим для автоматической очистки скачанных торрент файлов согласно условиям:  
1. Если торрент скачан на 100% и коэффициент отданного к скачанному (RATIO) больше, либо равен 2
2. Если кол-во дней на раздаче больше или равно 7  
Соответственно эти значения можно менять. Причем RATIO скрипт считывает из файла настроек самостоятельно (необходимо задать путь до него в скрипте)  
Больше информации по выводам команд применяемых в файле **torrentclear** вы можете найти в файле [torrentclear_example.md](./torrentclear_example.md)

История версий:
 * v1.0.1 - (26.01.2021) Удален параметр CLEARFLAG т.к. не используется
 * v1.0.0 - (10.01.2021) Добавлено определение директории. Если директория, то загруженное удаляется вместе с файлами т.к. мы по окончании загрузки и так все файлы скопировали
 * v0.9.17 - (24.03.2020) Изменен принцип сравнения "RATIO". Добавлено логирование RATIO
 * v0.9.16 - (24.03.2020) Добавлены комментарии к коду
 * v0.9.15 - (21.03.2020) Удалена отправка сообщений по email. Изменен принцип логирования. Небольшой рефакторинг
 * v0.0.13 - (19.04.2018) Локальные перемнные в функции отправки email заменены на глобальные
 * v0.0.11 - (18.04.2018) Добавлен комментарий к коду
 * v0.0.10 - (18.04.2018) echo заменен на функцию логирования
 * v0.0.9 - (18.04.2018) echo заменен на функцию логирования
 * v0.0.8 - (18.04.2018) Добавлен комментарий и вывод информации в консоль
 * v0.0.7 - (18.04.2018) Добавлен комментарий к коду
 * v0.0.6 - (18.04.2018) Изменена команда подключения к Transmission и добавлены комментарии
 * v0.0.5 - (18.04.2018) Добавлен комментарий к коду
 * v0.0.4 - (18.04.2018) Первая версия
 
## Установка
 Скрипт необходимо поместить в cron папку `/etc/cron.hourly/`  
 Также есть ряд других специальных папок: cron.daily, cron.monthly, cron.weekly
 ```
 cd /etc/cron.hourly/
 wget https://raw.githubusercontent.com/GregoryGost/Transmission/master/torrentclear
 ```
 Файл выполняется от **root**

# Ротация логов
 Скрипты пишут результат своей работы в LOG файлы **torrentdone.log** и **torrentclear.log**.  
 Log файлы располагаются по пути, где обычно хранятся все лог файлы:
 ```
 /var/log/transmission/torrentdone.log
 /var/log/transmission/torrentclear.log
 ```
 Ротация лог файлов обеспечивается подсистемой **logrotate**.  
 Ротация происходит для всех лог файлов папке `/var/log/transmission/`  
 Расположение файла настройки ротации логов:
 ```
 /etc/logrotate.d/transmission
 ```
 После создания или загрузки файла настройки, необходимо перезапустить службу:
 ```
 service logrotate restart
 ```
 
# Лицензирование
 Все исходные материалы для проекта распространяются по лицензии [GPL v3](./LICENSE "Описание лицензии").  
 Вы можете использовать проект в любом виде, в том числе и для коммерческой деятельности, но стоит помнить, что автор проекта не дает никаких гарантий на работоспособность исполняемых файлов, а так же не несет никакой ответственности по искам или за нанесенный ущерб.  