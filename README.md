# Django Finite State Machine Log

## Fork Rationale

- Original `django-fms-log` was only compatible with models implementing `django-fsm` that had numeric `id` fields.

- Changed the `object_id` within the logging model from `PostiveSmallIntegerField` to `CharField` of `max_length=32` so that it can work with either `uuid` (32 character) or `ulid` (26 character) primary keys

## Continue original README.md

Automatic logging for the excellent [Django FSM](https://github.com/kmmbvnr/django-fsm)
package.

Logs can be accessed before a transition occurs and before they are persisted to the database
by enabling a cached backend. See [Advanced Usage](#advanced-usage)

## Changelog

### 2.0.2-dev (unreleased)

### 2.0.1 (2020-03-26)

- Add support for django3.0
- Drop support for python2

### 1.6.2 (2019-01-06)

- Address Migration history breakage added in 1.6.1

### 1.6.1 (2018-12-02)

- Make StateLog.description field nullable

### 1.6.0 (2018-11-14)

- Add source state on transitions
- Fixed `get_state_display` with FSMIntegerField (#63)
- Fixed handling of transitions if target is None (#71)
- Added `fsm_log_description` decorator (#1, #67)
- Dropped support for Django 1.10 (#64)

### 1.5.0 (2017-11-29)

- cleanup deprecated code.
- add codecov support.
- switch to pytest.
- add Admin integration to visualize past transitions.

### 1.4.0 (2017-11-09)

- Bring compatibility with Django 2.0 and drop support of unsupported versions
  of Django: `1.6`, `1.7`, `1.9`.

### Compatibility

- Python 2.7 and 3.4+
- Django 1.8+
- Django-FSM 2+

## Installation

First, install the package with pip. This will automatically install any
dependencies you may be missing

```bash
pip install django-fsm-log
```

Register django_fsm_log in your list of Django applications:

```python
INSTALLED_APPS = (
    ...,
    'django_fsm_log',
    ...,
)
```

Then migrate the app to create the database table

```bash
python manage.py migrate django_fsm_log
```

## Usage

The app listens for the `django_fsm.signals.post_transition` signal and
creates a new record for each transition.

To query the log:

```python
from django_fsm_log.models import StateLog
StateLog.objects.all()
# ...all recorded logs...
```

### Disabling logging for specific models

By default transitions get recorded for all models. Logging can be disabled for
specific models by adding their fully qualified name to `DJANGO_FSM_LOG_IGNORED_MODELS`.

```python
DJANGO_FSM_LOG_IGNORED_MODELS = ('poll.models.Vote',)
```

### `for_` Manager Method

For convenience there is a custom `for_` manager method to easily filter on the generic foreign key:

```python
from my_app.models import Article
from django_fsm_log.models import StateLog

article = Article.objects.all()[0]

StateLog.objects.for_(article)
# ...logs for article...
```

### `by` Decorator

We found that our transitions are commonly called by a user, so we've added a
decorator to make logging this easy:

```python
from django.db import models
from django_fsm import FSMField, transition
from django_fsm_log.decorators import fsm_log_by

class Article(models.Model):

    state = FSMField(default='draft', protected=True)

    @fsm_log_by
    @transition(field=state, source='draft', target='submitted')
    def submit(self, by=None):
        pass
```

With this the transition gets logged when the `by` kwarg is present.

```python
article = Article.objects.create()
article.submit(by=some_user) # StateLog.by will be some_user
```

### Admin integration

There is an InlineForm available that can be used to display the history of changes.

To use it expand your own `AdminModel` by adding `StateLogInline` to its inlines:

```python
from django.contrib import admin
from django_fsm_log.admin import StateLogInline


@admin.register(FSMModel)
class FSMModelAdmin(admin.ModelAdmin):
    inlines = [StateLogInline]
```

### Advanced Usage

You can change the behaviour of this app by turning on caching for StateLog records.
Simply add `DJANGO_FSM_LOG_STORAGE_METHOD = 'django_fsm_log.backends.CachedBackend'` to your project's settings file.
It will use your project's default cache backend by default. If you wish to use a specific cache backend, you can add to
your project's settings:

```python
DJANGO_FSM_LOG_CACHE_BACKEND = 'some_other_cache_backend'
```

The StateLog object is now available after the `django_fsm.signals.pre_transition`
signal is fired, but is deleted from the cache and persisted to the database after `django_fsm.signals.post_transition`
is fired.

This is useful if:

- you need immediate access to StateLog details, and cannot wait until `django_fsm.signals.post_transition`
  has been fired
- at any stage, you need to verify whether or not the StateLog has been written to the database

Access to the pending StateLog record is available via the `pending_objects` manager

```python
from django_fsm_log.models import StateLog
article = Article.objects.get(...)
pending_state_log = StateLog.pending_objects.get_for_object(article)
```

## Running Tests

```bash
pip install tox
tox
```
