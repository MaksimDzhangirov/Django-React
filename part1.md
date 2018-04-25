# Современное приложение на Django: Часть 1: Настраиваем Django и React

## Введение

Данное учебное пособие будет состоять из нескольких частей и посвящено созданию "современного" веб-приложения или одностраничного приложения (SPA - single page application), используя Django и React.

Мы создадим приложение одностраничное приложение "Записная книжка", которое будет отображаться с помощью ReactJS и используем Django для создания API бэкэнда. Назовём наше приложение «ponynote», потому что пони популярные в сообществе Django.

Мы будем использовать несколько библиотек в этом проекте, в основном, `django-rest-framework` - чтобы упростить создание API, `agent-router-dom` - для маршрутизации приложения, `redux` - для отслеживания глобального состояния приложения.

Эта часть пособия посвящена настройке Вашего проекта, чтобы в дальнейшем без проблем использовать Django и React.

Код этого приложения находится на моём github аккаунте [v1k45/ponynote](https://github.com/v1k45/ponynote). Все изменения сделанные в этой части находятся в ветке `part-1`.

## Настраиваем окружение

Я предполагаю, что Вы сможете установить на своей машине Python3 и Node >=6 самостоятельно или используя инструкции, доступные в Интернете.

Я рекомендую Вам использовать python virualenv, чтобы зависимости, используемые в этом проекте, не конфликтовали с глобально установленной версией Python. Я использую [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/), что позволяет запускать несколько виртуальных окружений на одной машине с помощью простых команд. Я буду использовать его в этом приложении.

## Настраиваем Django

После того как Вы создали и активировали Ваше виртуальное окружение, Вы можете установить Django с помощью следующей команды:

```
(ponynote)  $ pip install django
```
После установки Django мы создадим проект с название `ponynote`, используя инструмент `django-admin`.
```
(ponynote)  $ django-admin startproject ponynote
```
После чего будет сгенерирован следующий проект:
```
(ponynote)  $ tree
.
└── ponynote
    ├── manage.py
    └── ponynote
        ├── __init__.py
        ├── settings.py
        ├── urls.py
        └── wsgi.py

2 directories, 5 files
```
После осуществления миграций и запуска сервера для разработки, мы должны увидеть надпись приветственную надпись Django "It worked!" в браузере по адресу [http://127.0.0.1:8000](http://127.0.0.1:8000).

```
(ponynote)  $ cd ponynote
(ponynote)  $ ./manage.py migrate
(ponynote)  $ ./manage.py runserver
```

Теперь, когда Django установлен и готов к работе, пора настроить React JS.

## Настраиваем ReactJS с помощью Webpack

Если Вы раньше не слышали о Webpack, то в настоящий момент он является дефакто инструментом для сборки модулей, позволяя компилировать все ваши зависимости в несколько небольших файлов (обычно в один). Чтобы узнать больше о Webpack, перейдите на его [официальный сайт](https://webpack.js.org/).

Мы не будем сами конфигурировать Webpack, а воспользуемся `create-react-app` для создания шаблона проекта с уже настроенной конфигурацией. Воспринимайте `create-react-app` как команду `django-admin startproject`, но с гораздо большими возможностями.

Прежде всего надо установить `create-react-app` так, чтобы оно была доступно на уровне всей системы с помощью `npm`. Поскольку мы устанавливаем приложение на уровне всей системы, нам потребуются права суперпользователя.

```
$ sudo npm install -g create-react-app
```

После установки мы создадим приложение React, используя эту команду, а затем извлечем конфигурационные файлы, чтобы мы могли их редактировать сами вручную.

```
$ create-react-app frontend
$ cd frontend
$ npm run eject
```

После выполнения этих команд, мы получим следующую структуру файлов:

```
$ tree -I node_modules
.
├── config
│   ├── env.js
│   ├── jest
│   │   ├── cssTransform.js
│   │   └── fileTransform.js
│   ├── paths.js
│   ├── polyfills.js
│   ├── webpack.config.dev.js
│   ├── webpack.config.prod.js
│   └── webpackDevServer.config.js
├── package.json
├── public
│   ├── favicon.ico
│   ├── index.html
│   └── manifest.json
├── README.md
├── scripts
│   ├── build.js
│   ├── start.js
│   └── test.js
└── src
    ├── App.css
    ├── App.js
    ├── App.test.js
    ├── index.css
    ├── index.js
    ├── logo.svg
    └── registerServiceWorker.js

5 directories, 23 files
```

По названию каталогов достаточно легко понять где хранится какой файл.

Вы можете проверить работоспособность этого приложения, выполнив команду

```
$ npm run start
```

После её выполнения отобразится стандартное окно приветствия React.

![react](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part1/react.png)

По умолчанию сервер для разработки настроен так, что реагирует на изменения налету. Это означает, что любые изменения в исходных файлах будут мгновенно отображены в браузере, без необходимости обновлять страницу вручную.

## Интеграция React и Django

Теперь когда у нас есть рабочий сервер с Django и Webpack сервер для разработки, на котором работает React, интегрируем их таким образом, чтобы Django мог отображать SPA, используя свой сервер. Для этого будем использовать `django-webpack-loader` - приложение Django, которое внедряет теги link и script из пакетов, генерируемых динамически Webpack.

Начнём с установки `webpack_loader` в наш Django проект:

```
(ponynote)  $ pip install django-webpack-loader
```

Затем в файл проекта `settings.py` добавьте `webpack_loader` в список `INSTALLED_APPS`:

```
WEBPACK_LOADER = {
    'DEFAULT': {
            'BUNDLE_DIR_NAME': 'bundles/',
            'STATS_FILE': os.path.join(BASE_DIR, 'webpack-stats.dev.json'),
        }
}
```

Чтобы работать с главной страницей приложения нам нужно создать представление и шаблон в Django. Мы начнём с создания шаблона для главной страницы в каталоге `templates/index.html`, если смотреть из корневой папки проекта. Нам также нужно обновить настройки проекта, касающиеся путей к шаблонам, чтобы он смог обнаружить каталог для шаблонов.

Измените настройку TEMPLATES в файле `ponynote.settings.py`:

```
TEMPLATES = [
    {
        # ... other settings
        'DIRS': [os.path.join(BASE_DIR, "templates"), ],
        # ... other settings
    },
]
```

Добавьте следующий код в файл `templates/index.html`:

```
{% load render_bundle from webpack_loader %}
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width" />
    <title>Ponynote</title>
  </head>
  <body>
    <div id="root">
    </div>
      {% render_bundle 'main' %}
  </body>
</html>
```

В HTML элементе с id - `root` будет располагаться наше приложение React. Тэг render_bundle c аргументом 'main' позволяет вывести в шаблон тэг скрипт для пакета с названием 'main', который создаётся используя нашу конфигурацию Webpack.

Теперь когда готов шаблон, давайте создадим представление. Поскольку здесь используется простой шаблон, мы можем использовать непосредственно `TemplateView` в нашей конфигурации url. В файл проекта `ponynote.urls.py` добавьте:

```
from django.conf.urls import url
from django.contrib import admin
from django.views.generic import TemplateView

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^', TemplateView.as_view(template_name="index.html")),
]
```

Если Вы обновите вашу главную страницу Django в браузере, то увидите страницу с ошибкой и текстом `Are you sure webpack has generated the file and the path is correct?`. Это означает, что `webpack_loader` не смог найти, указанный нами файл в настройках.

![django-error](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part1/django-error.png)

Чтобы сгенерировать этот файл, нам нужно установить плагин `webpack-bundle-tracker` и настроить Webpack так, чтобы он сгенерировал нужные файлы. В каталоге `frontend`, выполните следующие команды:

```
$ npm install webpack-bundle-tracker --save-dev
```

В файле `frontend/config/paths.js` добавьте следующую пару ключ-значение в объект `module.exports`:

```
module.exports = {
  // ... other values
  statsRoot: resolveApp('../'),
}
```

В `frontend/config/webpack.config.dev.js` измените `publicPath` и `publicUrl` на [http://localhost:3000/](http://localhost:3000/).

```
const publicPath = 'http://localhost:3000/';
const publicUrl = 'http://localhost:3000/';
```