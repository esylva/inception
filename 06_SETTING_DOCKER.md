# Предварительная настройка Docker

Тепрь нам нужно начинать работать с докером. Для начала сделаем его удобным для нас, а так же протестируем его работу.

По умолчанию докер запускается либо с привилегией суперпользователя, либо любым пользователем, состоящем в группе docker и обладающим возможность делать запросы под суперпользователем (например, через sudo). 

Установим утиллиту sudo, позволяющую пользователю делать запросы от имени root. Для этого логинимся под суперпользователем ```su -``` и вводим команду:

```apt install sudo```

После успешной установки правим конфиг /etc/sudoers: ```nano /etc/sudoers```

Наша задача - добавить запись с именем нашего пользователя и правами, равнозначными правам root:

![Настройка Docker](media/setting_docker/step_5.png)

![Настройка Docker](media/setting_docker/step_6.png)

Сохраняем изменения и закрываем файл.

Теперь добавим нашего пользователя ```user``` в группу ```docker```

Вот так выглядит список групп нашего пользователя сейчас (```groups user```):

![Настройка Docker](media/setting_docker/step_0.png)

Добавим же нашего пользователя в группу командой 

```usermod -aG docker user```

И проверим, что добавление произошло:

![Настройка Docker](media/setting_docker/step_1.png)

Как мы можем видеть, в списке групп в самом конце добавилась группа docker. Это значит, что теперь мы можем вызывать наш докер из под обычного пользователя.

Так переключимся же на нашего пользователя и перейдём в его домашний каталог:

```su user```

```cd ~/```

Так же скачаем в корень простую конфигурацию из одного докер-контейнера для проверки работы системы:

```git clone https://github.com/codesshaman/simple_docker_nginx_html.git```

![Настройка Docker](media/setting_docker/step_2.png)

Теперь мы можем переходить в эту папку и запускать контейнер:

```cd simple_docker_nginx_html```

```docker-compose up -d```

Через некоторое время наш контейнер сбилдится и мы увидим сообщение об успешном запуске:

![Настройка Docker](media/setting_docker/step_3.png)

Это значит, что мы можем протестировать запущенный контейнер и правильность настройки конфигурации. Если на шаге 01 при пробросе портов мы всё сделали правильно, значит 80-й порт открыт, и зайдя в браузер по адресу локального хоста ```127.0.0.1``` мы увидим следующую картину:

![Настройка Docker](media/setting_docker/step_4.png)

Если вдруг мы видим что-либо другое, значит, у нас не открыты порты или 80-й порт чем-то занят на хостовой машине. Пройдитесь по гайду 01 и удостоверьтесь, что порты открыты, а так же проверьте все запущенные приложения. Если среди них есть сервера или иные приложения для работы с локальным хостом, отключаем их. Можно так же попробовать перезагрузить компьютер.

--

Итак, мы смогли запустить наш первый контейнер и обратитсья к нему по локальному адресу. Но согласно сабжекту, наш адрес должен быть login.42.fr, где login - наш ник в интре.

Для этого нам придётся перенастроить файл hosts нашей гостевой машины. Открываем его:
