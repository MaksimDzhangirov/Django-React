# Современное приложение на Django: Часть 4: Добавляем аутентификацию к React SPA, используя DRF

В [предыдущей части](http://v1k45.com/blog/modern-django-part-3-creating-an-api-and-integrating-with-react/) учебного пособия мы разработали бэкэнд для создания/считывания/обновления/удаления заметок непосредственно из базы данных, используя API, созданный с помощью Django Rest фреймворка, взаимодействующий с фронтэндом на React. В этой части мы позволим пользователям иметь свои отдельные заметки и защитим их, используя аутентификацию.

Код этого приложения находится на моём github аккаунте [v1k45/ponynote](https://github.com/v1k45/ponynote). Все изменения, произведённые в этой части, находятся в ветке `part-4`.

## Связываем заметки с пользователями

Чтобы у каждого пользователя были свои заметки, нам нужно связать заметки с пользователями. Мы начнём с добавления поля `owner` к модели `Note`. Обновите `notes/models.py`:

```python
from django.db import models
from django.contrib.auth.models import User


class Note(models.Model):
    text = models.CharField(max_length=255)
    owner = models.ForeignKey(User, related_name="notes",
                              on_delete=models.CASCADE, null=True)
    created_at = models.DateTimeField(auto_now_add=True)

```
Осуществляем миграции и переносим изменения в базу данных.
```
(ponynote)  $ ./manage.py makemigrations
Migrations for 'notes':
  notes/migrations/0002_note_owner.py
    - Add field owner to note
(ponynote)  $ ./manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, notes, sessions
Running migrations:
  Applying notes.0002_note_owner... OK
```

## Создаём API для аутентификации

Нам нужно создать API для основных действий при аутентификации, таких как регистрация, конечная точка, позволяющая получать данные о пользователе, вход и выход из системы.

Django Rest фреймворк позволяет использовать различные методы аутентификации, включая BasicAuth, SessionAuth, TokenAuth. Для одностраничных приложений довольно часто используется аутентификация с использованием токенов и их разновидность - JSON Web Tokens (JWT).

## Устанавливаем и настраиваем `knox`

В состав DRF встроена возможность аутентификации с использованием токенов, но она не идеальна для пользователей, использующих её в одностраничных приложениях из-за отсутствия определенных базовых функций. Вместо неё мы будем использовать `django-rest-knox`, он похож на TokenAuth от DRF, но намного лучше и надежнее.

Для начала установим его:

```
(ponynote)  $ pip install django-rest-knox
```

Обновим `ponynote/settings.py` добавив `knox` и `rest_framework` к INSTALLED_APPS и установив класс TokenAuthentication из knox в качестве используемого по умолчанию в DRF.

```python
INSTALLED_APPS = [
    'rest_framework',
    'knox',
]

# Rest framework settings
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': ('knox.auth.TokenAuthentication',),
}
```

Добавим `knox` маршруты в `ponynote/urls.py`:

```python
urlpatterns = [
    url(r'^api/', include(endpoints)),
    url(r'^api/auth/', include('knox.urls')),
    url(r'^', TemplateView.as_view(template_name="index.html")),
]
```

Наконец, выполним миграции для базы данных:

```
(ponynote)  $ ./manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, knox, notes, sessions
Running migrations:
  Applying knox.0001_initial... OK
  Applying knox.0002_auto_20150916_1425... OK
  Applying knox.0003_auto_20150916_1526... OK
  Applying knox.0004_authtoken_expires... OK
  Applying knox.0005_authtoken_token_key... OK
  Applying knox.0006_auto_20160818_0932... OK
```

## Создаём API для регистрации

Чтобы позволить пользователям создавать аккаунты, мы создадим API для регистрации. В идеале Вы должны использовать многофункциональные сторонние приложения, такие как `allauth`, `rest-auth`, `djoser` и т.д. для обработки различных способов аутентификации. Но поскольку мы разрабатываем простое приложение, мы создадим свои собственные представления/конечные точки.

Начнём с создания `CreateUserSerializer` и `UserSerializer` в `notes/serializers.py`:

```python
from rest_framework import serializers
from django.contrib.auth.models import User

class CreateUserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('id', 'username', 'password')
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User.objects.create_user(validated_data['username'],
                                        None,
                                        validated_data['password'])
        return user


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ('id', 'username')
```

Мы будем использовать `CreateUserSerializer` для проверки данных, введенных при регистрации. Мы не будем использовать `email`, пользователи будут входить в систему, используя имя пользователя и пароль. `UserSerializer` будет использоваться для выдачи информации о пользователе, если его регистрация прошла успешно.

Теперь создайте API для регистрации. В `notes/api.py`:

```python
from rest_framework import viewsets, permissions, generics
from rest_framework.response import Response

from knox.models import AuthToken

from .models import Note
from .serializers import NoteSerializer, CreateUserSerializer, UserSerializer


class RegistrationAPI(generics.GenericAPIView):
    serializer_class = CreateUserSerializer

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.save()
        return Response({
            "user": UserSerializer(user, context=self.get_serializer_context()).data,
            "token": AuthToken.objects.create(user)
        })
```

API довольно простое - мы проверяем данные, введенные пользователем, и создаём аккаунт, если проверка прошла успешно. В качестве ответа мы возвращаем объект с данными пользователя в сериализованном виде и токен аутентификации, который будет использоваться приложением для выполнения пользовательских запросов к API.

Обновите конечные точки API, добавив API для регистрации. В `notes/endpoints.py`:

```python
from .api import NoteViewSet, RegistrationAPI

urlpatterns = [
    url("^", include(router.urls)),
    url("^auth/register/$", RegistrationAPI.as_view()),
]
```

После добавления конечной точки, Вы можете протестировать конечную точку, используя curl:

```
$ curl --request POST \
--url http://localhost:8000/api/auth/register/ \
--header 'content-type: application/json' \
--data '{
  "username": "user1",
  "password": "hunter2"
}'
```

Такой запрос создаст пользователя с именем `user1` и паролем `hunter2`. Вы получите следующий ответ от API:

```
{"user":{"id":1,"username":"user1"},"token":"<TOKEN HERE>"}
```

При выполнении запроса происходит проверка полей, включая проверку на уникальность имени пользователя. Если Вы попытаетесь отправить одни и те же данные дважды, то увидите, что API выдает ошибку, сообщающую о необходимости уникальности имени пользователя.

## Создаём API для входа в систему

Теперь когда пользователи могут создать учётную запись, нам нужно API позволяющее пользователям входить в приложение и получать токен аутентификации для выполнения пользовательских действий.

Сначала создадим `LoginUserSerializer` в `notes/serializers.py`:

```python
from django.contrib.auth import authenticate


class LoginUserSerializer(serializers.Serializer):
    username = serializers.CharField()
    password = serializers.CharField()

    def validate(self, data):
        user = authenticate(**data)
        if user and user.is_active:
            return user
        raise serializers.ValidationError("Unable to log in with provided credentials.")
```

Метод `validate` этого сериализатора проверяет правильность комбинации имени пользователя и пароля, используя функцию `authenticate` Django. Он также гарантирует, что пользователь активен.

затем создайте `LoginAPI` в `notes/api.py`:

```python
from .serializers import (NoteSerializer, CreateUserSerializer,
                          UserSerializer, LoginUserSerializer)


class LoginAPI(generics.GenericAPIView):
    serializer_class = LoginUserSerializer

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.validated_data
        return Response({
            "user": UserSerializer(user, context=self.get_serializer_context()).data,
            "token": AuthToken.objects.create(user)
        })
```

После этого добавьте API к списку конечных точек API в `notes/endpoints.py`:

```python
from .api import NoteViewSet, RegistrationAPI, LoginAPI

urlpatterns = [
    url("^", include(router.urls)),
    url("^auth/register/$", RegistrationAPI.as_view()),
    url("^auth/login/$", LoginAPI.as_view()),
]
```

Это позволит использовать наше API для входа в систему через конечные точки. Мы можем протестировать его с помощью curl:

```
$ curl --request POST \
  --url http://localhost:8000/api/auth/login/ \
  --header 'content-type: application/json' \
  --data '{
    "username": "user1",
    "password": "hunter2"
}'
```

## Конечная точка для получения данных пользователя

Теперь, когда API для регистрации и входа в систему работает, нам необходимо API, возвращающее данные залогиненного пользователя. Мы будем использовать это API, чтобы определить залогинен ли пользователь и получения его токена для осуществления вызовов API, связанных с конкретным пользователем.

Поскольку у нас уже есть `UserSerializer`, мы можем сразу создать `UserAPI` и добавить его в конечные точки.

Вставьте следующий код в `notes/api.py`:

```python
class UserAPI(generics.RetrieveAPIView):
    permission_classes = [permissions.IsAuthenticated, ]
    serializer_class = UserSerializer

    def get_object(self):
        return self.request.user
```

Вышеприведенное API возвратит данные пользователя для аутентифицированного пользователя либо ошибку из диапазона 4ХХ, если  пользователь не аутентифицирован или использовался неправильный токен.

Добавьте API в `notes/endpoints.py`:

```python
from .api import NoteViewSet, RegistrationAPI, LoginAPI, UserAPI

urlpatterns = [
    url("^", include(router.urls)),
    url("^auth/register/$", RegistrationAPI.as_view()),
    url("^auth/login/$", LoginAPI.as_view()),
    url("^auth/user/$", UserAPI.as_view()),
]
```

Вы можете протестировать API, используя токен аутентификации, который получили из `LoginAPI`:

```
$ curl --request GET \
  --url http://localhost:8000/api/auth/user/ \
  --header 'authorization: Token YOUR_API_TOKEN_HERE' \
  --header 'content-type: application/json' \
```

Вышеприведенный запрос возвратит объект с данными аутентифицированного пользователя.

## Разрешаем доступ к NotesAPI только аутентифицированным пользователям

Теперь, когда все API, связанные с аутентификацией работают, мы можем обновить `NoteViewSet` и `NoteSerializer`, чтобы ограничить разрешить доступ к ним только для аутентифицированных пользователей.

Начнём с обновления `NoteViewSet` в `notes/api.py`:

```python
class NoteViewSet(viewsets.ModelViewSet):
    permission_classes = [permissions.IsAuthenticated, ]
    serializer_class = NoteSerializer

    def get_queryset(self):
        return self.request.user.notes.all()

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

Мы осуществили следующие изменения в нём:

* Изменили доступ к API с `AllowAny` на `IsAuthenticated`. Это потребует от пользователей войти в систему/послать токен аутентификации, чтобы воспользоваться этим API.
* Переопределили метод `perform_create`, чтобы сохранить владельца заметки при её создании.
* Заменили атрибут `queryset` на метод `get_queryset`, который будет возвращать все заметки, которыми владеет аутентифицированный пользователь.

После этого обновить маршрутизатор для регистрации заметок, добавив `base_name`. Он необходим в случае, когда набор представлений не содержит атрибута `queryset`:

```
router.register('notes', NoteViewSet, 'notes')
```

Это гарантирует, что API для заметок будет доступно только для аутентифицированных пользователей и пользователи смогут видеть и изменять только свои собственные заметки.

## Добавляем аутентификацию в React приложение

Блок-схема аутентификации в React приложении  будет очень простой, мы перенаправляем пользователя на страницу входа в систему, если он не залогинен и перенаправляем его на страницу с заметками после успешного входа в систему.

![auth-react](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part4/auth-react.png)

## Страница для входа в систему

Начнём с создания пока не рабочей страницы для входа в систему, на которую мы будем перенаправлять неаутентифицированных пользователей.

Создайте `frontend/src/components/Login.jsx`:

```jsx
import React, {Component} from "react";
import {connect} from "react-redux";

import {Link} from "react-router-dom";


class Login extends Component {

  state = {
    username: "",
    password: "",
  }

  onSubmit = e => {
    e.preventDefault();
    console.error("Not implemented!!1");
  }

  render() {
    return (
      <form onSubmit={this.onSubmit}>
        <fieldset>
          <legend>Login</legend>
          <p>
            <label htmlFor="username">Username</label>
            <input
              type="text" id="username"
              onChange={e => this.setState({username: e.target.value})} />
          </p>
          <p>
            <label htmlFor="password">Password</label>
            <input
              type="password" id="password"
              onChange={e => this.setState({password: e.target.value})} />
          </p>
          <p>
            <button type="submit">Login</button>
          </p>

          <p>
            Don't have an account? <Link to="/register">Register</Link>
          </p>
        </fieldset>
      </form>
    )
  }
}

const mapStateToProps = state => {
  return {};
}

const mapDispatchToProps = dispatch => {
  return {};
}

export default connect(mapStateToProps, mapDispatchToProps)(Login);
```

Затем добавьте маршрут для входа в систему в App.js:

```jsx
import Login from "./components/Login";

class App extends Component {
  render() {
    return (
      <Provider store={store}>
        <BrowserRouter>
          <Switch>
            <Route exact path="/" component={PonyNote} />
            <Route exact path="/login" component={Login} />
            <Route component={NotFound} />
          </Switch>
        </BrowserRouter>
      </Provider>
    );
  }
}

export default App;
```

Затем перейдите по адресу [localhost:8000/login](http://localhost:8000/login), чтобы увидеть страницу для входа в систему:

![login](https://github.com/MaksimDzhangirov/Django-React/raw/master/img/part4/login.png)

## Actions и reducers для аутентификации

Чтобы страница входа в систему заработала и для разрешения доступа к странице с заметками только аутентифицированным пользователям, нам нужно добавить несколько actions и аутентификационный reducer. 

## Аутентификационный reducer

Начните с создания файла `auth.js` в `frontend/src/reducers/` и добавьте код reducer:

```jsx
const initialState = {
  token: localStorage.getItem("token"),
  isAuthenticated: null,
  isLoading: true,
  user: null,
  errors: {},
};


export default function auth(state=initialState, action) {

  switch (action.type) {

    case 'USER_LOADING':
      return {...state, isLoading: true};

    case 'USER_LOADED':
      return {...state, isAuthenticated: true, isLoading: false, user: action.user};

    case 'LOGIN_SUCCESSFUL':
      localStorage.setItem("token", action.data.token);
      return {...state, ...action.data, isAuthenticated: true, isLoading: false, errors: null};

    case 'AUTHENTICATION_ERROR':
    case 'LOGIN_FAILED':
    case 'LOGOUT_SUCCESSFUL':
      localStorage.removeItem("token");
      return {...state, errors: action.data, token: null, user: null,
        isAuthenticated: false, isLoading: false};

    default:
      return state;
  }
}
```

Заметили, что значение токена было получено из `localStorage`? Мы будем хранить токен аутентификации в localStorage и затем загружать его в `auth` reducer при начальной загрузке. Это поможет нам хранить аутентификационное состояние пользователя даже если он закроет окно браузера.

Мы работаем с пользовательскими actions загрузки и входа в систему. Если вход в систему выполнен успешно, reducer обновит localStorage и redux токен. Если вход в систему завершается с ошибкой, пользователь выходит из системы или приложение генерирует ошибки, связанные с аутентификацией, reducer удаляет токен аутентификации, сохраняет ошибки и устанавливает соответствующее состояние входа в систему.

Добавьте этот reducer в `reducers/index.js`:

```jsx
import auth from "./auth";

const ponyApp = combineReducers({
  notes, auth,
})
```

## Actions, связанные с аутентификацией

Создайте `frontend/src/actions/auth.js` и добавьте action `loadUser` в него:

```jsx
export const loadUser = () => {
  return (dispatch, getState) => {
    dispatch({type: "USER_LOADING"});

    const token = getState().auth.token;

    let headers = {
      "Content-Type": "application/json",
    };

    if (token) {
      headers["Authorization"] = `Token ${token}`;
    }
    return fetch("/api/auth/user/", {headers, })
      .then(res => {
        if (res.status < 500) {
          return res.json().then(data => {
            return {status: res.status, data};
          })
        } else {
          console.log("Server Error!");
          throw res;
        }
      })
      .then(res => {
        if (res.status === 200) {
          dispatch({type: 'USER_LOADED', user: res.data });
          return res.data;
        } else if (res.status >= 400 && res.status < 500) {
          dispatch({type: "AUTHENTICATION_ERROR", data: res.data});
          throw res.data;
        }
      })
  }
}
```

В вышеприведенном action мы посылаем в заголовке Authorization токен, хранящийся в redux store. Если этот токен существует и верный, то API вернет объект с данными пользователя, в противном случае, мы вызываем action `AUTHENTICATION_ERROR`.

Обновите `frontend/src/actions/index.js`, добавив `auth.js`:

```jsx
import * as notes from "./notes";
import * as auth from "./auth";

export {notes, auth}
```

## Ограничиваем доступ для неавторизованных пользователей

Чтобы неаутентифицированные пользователи не могли получить доступ к страницам, которые доступны только залогиченным пользователям, мы должны перенаправлять такие запросы на страницу входа в систему.

Для этого мы создадим метод `PrivateRoute`, который отображает указанный компонент только в том случае, если пользователь аутентифицирован.

Обновите файл `frontend/src/App.js`:

```jsx
import React, { Component } from 'react';
import {Route, Switch, BrowserRouter, Redirect} from 'react-router-dom';

import { Provider, connect } from "react-redux";
import { createStore, applyMiddleware } from "redux";
import thunk from "redux-thunk";

import {auth} from "./actions";
import ponyApp from "./reducers";

import PonyNote from "./components/PonyNote";
import NotFound from "./components/NotFound";
import Login from "./components/Login";

let store = createStore(ponyApp, applyMiddleware(thunk));

class RootContainerComponent extends Component {

  componentDidMount() {
    this.props.loadUser();
  }

  PrivateRoute = ({component: ChildComponent, ...rest}) => {
    return <Route {...rest} render={props => {
      if (this.props.auth.isLoading) {
        return <em>Loading...</em>;
      } else if (!this.props.auth.isAuthenticated) {
        return <Redirect to="/login" />;
      } else {
        return <ChildComponent {...props} />
      }
    }} />
  }

  render() {
    let {PrivateRoute} = this;
    return (
      <BrowserRouter>
        <Switch>
          <PrivateRoute exact path="/" component={PonyNote} />
          <Route exact path="/login" component={Login} />
          <Route component={NotFound} />
        </Switch>
      </BrowserRouter>
    );
  }
}

const mapStateToProps = state => {
  return {
    auth: state.auth,
  }
}

const mapDispatchToProps = dispatch => {
  return {
    loadUser: () => {
      return dispatch(auth.loadUser());
    }
  }
}

let RootContainer = connect(mapStateToProps, mapDispatchToProps)(RootContainerComponent);

export default class App extends Component {
  render() {
    return (
      <Provider store={store}>
        <RootContainer />
      </Provider>
    )
  }
}
```

В вышеприведенном коде, мы переместили код после `Provider` в отдельный компонент под названием `RootContainerComponent`.

Как следует из названия `RootContainerComponent` - это корневой контейнер приложения и он подсоединен к redux store. Он содержит метод `PrivateRoute`, который выводит разное содержимое на страницу при совпадении маршрута в зависимости от состояния `isAuthenticated` приложения. Если пользователь не вошел в систему, то его перенаправляет на страницу `/login`.

`RootContainer` затем используется внутри компонента `App` и находится внутри компонента `Provider`.

После этого, если вы захотите перейти на страницу с заметками ([http://localhost:8000/](http://localhost:8000/)), Вас перенаправит на страницу входа в систему.

## Создание страницы входа в систему

Мы уже защитили наше приложение от несанкционированный доступ к странице заметок, но у нас по-прежнему нет функции входа в систему, позволяющей аутентифицированным пользователям получить доступ к странице.

Для этого создайте функцию `login` в файле `frontend/src/actions/auth.js`:

```jsx
export const login = (username, password) => {
  return (dispatch, getState) => {
    let headers = {"Content-Type": "application/json"};
    let body = JSON.stringify({username, password});

    return fetch("/api/auth/login/", {headers, body, method: "POST"})
      .then(res => {
        if (res.status < 500) {
          return res.json().then(data => {
            return {status: res.status, data};
          })
        } else {
          console.log("Server Error!");
          throw res;
        }
      })
      .then(res => {
        if (res.status === 200) {
          dispatch({type: 'LOGIN_SUCCESSFUL', data: res.data });
          return res.data;
        } else if (res.status === 403 || res.status === 401) {
          dispatch({type: "AUTHENTICATION_ERROR", data: res.data});
          throw res.data;
        } else {
          dispatch({type: "LOGIN_FAILED", data: res.data});
          throw res.data;
        }
      })
  }
}
```

После этого отредактируйте файл `frontend/src/components/Login.jsx`, чтобы можно было использовать только что созданное action:

```jsx
import {Link, Redirect} from "react-router-dom";

import {auth} from "../actions";

class Login extends Component {

  onSubmit = e => {
    e.preventDefault();
    this.props.login(this.state.username, this.state.password);
  }

  render() {
    if (this.props.isAuthenticated) {
      return <Redirect to="/" />
    }
    return (
      <form onSubmit={this.onSubmit}>
        <fieldset>
          <legend>Login</legend>
          {this.props.errors.length > 0 && (
            <ul>
              {this.props.errors.map(error => (
                <li key={error.field}>{error.message}</li>
              ))}
            </ul>
          )}
          {/*KEEP THE OTHER ELEMENTS*/}
        </fieldset>
      </form>
    )
  }
}

const mapStateToProps = state => {
  let errors = [];
  if (state.auth.errors) {
    errors = Object.keys(state.auth.errors).map(field => {
      return {field, message: state.auth.errors[field]};
    });
  }
  return {
    errors,
    isAuthenticated: state.auth.isAuthenticated
  };
}

const mapDispatchToProps = dispatch => {
  return {
    login: (username, password) => {
      return dispatch(auth.login(username, password));
    }
  };
}

export default connect(mapStateToProps, mapDispatchToProps)(Login);
```

После этого вы сможете войти в систему со страницы входа и перейти на страницу заметок, если Вы ввели правильные учетные данные.

## Страница регистрации

До сих пор для создания пользователей мы использовали непосредственно API регистрации. Поскольку у нас есть все actions и reducers, мы также можем создать страницу регистрации.

Начните с создания файла `frontend/src/components/Register.jsx` и добавьте в него следующий код:

```jsx
import React, {Component} from "react";
import {connect} from "react-redux";

import {Link, Redirect} from "react-router-dom";

import {auth} from "../actions";

class Login extends Component {

  state = {
    username: "",
    password: "",
  }

  onSubmit = e => {
    e.preventDefault();
    this.props.register(this.state.username, this.state.password);
  }

  render() {
    if (this.props.isAuthenticated) {
      return <Redirect to="/" />
    }
    return (
      <form onSubmit={this.onSubmit}>
        <fieldset>
          <legend>Register</legend>
          {this.props.errors.length > 0 && (
            <ul>
              {this.props.errors.map(error => (
                <li key={error.field}>{error.message}</li>
              ))}
            </ul>
          )}
          <p>
            <label htmlFor="username">Username</label>
            <input
              type="text" id="username"
              onChange={e => this.setState({username: e.target.value})} />
          </p>
          <p>
            <label htmlFor="password">Password</label>
            <input
              type="password" id="password"
              onChange={e => this.setState({password: e.target.value})} />
          </p>
          <p>
            <button type="submit">Register</button>
          </p>

          <p>
            Already have an account? <Link to="/login">Login</Link>
          </p>
        </fieldset>
      </form>
    )
  }
}

const mapStateToProps = state => {
  let errors = [];
  if (state.auth.errors) {
    errors = Object.keys(state.auth.errors).map(field => {
      return {field, message: state.auth.errors[field]};
    });
  }
  return {
    errors,
    isAuthenticated: state.auth.isAuthenticated
  };
}

const mapDispatchToProps = dispatch => {
  return {
    register: (username, password) => dispatch(auth.register(username, password)),
  };
}

export default connect(mapStateToProps, mapDispatchToProps)(Login);
```

Как видно из кода, компонент очень похож на Login.jsx, единственным заметным отличием является использование функции/action `register` вместо `login` и изменения, связанные с версткой.

Добавьте action `register` в `frontend/src/actions/auth.js`:

```jsx
export const register = (username, password) => {
  return (dispatch, getState) => {
    let headers = {"Content-Type": "application/json"};
    let body = JSON.stringify({username, password});

    return fetch("/api/auth/register/", {headers, body, method: "POST"})
      .then(res => {
        if (res.status < 500) {
          return res.json().then(data => {
            return {status: res.status, data};
          })
        } else {
          console.log("Server Error!");
          throw res;
        }
      })
      .then(res => {
        if (res.status === 200) {
          dispatch({type: 'REGISTRATION_SUCCESSFUL', data: res.data });
          return res.data;
        } else if (res.status === 403 || res.status === 401) {
          dispatch({type: "AUTHENTICATION_ERROR", data: res.data});
          throw res.data;
        } else {
          dispatch({type: "REGISTRATION_FAILED", data: res.data});
          throw res.data;
        }
      })
  }
}
```

Обновите reducers для аутентификации, чтобы обрабатывать actions, связанные с регистрацией. Обновите файл `frontend/src/reducers/auth.js`:

```jsx
case 'LOGIN_SUCCESSFUL':
case 'REGISTRATION_SUCCESSFUL':
    localStorage.setItem("token", action.data.token);
    return {...state, ...action.data, isAuthenticated: true, isLoading: false, errors: null};

case 'AUTHENTICATION_ERROR':
case 'LOGIN_FAILED':
case 'REGISTRATION_FAILED':
case 'LOGOUT_SUCCESSFUL':
    localStorage.removeItem("token");
    return {...state, errors: action.data, token: null, user: null,
        isAuthenticated: false, isLoading: false};
```

Обновите операторы case, добавив action `REGISTRATION_SUCCESSFUL` и `REGISTRATION_FAILED`.

Добавьте компонент `Register` в список маршрутов маршрутизатора React внутри `RootContainerComponent` файла `App.js`:

```jsx
import Register from "./components/Register";

class RootContainerComponent extends Component {

  render() {
    let {PrivateRoute} = this;
    return (
      <BrowserRouter>
        <Switch>
          <PrivateRoute exact path="/" component={PonyNote} />
          <Route exact path="/register" component={Register} />
          <Route exact path="/login" component={Login} />
          <Route component={NotFound} />
        </Switch>
      </BrowserRouter>
    );
  }
}
```

Теперь вы сможете зарегистрироваться, перейдя на [localhost:8000/register](http://localhost:8000/register). Вы также увидите ошибки на странице, если попытаетесь зарегистрироваться, используя уже существующее имя пользователя.

## Использование аутентификации в actions, связанных с заметками

