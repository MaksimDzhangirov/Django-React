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