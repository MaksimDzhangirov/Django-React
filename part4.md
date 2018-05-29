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