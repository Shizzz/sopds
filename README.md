#### Simple OPDS Catalog - Простой OPDS Каталог  
#### Author: Dmitry V.Shelepnev  
#### Версия 0.33 beta

#### 1. Простая установка Simple OPDS (используем простую БД sqlite3)

1.1 Установка проекта  
Загрузить архив с проектом можно с сайта www.sopds.ru, 
либо из github.com следующей командой:

	git clone https://github.com/mitshel/sopds.git

1.2 Зависимости.  
- Требуется Python не ниже версии 3.3 (используется атрибут zlib.Decompressor.eof, введенный в версии 3.3)  
- Django 1.9 (для Python 3.3 необходимо устанавливать Django 1.8: https://code.djangoproject.com/ticket/25868)
- Pillow 2.9.0
- apscheduler 3.3.0

Для работы проекта необходимо установить указанные  зависимости: 

	yum install python3                    # команда установки для RedHad, Fedora, CentOS
	pip3 install -r requirements.txt       # для Python 3.4 и выше
	pip3 install -r requirements-p33.txt   # для Python 3.3

1.3 Настраиваем ./sopds/settings.py (настройки в конце файла)

	LANGUAGE_CODE = 'ru-RU'
	
	SOPDS_ROOT_LIB = < Путь к каталогу с книгами >
	SOPDS_AUTH = < False | True >
	SOPDS_SCAN_SHED_MIN  = '0'
	SOPDS_SCAN_SHED_HOUR = '0,12'
    
1.4 Производим инициализацию базы данных и заполнение начальными данными (жанры)

	python3 manage.py migrate
	python3 manage.py sopds_util clear
	
1.5 Cоздаем суперпользователя

	python3 manage.py createsuperuser
	
1.6 Вручную запускаем разовое сканирование коллекции книг  
(Выполняется относительно долго: например, моя коллекция книг в архивах объемом 180Гб сканировалась в БД MYSQL - 1час; в БД SQLite3 - 5 часов)

	python3 manage.py sopds_scanner scan --daemon

1.7 Запускаем встроенный HTTP/OPDS сервер

	python3 manage.py sopds_server start --daemon
	
Однако наилучшим способом, все же является настройка в качестве HTTP/OPDS серверов Apache или Nginx 
(точка входа ./sopds/wsgi.py)
	
1.8 Запускаем SCANNER сервер (опционально, необходим для автоматизированного периодического пересканирования коллекции)  
Перед запуском SCANNER сервера необходимо убедится, что сканирование, запущеное в п.1.6 уже завершено,
т.к. может возникнуть ситуация с запуском параллельного процесса сканирования, что может привести к ошибкам.
Примите во внимание, что в  настройках, указанных в п.1.3 задан периодический запуск сканирования 2 раза 
в день 12:00 и 0:00.

	python3 manage.py sopds_scanner start --daemon
	
1.9 Доступ к информации  
Если все предыдущие шаги выполнены успешно, то к библиотеке можно получить доступ по следующим URL:  

>     OPDS-версия: http://<Ваш сервер>:8001/opds/  
>     HTTP-версия: http://<Ваш сервер>:8001/

Следует принять во внимание, что по умолчанию в проекте используется простая БД sqlite3, которая
является одно-пользовательской. Поэтому пока не будет завершен процесс сканирования, запущенный 
ранее пунктом 1.6 попытки доступа к серверу будут завершаться ошибкой 
"A server error occurred.  Please contact the administrator."  
Для устранения указанной проблемы необходимо ипользовать многопользовательские БД, Например MYSQL.
	
#### 2. Настройка базы данных MySQL (опционально, но очень желательно для увеличения производительности).
2.1 Для работы с большим количеством книг, очень желательно не использовать sqlite, а настроить для работы БД MySQL.
MySQL по сравнению с sqlite работает гораздо быстрее. Кроме того SQLite - однопользователская БД, т.е. во время сканирования доступ
к БД будет невозможен.

UBUNTU: для работы с БД Mysql в UBUNTU потребовалось установить доп пакет:   
   
    sudo apt-get install python3-mysqldb

Далее необходимо сначала в БД MySQL создать базу данных "sopds" и пользователя с необходимыми правами,
например следующим образом:

	mysql -uroot -proot_pass mysql  
	mysql > create database if not exists sopds default charset=utf8;  
	mysql > grant all on sopds.* to 'sopds'@'localhost' identified by 'sopds';  
	mysql > commit;  
	mysql > ^C  
	
2.2 Далее в конфигурационном файде нужно закомментировать строки подключения к БД sqlite и соответсвенно раскомментировать
строки подключения к БД Mysql:


	DATABASES = {
	    'default': {
	        'ENGINE': 'django.db.backends.mysql',
	        'NAME': 'sopds',
	        'HOST': 'localhost',
	        'USER': 'sopds',
	        'PASSWORD' : 'sopds',
	        'OPTIONS' : {
	            'init_command': "SET default_storage_engine=MyISAM;\
	                             SET sql_mode='';"
	        }
	    }
	}


    # DATABASES = {
    #    'default': {
    #        'ENGINE': 'django.db.backends.sqlite3',
    #        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    #    }         
    #}  

2.4 Использование InnoDB вместо MyISAM.  
Указанная конфигурация MySQL использует в качестве движка БД MyISAM.
Однако возможно использовать более современный движок InnoDB. Он несколько быстрее и поддерживает транзакции, что положительно скажется
на целостности БД. Но иногда на устаревших версиях MySQL (например MariaDB 5.5) с ним возникают проблемы из-за ограничений на максимальную длину
индексов.  
Тем не менее если у Вас современная версия MySQL, то в настройках БД Mysql Вместо указанных параметров OPTIONS просто используйте следующие:

    'OPTIONS' : {
        'init_command': """SET default_storage_engine=INNODB; \
                           SET sql_mode='STRICT_TRANS_TABLES'; \
                           SET NAMES UTF8; \
                           SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED
                        """
    }

2.5 Далее необходимо для инициализации и заполнения вновь созданной БД заново выполнить пункты 1.4 - 1.9 данной инструкции
Однако, если Вы уже ранее запустили HTTP/OPDS сервер и SCANNER сервер, то потребуется сначала остановить их:

	python3 manage.py sopds_server stop
	python3 manage.py sopds_scanner stop
	

#### 3. Настройка конвертации fb2 в EPUB или MOBI (опционально, можно не настраивать)  

3.1 Конвертер fb2-to-epub http://code.google.com/p/fb2-to-epub-converter/
- во первых необходимо скачать последнюю версию конвертера fb2toepub по ссылке выше (текущая уже находится в проекте)
  к сожалению конвертер не совершенный и не все книги может конвертировать, но большинство все-таки конвертируется 
- далее, необходимо скопировать архив в папку **./convert/fb2toepub** и разархивировать 
- далее, компилируем проект командой make, в результате в папке  unix_dist появится исполняемый файл fb2toepub 
- в конфигурационном файле ./sopds/settings.py необходимо задать путь к этому конвертеру, например таким образом:  

>     SOPDS_FB2TOEPUB = os.path.join(BASE_DIR,'convert/fb2toepub/unix_dist/fb2toepub')

- В результате OPDS-клиенту будут предоставлятся ссылки на FB2-книгу в формате epub  

3.2 Конвертер fb2epub http://code.google.com/p/epub-tools/ (конвертер написан на Java, так что в вашей системе должнен быть установлен как минимум JDK 1.5)  
- также сначала скачать последнюю версию по ссылке выше (текущая уже находится в проекте)  
- скопировать jar-файл например в каталог **./convert/fb2epub** (Здесь уже лежит shell-скрипт для запуска jar-файла)  
- Соответственно прописать пути в файле конфигурации **./sopds/settings.py** к shell-скрипту fb2epub (данный конвертер работает также и в Windows) 

>     SOPDS_FB2TOEPUB = os.path.join(BASE_DIR, 'convert\\fb2epub\\fb2epub.cmd' if sys.platform =='win32' else 'convert/fb2epub/fb2epub' )

3.3 Конвертер fb2conv (конвертация в epub и mobi) http://www.the-ebook.org/forum/viewtopic.php?t=28447  
- Необходимо установить python 2.7 и пакеты lxml, cssutils:   
  
         yum install python  
         yum install python-lxml  
         yum install python-cssutils  
  
- скачать последнюю версию конвертера по ссылке выше (текущая уже находится в каталоге fb2conv проекта)  
- скачать утилиту KindleGen с сайта Amazon http://www.amazon.com/gp/feature.html?ie=UTF8&docId=1000234621 
  (текущая версия утилиты уже находится в каталоге fb2conv проекта)  
- скопировать архив проекта в **./convert/fb2conv** (Здесь уже подготовлены shell-скрипты для запуска конвертера) и разархивировать его  
- Для конвертации в MOBI нужно архив с утилитой KindleGen положить в каталог с конвертером и разархивировать  
- В конфигурационном файле **./sopds/settings.py** задать пути к соответствующим скриптам:  
   
>     SOPDS_FB2TOEPUB = os.path.join(BASE_DIR,'convert/fb2conv/fb2epub')
>     SOPDS_FB2TOMOBI = os.path.join(BASE_DIR,'convert/fb2conv/fb2mobi')

#### 4. Опции каталогизатора Simple OPDS (www.sopds.ru)
Каталогизатор Simple OPDS имеет дополнительные настройки которые можно указывать в конце файла sopds/settings.py  

**SOPDS_ROOT_LIB** - содержит путь к каталогу, в котором расположена ваша коллекция книг.  

**SOPDS_BOOK_EXTENSIONS** - Список форматов книг, которые будут включаться в каталог.  
(по умолчанию SOPDS_BOOK_EXTENSIONS = ['.pdf', '.djvu', '.fb2', '.epub'])  
	
**SOPDS_DOUBLES_HIDE** - Скрывает, найденные дубликаты в выдачах книг.  
(по умолчанию SOPDS_DOUBLES_HIDE = True)  
	
**SOPDS_FB2PARSE** - Извлекать метаинформацию из книг fb2.  
(по умолчанию SOPDS_FB2PARSE = True)  
	
**SOPDS_COVER_SHOW** - способ показа обложек (False - не показывать, True - извлекать обложки на лету и показывать).  
(по умолчанию SOPDS COVER_SHOW = True)  
    
**SOPDS_ZIPSCAN** - Настройка сканирования ZIP архивов.  
(по умолчанию SOPDS_ZIPSCAN = True)  
	
**SOPDS_ZIPCODEPAGE** - Указываем какая кодировка для названий файлов используется в ZIP-архивах. Доступные кодировки: cp437, cp866, cp1251, utf-8. По умолчанию применяется кодировка cp437. Поскольку в самом ZIP архиве сведения о кодировке, в которой находятся имена файлов - отсутствуют, то автоматически определить правильную кодировку для имен файлов не представляется возможным, поэтому для того чтобы кириллические имена файлов не ваыглядели как крякозябры следует применять кодировку cp866.  
(по умолчанию SOPDS_ZIPCODEPAGE = "cp866")  

**SOPDS_INPX_ENABLE** - Если True, то при обнаружении INPX файла в каталоге, сканер не сканирует его содержимое вместе с подгаталогами, а загружает	данные из найденного INPX файла. Сканер считает что сами архивыс книгами расположены в этом же каталоге. Т.е. INPX-файл должен находится именно в папке с архивами книг.   
(по умолчанию SOPDS_INPX_ENABLE = True)  

**SOPDS_INPX_SKIP_UNCHANGED** - Если True, то сканер пропускает повторное сканирование, если размер INPX не изменялся.  
(по умолчанию SOPDS_INPX_SKIP_UNCHANGED = True)  

**SOPDS_INPX_TEST_ZIP** - Если  True, то сканер пытается найти описанный в INPX архив. Если какой-то архив не обнаруживается, то сканер не будет добавлять вязанные с ним данные из INPX в базу данных соответсвенно, если SOPDS_INPX_TEST_ZIP = False, то никаких проверок сканер не производит, а просто добавляет данные из INPX в БД. Это гораздо быстрее.  
(по умолчанию SOPDS_INPX_TEST_ZIP = False)  


**SOPDS_INPX_TEST_FILES** - Если  True, то сканер пытается найти описанный в INPX конкретный файл с книгой (уже внутри архивов). Если какой-то файл не обнаруживается, то сканер не будет добавлять эту книгу в базу данных соответсвенно, если INPX_TEST_FILES = False, то никаких проверок сканер не производит, а просто добавляет книгу из INPX в БД. Это гораздо быстрее.   
(по умолчанию SOPDS_TEST_FILES = False)  

**SOPDS_DELETE_LOGICAL** - True приведет к тому, что при обнаружении сканером, что книга удалена, запись в БД об этой книге будет удалена логически (avail=0). Если значение False, то произойдет физическое удаление таких записей из базы данных. Пока работает только SOPDS_DELETE_LOGICAL = False.  
(по умолчанию SOPDS_DELETE_LOGICAL = False)  

**SOPDS_SPLITITEMS** - Устанавливает при достижении какого числа элементов в группе - группа будет "раскрываться". Для выдач "By Title", "By Authors", "By Series".  
(по умолчанию SOPDS_SPLITITEMS = 300)  

**SOPDS_MAXITEMS** - Количество выдаваемых результатов на одну страницу.  
(по умолчанию SOPDS_MAXITEMS = 60)  

**SOPDS_FB2TOEPUB** и **SOPDS_FB2TOMOBI** задают пути к програмам - конвертерам из FB2 в EPUB и MOBI/  
(по умолчанию SOPDS_FB2TOEPUB = "")  
(по умолчанию SOPDS_FB2TOMOBI = "")  

**SOPDS_TEMP_DIR** задает путь к временному каталогу, который используется для копирования оригинала и результата конвертации.  
(по умолчанию SOPDS_TEMP_DIR = os.path.join(settings.BASE_DIR,'tmp'))  

**SOPDS_TITLE_AS_FILENAME** - Если True, то при скачивании вместо оригинального имени файла книги выдает транслитерацию названия книги.  
(по умолчанию SOPDS_TITLE_AS_FILENAME = True)  

**SOPDS_ALPHABET_MENU** - Включение дополнительного меню выбора алфавита.  
(по умолчанию SOPDS_ALPHABET_MENU = True)  

**SOPDS_NOCOVER_PATH** - Файл обложки, которая будет демонстрироваться для книг без обложек.  
(по умолчанию SOPDS_NOCOVER_PATH = os.path.join(settings.BASE_DIR,'static/images/nocover.jpg'))

**SOPDS_AUTH** - Включение BASIC - авторизации.  
(по умолчанию SOPDS_AUTH = False)  

**SOPDS_SERVER_LOG** и **SOPDS_SCANNER_LOG** задают размещение LOG файлов этих процессов.  
(по умолчанию SOPDS_SERVER_LOG = os.path.join(settings.BASE_DIR,'opds_catalog/log/sopds_server.log'))  
(по умолчанию SOPDS_SCANNER_LOG = os.path.join(settings.BASE_DIR,'opds_catalog/log/sopds_scanner.log'))  

**SOPDS_SERVER_PID** и **SOPDS_SCANNER_PID** задают размещение PID файлов этих процессов при демонизации.  
(по умолчанию SOPDS_SERVER_PID = os.path.join(settings.BASE_DIR,'opds_catalog/tmp/sopds_server.pid'))  
(по умолчанию SOPDS_SCANNER_PID = os.path.join(settings.BASE_DIR,'opds_catalog/tmp/sopds_scanner.pid'))  

Параметры **SOPDS_SCAN_SHED_XXX** устанавливают значения шедулера, для периодического сканирования коллекции книг при помощи **manage.py sopds_scanner start**.  Возможные значения можно найти на следующей странице: # https://apscheduler.readthedocs.io/en/latest/modules/triggers/cron.html#module-apscheduler.triggers.cron  
(по умолчанию SOPDS_SCAN_SHED_MIN = '0')  
(по умолчанию SOPDS_SCAN_SHED_HOUR = '0')  
(по умолчанию SOPDS_SCAN_SHED_DAY = '*')  
(по умолчанию SOPDS_SCAN_SHED_DOW = '*')  


