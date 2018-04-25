# Современное приложение на Django: Часть 1: Настраиваем Django и React

## Введение

Данное учебное пособие будет состоять из нескольких частей и посвящено созданию "современного" веб-приложения или одностраничного приложения (SPA - single page application), используя Django и React.

Мы создадим приложение одностраничное приложение "Записная книжка", которое будет отображаться с помощью ReactJS и используем Django для создания API бэкэнда. Назовём наше приложение «ponynote», потому что пони популярные в сообществе Django.

Мы будем использовать несколько библиотек в этом проекте, в основном, `django-rest-framework` - чтобы упростить создание API, `agent-router-dom` - для маршрутизации приложения, `redux` - для отслеживания глобального состояния приложения.

Эта часть пособия посвящена настройке Вашего проекта, чтобы в дальнейшем без проблем использовать Django и React.

Код этого приложения находится на моём github аккаунте [v1k45/ponynote](https://github.com/v1k45/ponynote). Все изменения, произведённые в этой части, находятся в ветке `part-1`.

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
После осуществления миграций и запуска сервера для разработки, мы должны увидеть приветственную надпись Django "It worked!" в браузере по адресу [http://127.0.0.1:8000](http://127.0.0.1:8000).

```
(ponynote)  $ cd ponynote
(ponynote)  $ ./manage.py migrate
(ponynote)  $ ./manage.py runserver
```

Теперь, когда Django установлен и готов к работе, пора настроить React JS.

## Настраиваем ReactJS с помощью Webpack

Если Вы раньше не слышали о Webpack, то в настоящий момент он является дефакто инструментом для сборки модулей, позволяя компилировать все ваши зависимости в несколько небольших файлов (обычно в один). Чтобы узнать больше о Webpack, перейдите на его [официальный сайт](https://webpack.js.org/).

Мы не будем сами конфигурировать Webpack с нуля, а воспользуемся `create-react-app` для создания шаблона проекта с уже настроенной конфигурацией. Воспринимайте `create-react-app` как команду `django-admin startproject`, но с гораздо большими возможностями.

Прежде всего надо установить `create-react-app` так, чтобы оно было доступно на уровне всей системы с помощью `npm`. Поскольку мы устанавливаем приложение на уровне всей системы, нам потребуются права суперпользователя.

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

После её выполнения отобразится стандартное страница приветствия React.

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

В том же файле импортируйте `webpack-bundle-tracker` и добавьте `BundleTracker` в плагины Webpack, а также замените строку `webpackHotDevClient` на два других модуля.

const BundleTracker = require('webpack-bundle-tracker');

```
module.exports = {
  entry: [
    // ... НЕ ИЗМЕНЯЙТЕ ДРУГИЕ СТРОКИ В ФАЙЛЕ
    // строка, которую нужно заменить является примерно 30 строкой в файле
    require.resolve('webpack-dev-server/client') + '?http://localhost:3000',
    require.resolve('webpack/hot/dev-server'),
    // require.resolve('react-dev-utils/webpackHotDevClient'),
  ],
  plugins: [
    // заменяем плагин, который находится примерно на 215-220 строке файла.
    // ... другие плагины
    new BundleTracker({path: paths.statsRoot, filename: 'webpack-stats.dev.json'}),
  ],
}
```

Мы заменяем `webpackHotDevClient`, `publicPath` и `publicUrl`, потому что хотим, чтобы для пакетов, созданных Webpack, был указан верный путь на Django страницах и мы не хотим, чтобы Webpack посылал запросы на неверные url/хосты.

Теперь в `frontend/config/webpackDevServer.config.js` нам нужно разрешить серверу принимать запросы от внешних источников. [http://localhost:8000](http://localhost:8000) (сервер Django) отправит XHR запросы на Webpack сервер [http://localhost:3000](http://localhost:3000), чтобы проверить не изменились ли исходные файлы.

Добавьте объект `headers` в объект, который возвращается экспортируемой функцией. Этот объект должен располагаться на том же уровне вложенности, что и свойства `https` и `host`.

```
headers: {
  'Access-Control-Allow-Origin': '*'
},
```

Это всё что нужно сделать. Перезапустите Webpack сервер и перейдите на страницу [http://localhost:8000](http://localhost:8000). При этом Вы должны увидеть ту же стандартную приветственную страницу React, которую Вы видели, когда использовали Webpack сервер.

Теперь любые изменения, производимые Вами в файле `src/App.js`, будут мгновенно отображаться в браузере, благодаря возможности производить замены налету. Произведите следующие изменения в файле `src/App.js`:

```
import React, { Component } from 'react';
import logo from './logo.svg';
import './App.css';

class App extends Component {
  render() {
    return (
      <div className="App">
        <header className="App-header">
          <img src={logo} className="App-logo" alt="logo" />
          <h1 className="App-title">Welcome to Ponynote</h1>
        </header>
        <p className="App-intro">
            A react app with django as a backend.
        </p>
      </div>
    );
  }
}

export default App;
```

После этого отображаемая в браузере страница изменится следующим образом:

![new-react-page](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part1/react.png)

## Настройки для запуска на продакшен сервере

Поскольку это пособие состоит из нескольких частей Вам нужно будет изучить его 4/5 часть прежде, чем Вы достигните части, которую я хотел бы посвятить развертыванию приложения на реальном сервере. Я пишу это руководство по развертыванию Webpack для тех, кто не хочет ждать и тех, кто знает React и им нужны настройки для Django + Webpack для продакшен сервера.

Настройки для продакшен сервера практически те же, что мы использовали для разработки. Нам нужно отредактировать только несколько параметров в файле `webpack.config`.

Начнём с создания файла `ponynote/production_settings.py` в том же каталоге проекта, что и `settings.py`. Для этого проекта, продакшен сервер будет использовать следующий файл с настройками:

```
from .settings import *

STATICFILES_DIRS = [
    os.path.join(BASE_DIR, "assets"),
]

WEBPACK_LOADER = {
    'DEFAULT': {
            'BUNDLE_DIR_NAME': 'bundles/',
            'STATS_FILE': os.path.join(BASE_DIR, 'webpack-stats.prod.json'),
        }
}
```

Вышеприведенный конфигурационный файл указывает Django искать статические файлы в каталоге `assets` проекта и использовать `webpack-stats.prod.json` в качестве файла статистики Webpack.

Затем создайте каталог `assets/bundles/` в корневом каталоге Вашего проекта. В этом каталоге будут хранится все наши статические ресурсы. В подкаталог `bundles` будут помещаться файлы, скомпилированные Webpack.

```
$ mkdir -p assets/bundles
```

После создания каталога измените каталог, в который Webpack помещает результат сборки, на `assets/bundles`. В файле `frontend/config/paths.js` измените значение `appBuild`:

```
// измените файл после его извлечения: он должен находится в ./config/
module.exports = {
  // ... НЕ ИЗМЕНЯЙТЕ ДРУГИЕ ЗНАЧЕНИЯ
  appBuild: resolveApp('../assets/bundles/'),
};
```

Теперь в файле `frontend/config/webpack.config.prod.js` произведите следующие изменения:

```
const BundleTracker = require('webpack-bundle-tracker');

const publicPath = "/static/bundles/";

const cssFilename = 'css/[name].[contenthash:8].css';

module.exports = {
  // НЕ ИЗМЕНЯЙТЕ ДРУГИЕ ЗНАЧЕНИЯ
  output: {
    // Примерно 67 строка

    // Генерируемые названия JS файлов (вместе с вложенными каталогами).
    // Будет создан один основной пакет и по одному файлу для каждой асинхронной части.
    // Мы не будем разбивать код на части, но Webpack поддерживает эту функцию.
    filename: 'js/[name].[chunkhash:8].js',
    chunkFilename: 'js/[name].[chunkhash:8].chunk.js',
  },
  module: {
    // .. НЕ ИЗМЕНЯЙТЕ ДРУГИЕ ЗНАЧЕНИЯ, ОБНОВИТЕ ТОЛЬКО СЛЕДУЮЩИЕ ЗНАЧЕНИЯ
    rules: [
      {
        oneOf: [
          // СТРОКА 140
          {
            options: {
              limit: 10000,
              name: 'media/[name].[hash:8].[ext]',
            },
          },
          {
            // СТРОКА 220
            options: {
              name: 'media/[name].[hash:8].[ext]',
            },
          },
        ],
      },
    ],
  },
  plugins: [
    // НЕ ИЗМЕНЯЙТЕ ДРУГИЕ ЗНАЧЕНИЯ
    // СТРОКА 320
    new BundleTracker({path: paths.statsRoot, filename: 'webpack-stats.prod.json'}),
  ],
}
```

Здесь мы назначили `static/bundles/` в качестве `publicPath`, поскольку файлы сборки будут хранится в `assets/bundles` и url `/static/` указывает на каталог `assets`. Мы также убрали префиксы `static`из имён файлов и путей, чтобы предотвратить ненужную вложенность файлов сборки. Это приведёт к тому, что Webpack будет создавать все файлы в каталоге `assets/bundles`, не создавая в нём дополнительный каталог `static`.

После сохранения файла мы можем собрать javascript файлы, используя следующую команду:

```
$ npm run build
```

Эта команда создаст множество файлов в каталоге `assets/bundles` и файл `webpack-stats.prod.json` в корневом каталоге проекта.

Чтобы проверить, что все настройки работают правильно, мы можем запустить Django сервер, используя продакшен настройки, остановив Webpack сервер.

```
$ python manage.py runserver --settings=ponynote.production_settings
```

Если Вы перейдете по адресу [http://localhost:8000](http://localhost:8000), Вы должны увидеть ту же страницу, которая выводилась при запущенном Webpack сервере. Если Вы откроете исходный код страницы, то увидите, что js файлы теперь передаются с помощью Django, а не Webpack.

Существенные замечания

* лучше собирать Ваши js файлы на вашем сервере для непрерывной разработки или вам сервере для развертывания приложения, а не добавлять их в систему контроля версий или исходный код.
* убедитесь, что Вы запускаете `collectstatic` после того как собрали js файлы, поскольку в противном случае Ваш веб-сервер не сможет найти собранные файлы.
* убедитесь, что Ваша сборка генерирует файл `webpack-stats.prod.json`. Если Вы осуществляете сборку файлов из консоли вручную, убедитесь, что Вы также добавили его при копировании файлов с Вашего компьютера на сервер.

Все используемые выше названия каталогов и файлов могут быть любыми другими. Вы можете свободно изменить расположение каталога или названия файлов на те, которые Вам больше нравятся. Но убедитесь, что Вы также изменили эти пути/названия во всех конфигурационных файлах.

На этом я заканчиваю часть, посвященную настройке React и Django для совместной работы. В следующей части мы настроим маршрутизацию в приложении, используя react-router-dom, и управление глобальным состоянием с помощью redux.

## Ссылки

* [Используем Webpack вместе с Django](http://owaislone.org/blog/webpack-plus-reactjs-and-django/)
* [django-webpack-loader](https://github.com/ezhome/django-webpack-loader)
* [create-react-app](https://github.com/facebookincubator/create-react-app)
* [Создаём приложение на React и Django](https://www.fusionbox.com/blog/detail/create-react-app-and-django/624/)