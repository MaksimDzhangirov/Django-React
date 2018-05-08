# Современное приложение на Django: Часть 2: Настройка Redux и React маршрутизации

Это вторая часть учебного пособия о том как создать "современное" веб приложение или SPA, используя Django и React.js.

В этой части мы настроим Redux и React маршрутизацию для нашего приложения "Записная книжка". Затем мы подсоединим этот фронтэнд к API бэкэнда.

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

```jsx
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

```jsx
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

```jsx
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

Redux - это библиотека для управления глобальным состоянием приложения, основанная на архитектуре [Flux](http://facebook.github.io/flux/) однонаправленного потока данных.

В Redux состоит из трёх основных составляющих: [actions](https://redux.js.org/basics/actions), [reducers](https://redux.js.org/basics/reducers) и [store](https://redux.js.org/basics/store).

![redux](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part2/redux.png)

* Actions - это полезные данные, которые посылаются из Вашего приложения в store Redux. Они являются ~единственным~ источником информации для store.
* Reducers определяют как изменяется состояние приложения в зависимости от отправленных им actions.
* Store хранит дерево состояний всего приложения. Это объект, содержащий методы для получения состояний и отправки actions для изменения состояний.

## Настраиваем Redux

Сначала установите `react-redux`:

```
$ npm install --save redux react-redux
```

После установки создайте каталоги для actions и reducers:

```
$ mkdir actions reducers
```

Создайте пустые файлы action и reducers:

```
$ touch actions/index.js reducers/index.js
$ touch actions/notes.js reducers/notes.js
```

После того как файлы созданы, давайте создадим наш первый reducer! Откройте `reducers/notes.js` и добавьте в него следующий код:

```jsx
const initialState = [
  {text: "Write code!"}
];


export default function notes(state=initialState, action) {
  switch (action.type) {
    default:
      return state;
  }
}
```

`initialState` - это начальное состояние приложения для заметок (кто бы мог бы подумать!). Функция `notes` - это reducer, она принимает в качестве аргументов state и action. Пока что мы определили только одну заметку.

Обратите внимание, что `initialState` может содержать любой допустимый javascript-тип. Для нашего конкретного случая мы может непосредственно использовать Array, но чаще всего состояние определяется как Javascript объект.

Один из наиболее часто встречающихся способов создания reducers - это использование оператора switch, который обрабатывает все возможные actions с помощью метки case. Пока что давайте вернём по умолчанию текущее состояние приложения, не используя операторы case.

После того как reducer создан, нам нужно добавить его в наше приложение с помощью Redux store. Для этого отредактируйте файл `reducers/index.js`:

```jsx
import { combineReducers } from 'redux';
import notes from "./notes";


const ponyApp = combineReducers({
  notes,
})

export default ponyApp;
```

Используя вышеприведенный код мы можем объединить несколько reducers в один. В этом нет необходимости для нашего приложения, но для реальных приложений с большим количеством reducer-ов это удобно.

Теперь давайте создадим redux store, используя этот reducer. В `App.js` создайте store:

```jsx
import { createStore } from "redux";
import ponyApp from "./reducers";

let store = createStore(ponyApp);
```

После создания store нам необходимо обернуть корневой компонент нашего React приложения в компонент `Provider` `react-redux` и передать в него `store`, чтобы он мог его использовать. Окончательная версия App.js будет выглядеть так:

```jsx
import React, { Component } from 'react';
import {Route, Switch, BrowserRouter} from 'react-router-dom';

import { Provider } from "react-redux";
import { createStore } from "redux";
import ponyApp from "./reducers";

import PonyNote from "./components/PonyNote";
import NotFound from "./components/NotFound";

let store = createStore(ponyApp);

class App extends Component {
  render() {
    return (
      <Provider store={store}>
        <BrowserRouter>
          <Switch>
            <Route exact path="/" component={PonyNote} />
            <Route component={NotFound} />
          </Switch>
        </BrowserRouter>
      </Provider>
    );
  }
}

export default App;
```

## Использование Redux

Теперь когда мы закончили настраивать Redux, мы можем использовать наше Redux состояние в компоненте `PonyNote`. Подключите компонент `PonyNote` к Redux store, чтобы отобразить заметки в нашем веб-приложении:

```jsx
import React, { Component } from 'react';
import {connect} from 'react-redux';


class PonyNote extends Component {
  render() {
    return (
      <div>
        <h2>Welcome to PonyNote!</h2>
        <hr />

        <h3>Notes</h3>
        <table>
          <tbody>
            {this.props.notes.map(note => (
              <tr>
                <td>{note.text}</td>
                <td><button>edit</button></td>
                <td><button>delete</button></td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    )
  }
}


const mapStateToProps = state => {
  return {
    notes: state.notes,
  }
}

const mapDispatchToProps = dispatch => {
  return {
  }
}


export default connect(mapStateToProps, mapDispatchToProps)(PonyNote);
```

В вышеприведенном фрагменте кода мы "подключаем" наш компонент, используя высокоуровневую функцию connect, предоставляемую `react-redux`. `mapStateToProps` используется для сопоставления состояния приложения с "props" компонента. Здесь мы передаём массив `notes` в виде свойства компонента, используя то же имя. `mapDispatchToProps` представляем собой набор функций-action для "props" компонента.

Внутри функции `render` мы создали таблицу и прошли по всем заметкам, добавив к каждой кнопку Edit и Delete. Вышеприведенный код должен привести к созданию веб-страницы, выглядящей примерно следующим образом:

![redux-in-action](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part2/redux-in-action.png)

## Работаем с Redux состояниями, используя actions

Пока что у нас есть реализация нашего приложения для заметок, работающая в режиме только чтения, т. е. позволяющая только отображать заметки. Сейчас мы создадим actions, дополнительные операторы case с reducer и UI элементы для изменения Redux состояния.

## Определяем actions и reducers

В `reducers/notes.js` добавьте следующие операторы case внутри инструкции для создания, обновления и удаления заметок:

```jsx
export default function notes(state=initialState, action) {
  let noteList = state.slice();

  switch (action.type) {

    case 'ADD_NOTE':
      return [...state, {text: action.text}];

    case 'UPDATE_NOTE':
      let noteToUpdate = noteList[action.id]
      noteToUpdate.text = action.text;
      noteList.splice(action.id, 1, noteToUpdate);
      return noteList;

    case 'DELETE_NOTE':
      noteList.splice(action.id, 1);
      return noteList;

    default:
      return state;
  }
}
```

В вышеприведенном фрагменте кода мы обрабатываем различные возможные данные, а именно `ADD_NOTE`, `UPDATE_NOTE` and `UPDATE_NOTE`. Для чего нужен каждый из операторов case понятно из названия метки, стоящей рядом с ним, при этом возвращается модифицированная копия состояния после выполнения каждого из названных action. Здесь мы не обновляем состояние непосредственно, а возвращаем новое состояние, которое заменяет текущее состояние reducer-а заметок.

Эти операторы case в reducers будут вызываться только при отправке action соответствующего типа. В `actions/notes.js` объявите следующие actions:

```jsx
export const addNote = text => {
  return {
    type: 'ADD_NOTE',
    text
  }
}

export const updateNote = (id, text) => {
  return {
    type: 'UPDATE_NOTE',
    id,
    text
  }
}

export const deleteNote = id => {
  return {
    type: 'DELETE_NOTE',
    id
  }
}
```

Каждая из приведенных выше функций возвращает объект со свойством `type`, используя которое reducer определяет ~как~ именно нужно обновить состояние. Кроме свойства `type` эти данные могут содержать любое свойство со значением, которое может быть в дальнейшем использовано внутри функции reducer при модификации состояния.

Обновим файл `actions/index.js` так, чтобы мы имели доступ ко всем actions в одном файле:

```jsx
import * as notes from "./notes";

export {notes}
```

## Используем Actions внутри компонента

После того как определены actions, мы можем использовать их внутри компонента `PonyNote`, объявив свойства в функции `mapDispatchToProps`.

Обновите функцию `mapDispatchToProps`, чтобы в ней использовались следующие действия:

```jsx
import {notes} from "../actions";

const mapDispatchToProps = dispatch => {
  return {
    addNote: (text) => {
      dispatch(notes.addNote(text));
    },
    updateNote: (id, text) => {
      dispatch(notes.addNote(id, text));
    },
    deleteNote: (id) => {
      dispatch(notes.deleteNote(id));
    },
  }
}
```

Теперь все эти actions доступны внутри компонента через `this.props`.

## Создаём UI элементы для actions

Давайте начнём с создания формы для добавления новых заметок к состоянию notes Redux. В компоненте `PonyNote` объявите метод `state` и `submitNote`.

```jsx
state = {
  text: ""
}

submitNote = (e) => {
  e.preventDefault();
  this.props.addNote(this.state.text);
  this.setState({text: ""});
}
```

Внутри компонента добавьте HTML форму для ввода текста и сохранения заметки:

```jsx
<h3>Add new note</h3>
<form onSubmit={this.submitNote}>
  <input
    value={this.state.text}
    placeholder="Enter note here..."
    onChange={(e) => this.setState({text: e.target.value})}
    required />
  <input type="submit" value="Save Note" />
</form>
```

Вышеприведенный код сохранит текст записки в состоянии компонента при отправке формы. При вызове `onSubmit` приложение отправит action `ADD_NOTE`, которое в последствии добавит заметку к Redux состоянию.

Подобным образом мы можем добавить возможность удалять заметки при нажатии кнопки "Delete". Замените содержимое `tbody` на следующее:

```jsx
{this.props.notes.map((note, id) => (
  <tr key={`note_${id}`}>
    <td>{note.text}</td>
    <td><button>edit</button></td>
    <td><button onClick={() => this.props.deleteNote(id)}>delete</button></td>
  </tr>
))}
```

Для того чтобы можно было обновлять заметки нам нужно ввести некоторые дополнительные изменения в наше состояние компонента и элемент формы, чтобы оно поддерживало одновременно создание и обновление заметок.

```jsx
state = {
  text: "",
  updateNoteId: null,
}

resetForm = () => {
  this.setState({text: "", updateNoteId: null});
}

selectForEdit = (id) => {
  let note = this.props.notes[id];
  this.setState({text: note.text, updateNoteId: id});
}

submitNote = (e) => {
  e.preventDefault();
  if (this.state.updateNoteId === null) {
    this.props.addNote(this.state.text);
  } else {
    this.props.updateNote(this.state.updateNoteId, this.state.text);
  }
  this.resetForm();
}
```

Состояние компонента теперь может определить создаём или обновляем ли мы заметку, а метод `submitNote` изменяет своё поведение в зависимости от состояния компонента. Мы также добавили вспомогательный метод для загрузки данных в форму и сброса данных формы при обновлении заметки.

Внутри элемента формы добавьте кнопку для сброса данных формы, если заметка, которую нужно отредактировать, была выбрана по ошибке.

```jsx
<button onClick={this.resetForm}>Reset</button>
```

Обновите код внутри кнопки Edit, чтобы при нажатии на неё загружались данные заметки, выбранной для редактирования:

```jsx
<td><button onClick={() => this.selectForEdit(id)}>edit</button></td>
```

После этого мы получил простое веб-приложение для заметок, позволяющее добавлять, редактировать и удалять заметки. Оно должно выглядеть примерно следующим образом:

[!redux-working-webapp](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part2/redux-working-webapp.png)

## Добавляем немного css

Перед тем как продолжить, давайте добавим немного css стилей, чтобы улучшить внешний вид нашего приложения. Для этого мы будем использовать [sakura.css](https://github.com/oxalorg/sakura), ту же библиотеку, не использующую классы, которая применялась при создании этого блога!

Начните с загрузки `normalize.css` и `sakura.css` в каталог `css` корневой папки фронтэнда.

```
$ mkdir css
$ cd css
$ wget https://raw.githubusercontent.com/oxalorg/sakura/master/css/normalize.css
$ wget wget https://raw.githubusercontent.com/oxalorg/sakura/master/css/sakura.css
```

После этого добавьте в файл `index.css` следующие строки для импорта этих css файлов:

```css
@import url("css/normalize.css");
@import url("css/sakura.css");
```

Теперь наше приложении выглядит намного лучше!

[!sakura](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part2/sakura.png)

Поскольку мы спроектировали работоспособную клиентскую часть нашего веб-приложения, но она не сохраняет введённые данные, в [следующей части](http://v1k45.com/blog/modern-django-part-3-creating-an-api-and-integrating-with-react/) мы создадим модели и API в Django для хранения и обработки заметок, используя базу данных, чтобы полученные от пользователя данные не терялись.

Ссылки

* [Пример реализации списка задач с помощью Redux](https://redux.js.org/basics/example-todo-list)
* [Документация react-router-dom](https://reacttraining.com/react-router/web/guides/philosophy)
* [Блок-схема потока данных в Redux](https://medium.com/@abhayg772/introduction-to-redux-using-react-native-a8f1e8778333)