# Современное приложение на Django: Часть 3: Создаём API и интегрируем его с React

В [предыдущей части](http://v1k45.com/blog/modern-django-part-2-redux-and-react-router-setup/) учебного пособия мы разработали фронтэнд для нашего приложения "Записная книжка", который позволяет хранить заметки на стороне клиента, используя Redux store. В этой части мы спроектируем модели базы данных и API для создания, считывания, обновления и удаления заметок в/из базы данных, используя React фронтэнд и Redux store.

Код этого приложения находится на моём github аккаунте [v1k45/ponynote](https://github.com/v1k45/ponynote). Все изменения, произведённые в этой части, находятся в ветке `part-3`.

## Создаём модели базы данных

Чтобы сохранить заметки в базу данных, сначала нам необходимо создать модели. Мы начнём с создания приложения, а затем модели Note внутри него.

В корневом каталоге проекта создайте приложение, используя команду `startapp` и добавьте его в админку.

```
(ponynote)  $ ./manage.py startapp notes
```

Добавьте `notes.apps.NotesConfig` к списку `INSTALLED_APPS` в файле `ponynote/settings.py`.

## Модель Note

Откройте файл `notes/models.py` и создайте следующую модель:

```python
from django.db import models


class Note(models.Model):
    text = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.text
```

Посколльку возможности нашего приложения ограничены, двух полей в модели будет достаточно.

Осуществите миграцию, чтобы добавить эту таблицу в базу данных, используя следующую команду:

```
(ponynote)  $ ./manage.py makemigrations
Migrations for 'notes':
  notes/migrations/0001_initial.py
    - Create model Note

(ponynote)  $ ./manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, notes, sessions
Running migrations:
  Applying notes.0001_initial... OK
```

## Создаём API, используя DRF

Теперь, когда создана модели, мы можем спроектировать API для осуществления CRUD операций с базой данных. Для этого мы воспользуемся `django-rest-framework`.

Установите `django-rest-framework` в виртуальное окружение проекта:

```
(ponynote)  $ pip install djangorestframework
```

Теперь создайте три питоновских файла: `api.py`, `serialiers.py` и `endpoints.py`.

```
$ touch notes/api.py notes/serializers.py notes/endpoints.py
```

Начнём работу с создания сериализатора для нашей модели Notes, добавьте следующий сериализатор в `serializers.py`:

```python
from rest_framework import serializers

from .models import Note


class NoteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Note
        fields = ('id', 'text', )
```

Вышеприведенный сериализатор - это `ModelSerializer`, он имеет API подобное (в некотором роде) классу `ModelForm` в Django.

После добавления сериализатора, создайте API для модели `Note`, используя `NoteSerializer`:

```python
from rest_framework import viewsets, permissions

from .models import Note
from .serializers import NoteSerializer


class NoteViewSet(viewsets.ModelViewSet):
    queryset = Note.objects.all()
    permission_classes = [permissions.AllowAny, ]
    serializer_class = NoteSerializer
```

Работа наборов представлений подобна работе обобщенных представлений для модели в Django представлениях. Пока что разрешим разрешим обрабатывать все виды запросов этой конечной точке.

Давайте зарегистрируем это набор представлений в DRF маршрутизаторе и добавим его в urls Django. Отредактируйте `notes/endpoints.py` следующим образом:

```python
from django.conf.urls import include, url
from rest_framework import routers

from .api import NoteViewSet

router = routers.DefaultRouter()
router.register('notes', NoteViewSet)

urlpatterns = [
    url("^", include(router.urls)),
]
```

В файл `urls.py` проекта `ponynote/urls.py` добавьте:

```python
from django.conf.urls import url, include
from django.views.generic import TemplateView

from notes import endpoints

urlpatterns = [
    url(r'^api/', include(endpoints)),
    url(r'^', TemplateView.as_view(template_name="index.html")),
]
```

После этого Вы сможете осуществлять запросы к API заметок, используя curl или Ваш любимый http клиент, такой как `Postman` или `Insomnia`.

## Тестируем работу API

* Создаём заметку:

```
curl --request POST \
  --url http://localhost:8000/api/notes/ \
  --header 'content-type: application/json' \
  --data '{
    "text": "First Note!"
}'
```

* Получает все заметки:

```
curl --request GET \
  --url http://localhost:8000/api/notes/ \
  --header 'content-type: application/json'

```

* Получаем конкретную заметку, используя её id:

```
curl --request GET \
  --url http://localhost:8000/api/notes/1/ \
  --header 'content-type: application/json'

```

* Удаляем конкретную заметку, используя её id:

```
curl --request DELETE \
  --url http://localhost:8000/api/notes/1/ \
  --header 'content-type: application/json'
```

## Интегрируем DRF API с фронтэндом

Чтобы можно было получать и работать с заметками в бекэнде из фронтэнда, нам нужно добавить в проект несколько библиотек. Чтобы получать заметки, мы будем использовать `whatwg-fetch` (уже добавлено в `create-react-app`) и `redux-thunk` для асинхронного создания action.

## Что такое `redux-thunk`?

Связующее программное обеспечение (middleware) Redux Thunk позволяет Вам писать actions, которые возвращают функцию вместо action. Thunk может использоваться, чтобы задержать отправку action или осуществления отправки только при выполнении определенных условий. Функция внутри Thunk принимает в качестве аргументов методы store и getState.

## Установка и настройка redux-thunk

```
$ npm install redux-thunk --save
```

Затем в файле `App.js` импортируйте функции `thunk` и `applyMiddleware`, передайте их в функцию `createStore`, после чего можно использовать Redux Thunk.

```jsx
import { createStore, applyMiddleware } from "redux";
import thunk from "redux-thunk";

let store = createStore(ponyApp, applyMiddleware(thunk));
```

## Асинхронные Redux actions

Давайте создадим наше первое асинхронное action, используя Redux Thunk. Добавьте следующую функцию в `actions/notes.js`:

```jsx
export const fetchNotes = () => {
  return dispatch => {
    let headers = {"Content-Type": "application/json"};
    return fetch("/api/notes/", {headers, })
      .then(res => res.json())
      .then(notes => {
        return dispatch({
          type: 'FETCH_NOTES',
          notes
        })
      })
  }
}
```

Вышеприведенный код осуществит API вызов Django приложения по адресу `api/notes/` и отправит action `FETCH_NOTES`, когда получит ответ.

Теперь обработаем это action в reducer `reducers/notes.js` добавив оператор case `FETCH_NOTES` в управляющую инструкцию `switch`:

```jsx
case 'FETCH_NOTES':
    return [...state, ...action.notes];
```

Поскольку мы планируем использовать серверную базу данных для заметок, давайте в качестве начального значения для `initialState` будем использовать пустой массив удалив из него фиктивную заметку.

```jsx
const initialState = [];
```

## Используем асинхронное action в компоненте

Использование этого action ничем не отличается от использования обычного action. Просто добавьте его в `mapDispatchToProps`.

```jsx
const mapDispatchToProps = dispatch => {
  return {
    fetchNotes: () => {
      dispatch(notes.fetchNotes());
    },
  }
}
```

Добавьте вызов этого action, когда компонент монтируется, таким образом заметки будут извлекаться из API и загружаться в Redux store. Добавьте метод `componentDidMount` в класс `PonyNote`:

```jsx
componentDidMount() {
    this.props.fetchNotes();
}
```

После перезагрузки страницы Вы должны увидеть список заметок, который Вы создали, используя непосредственно API. Если Вы ещё этого не сделали, давайте подсоединим action `addNote` к API, чтобы мы могли просматривать заметки, полученные непосредственно из базы данных.

## Добавление заметок, используя вызов API

Давайте обновим action `addNote` в файле `actions/notes.js`, чтобы оно могло послать `POST` запрос в API заметок:

```jsx
export const addNote = text => {
  return dispatch => {
    let headers = {"Content-Type": "application/json"};
    let body = JSON.stringify({text, });
    return fetch("/api/notes/", {headers, method: "POST", body})
      .then(res => res.json())
      .then(note => {
        return dispatch({
          type: 'ADD_NOTE',
          note
        })
      })
  }
}
```

В вышеприведенной функции-action наше приложение пошлет `POST` запрос с JSON данными текста заметки и затем отправит `ADD_NOTE` action, которое вставит новую добавленную заметку в Redux store. В файле `reducers/notes.js` обновите оператор case `ADD_NOTE` добавив в него весь объект note вместо только текста.

```jsx
case 'ADD_NOTE':
    return [...state, action.note];
```

После этого немного измените наш компонент `PonyNote.jsx` так, чтобы данные нашей формы сбрасывались после успешного создания заметки.

Добавьте оператор `return` к вызову action, чтобы мы могли добавлять дополнительные функции обратного вызова при вызове API promise.

```jsx
addNote: (text) => {
    return dispatch(notes.addNote(text));
},
```

Обновите метод `submitNote`, переместив вызов `this.resetForm()` из нижней части функции в функцию обратного вызова для `addNote`:

```jsx
this.props.addNote(this.state.text).then(this.resetForm)
```

Подобным образом Вы можете отлавливать (`catch`) любые ошибки, генерируемые promise, обрабатывать ошибки API и показывать их с помощью UI. Чтобы упростить нашу задачу, мы не будем касаться этой темы.

## Обновляем заметки

Action для обновления заметок очень похож на action `addNote`, который тоже вызывает API. Всё что нам нужно сделать - это передать note.id в url конечной точки.

Обновите action `updateNote` в `actions/notes.js`:

```jsx
export const updateNote = (index, text) => {
  return (dispatch, getState) => {

    let headers = {"Content-Type": "application/json"};
    let body = JSON.stringify({text, });
    let noteId = getState().notes[index].id;

    return fetch(`/api/notes/${noteId}/`, {headers, method: "PUT", body})
      .then(res => res.json())
      .then(note => {
        return dispatch({
          type: 'UPDATE_NOTE',
          note,
          index
        })
      })
  }
}
```

Заметьте, что первый аргумент `updateNote` - `index`, а не `id`, это сделано для того, чтобы легко получать заметку, которую нужно обновить. Также мы получаем аргумент `getState` для функции возврата action, он используется, чтобы получить текущее состояние приложения. Мы используем его, чтобы получить `note.id`, используя `index`, который у нас есть.

Ещё одно важный момент заключается в том, что метод, используемый для запроса - `PUT`, который означает, что нужно обновить ресурс на сервере. В последнем action у нас есть index и только что обновленныя заметка, которую мы будем использовать в reducer.

Обновите оператор case `UPDATE_NOTE` в `redcers/notes.js`:

```jsx
case 'UPDATE_NOTE':
    let noteToUpdate = noteList[action.index]
    noteToUpdate.text = action.note.text;
    noteList.splice(action.index, 1, noteToUpdate);
    return noteList;
```

В `PonyNote.jsx`, обновите методы `mapDispatchToProps` and `submitNote`, как мы делали это для добавления заметок.

```jsx
updateNote: (id, text) => {
    return dispatch(notes.updateNote(id, text));
},
```

```jsx
this.props.updateNote(this.state.updateNoteId, this.state.text).then(this.resetForm);
```

Теперь Вы можете обновлять заметки и сохранять их в базе данных.

## Удаление заметок

Обновим action `deleteNote` в файле `actions/notes.js`, чтобы послать запрос `DELETE` API серверу:

```jsx
export const deleteNote = index => {
  return (dispatch, getState) => {

    let headers = {"Content-Type": "application/json"};
    let noteId = getState().notes[index].id;

    return fetch(`/api/notes/${noteId}/`, {headers, method: "DELETE"})
      .then(res => {
        if (res.ok) {
          return dispatch({
            type: 'DELETE_NOTE',
            index
          })
        }
      })
  }
}
```

В reducer `DELETE_NOTE` из файла `reducers/notes.js` внесите следующие изменения:

```jsx
case 'DELETE_NOTE':
    noteList.splice(action.index, 1);
    return noteList;
```

## Резюме

Теперь Вы можете создавать, считывать, обновлять и удалять заметки, используя API, спроектированное используя  django-rest-framework. Все заметки хранятся в Redux store на стороне клиента и изменения будут заносится в базу и сохраняться при перезагрузке.

В следующей части мы добавим аутентификацию в Pony Note, чтобы каждый пользователь мог хранить свои заметки отдельно от заметок других пользователей. Мы реализуем возможность входа в систему/регистрации и свяжем заметки с пользователями.

## Ссылки

* [Django Rest Framework](http://www.django-rest-framework.org/)
* [redux-thunk](https://github.com/gaearon/redux-thunk)
* [Как отправить Redux action с задержкой](https://stackoverflow.com/a/35415559/4726598)
* [Асинхронные Redux actions](https://redux.js.org/advanced/async-actions#async-action-creators)