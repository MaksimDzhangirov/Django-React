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

После этого Вы сможете осуществлять запросы к API заметок, используя curl или Ваш любымый http клиент, такой как `Postman` или `Insomnia`.

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