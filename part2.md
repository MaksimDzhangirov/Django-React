# Современное приложение на Django: Часть 2: Настройка Redux и React маршрутизации

Это вторая часть учебного пособия о том как создать "современное" веб приложение или SPA, используя Django и React.js.

В этой части мы настроим Redux и React маршрутизацию для нашего приложения "Записная книжка". Затем мы этот фронтэнд к API бэкэнда.

Код этого приложения находится на моём github аккаунте [v1k45/ponynote](https://github.com/v1k45/ponynote). Все изменения, произведённые в этой части, находятся в ветке `part-2`.

## Что такое React маршрутизация?

[React-router-dom](https://reacttraining.com/react-router/web) - это библиотека, которая используется для маршрутизации внутри React приложения. С её помощью Вы можете монтировать компоненты по Вашему усмотрению в зависимости от введенного url.

Сначала мы создадим маршрутизатор для нашего приложения с необходимыми нам путями. Поскольку это базовое приложение для хранения записей (заметок), одного компонента/маршрута должно быть достаточно, но мы всё равно создадим несколько других компонентов и маршрутов для демонстрационных целей.

## Настраиваем react-router-dom

Начнём с установки react-router-dom:

```
$ cd ponynote/frontend
$ npm install --save react-router-dom
```

После окончания установки давайте создадим несколько компонентов, которые мы будем использовать в приложении. Чтобы не нарушать уже созданную структуру каталогов в приложении, мы будем создавать все компоненты внутри каталога `components`. Мы создадим компонент `PonyNote` для нашего основного приложения и NotFound компонент для страницы 404.

```
$ cd src/
$ mkdir components/
$ touch components/index.js
$ touch components/PonyNote.jsx
$ touch components/NotFound.jsx
```

Теперь после создания всех этих файлов, обновите файл `App.js` добавив в его `react-router-dom`:

```javascript
import React, { Component } from 'react';
import {Route, Switch, BrowserRouter} from 'react-router-dom';
import PonyNote from "./components/PonyNote";
import NotFound from "./components/NotFound";

class App extends Component {
  render() {
  return (
    <BrowserRouter>
    <Switch>
      <Route exact path="/" component={PonyNote} />
      <Route component={NotFound} />
    </Switch>
    </BrowserRouter>
  );
  }
}

export default App;
```

В вышеприведенном фрагменте кода мы импортируем `[BrowserRouter](https://reacttraining.com/react-router/web/api/BrowserRouter)`, который использует API HTML5 истории для осуществления маршрутизации приложения. Компонент `[Switch](https://reacttraining.com/react-router/web/api/Switch)` является не обязательным, но используется для более эффективной маршрутизации. Компоненты `[Route](https://reacttraining.com/react-router/web/api/Route)` позволяют отображать конкретный компонент, когда текущее местоположение приложения соответствует определенному пути. Если ни один из путей не указан для Route, то любой путь возвращает true, что можно использовать для отображения страницы 404.

Теперь, когда создан компонент App, мы можем обновить содержимое файлов для других компонентов, чтобы они выводили то, для чего предназначались.

В файл `PonyNote.jsx` добавьте:

```javascript
import React, { Component } from 'react';
import {Link} from 'react-router-dom';


export default class PonyNote extends Component {
  render() {
  return (
    <div>
    <h2>Welcome to PonyNote!</h2>
    <p>
      <Link to="/contact">Click Here</Link> to contact us!
    </p>
    </div>
  )
  }
}
```

Этот компонент отображает сообщение с приветствием и ссылку на страницу контактов (которая не существует), поэтому вместо неё отобразится компонент NotFound.

В файл `NotFound.jsx` добавьте:

```javascript
import React from 'react';


const NotFound = () => {
  return (
  <div>
    <h2>Not Found</h2>
    <p>The page you're looking for does not exists.</p>
  </div>
  )
}

export default NotFound
```

После этого запустите сервера для разработки Django и Webpack:

```
(ponynote) $ ./manage.py runserver
$ cd frontend && npm run start
```

Вы увидите следующую страницу в Вашем браузере:

![index-page](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part2/index-page.png)

После нажатия на ссылку, перебрасывающую на страницу контактов, должна отобразиться 404 страница:

![404-page](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part2/404-page.png)

Также Вы можете вернуться на предыдущую страницу, используя кнопку "Назад" в Вашем браузере.

## Что такое Redux?