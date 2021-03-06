Title: Web исключения в Django
Date: 2017-06-10 15:00
Modified: 2017-06-10 15:00
Category: Python
Tags: django, exceptions, web, http, response
Image: /media/web-exceptions/web-exceptions.png
Image_width: 1280
Image_height: 791
Summary:
    Как и положено веб фраймворку, `Django` позволяет
    возвращать в ответ на запрос `HttpResponse` с любым статус кодом
    из диапазона `[100 .. 599]`. Этот ответ должен быть явно отправлен
    через `return` во вьюхе обрабатывающей запрос.
    Однако `pythonyc way` предусматривает не только
    явный `return`, но и гибкую обработку исключений.
    Рассмотрим веб исключения в `Django`.

Как и положено веб фраймворку, `Django` позволяет
возвращать в ответ на запрос `HttpResponse` с любым статус кодом
из диапазона `[100 .. 599]`. Этот ответ должен быть явно отправлен
через `return` во вьюхе обрабатывающей запрос.
Однако `pythonyc way` предусматривает не только
явный `return`, но и гибкую обработку исключений.
Рассмотрим веб исключения в `Django`.

### Стандартные исключения

`Django` допускает ограниченный набор `http` ответов через
выбрасывание исключений.

- `400` [SuspiciousOperation](https://docs.djangoproject.com/en/1.11/ref/exceptions/#suspiciousoperation)
- `403` [PermissionDenied](https://docs.djangoproject.com/en/1.11/ref/exceptions/#permissiondenied)
- `404` [Http404](https://docs.djangoproject.com/en/1.11/topics/http/views/#the-http404-exception)
- `500` Любое другое неперехваченное исключение

#### Настройка http ответа

Ответ возвращаемый каждым из этих исключений можно персонализировать
через перегрузку соответствующих
[хендлеров](https://docs.djangoproject.com/en/1.11/topics/http/views/#customizing-error-views)
в файле `urls.py`.

```python
# views.py
from django.views.generic import TemplateView

class ErrorHandler(TemplateView):

    """ Render error template """

    error_code = 404
    template_name = 'index/error.html'

    def dispatch(self, request, *args, **kwargs):
        """ For error on any methods return just GET """
        return self.get(request, *args, **kwargs)

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['error_code'] = self.error_code
        return context

    def render_to_response(self, context, **response_kwargs):
        """ Return correct status code """
        response_kwargs = response_kwargs or {}
        response_kwargs.update(status=self.error_code)
        return super().render_to_response(context, **response_kwargs)
```

```python
# urls.py
from index.views import ErrorHandler

# error handing handlers - fly binding
for code in (400, 403, 404, 500):
    vars()['handler{}'.format(code)] = ErrorHandler.as_view(error_code=code)
```

#### Тестирование хендлеров

Проверить работу хендлеров, можно явно выбросив соответствующие исключения.
Перегрузив какую либо из существующих вьюх через `mock`.
Но этот способ не позволяет проверить ответ со статусом `500`, т.к. в тестах
общие исключения не перехватываются. Полностью убедится в корректности
обработки ошибки сервера -- можно только через
[LiveServerTestCase](https://docs.djangoproject.com/en/1.11/topics/testing/tools/#django.test.LiveServerTestCase).

```python
# tests.py
from unittest import mock

from django.test import TestCase
from django.core.exceptions import SuspiciousOperation, PermissionDenied
from django.http import Http404
from index import views

class ErrorHandlersTestCase(TestCase):

    """ Check is correct error handlers work """

    def raise_(exception):
        def wrapped(*args, **kwargs):
            raise exception('Test exception')
        return wrapped

    def test_index_page(self):
        """ Should check is 200 on index page """
        response = self.client.get('/')
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'index/index.html')

    @mock.patch('index.views.IndexView.get', raise_(Http404))
    def test_404_page(self):
        """ Should check is 404 page correct """
        response = self.client.get('/')
        self.assertEqual(response.status_code, 404)
        self.assertTemplateUsed(response, 'index/error.html')
        self.assertIn('404 Page not found', response.content.decode('utf-8'))

    @mock.patch('index.views.IndexView.get', views.ErrorHandler.as_view(error_code=500))
    def test_500_page(self):
        """ Should check is 500 page correct """
        response = self.client.get('/')
        self.assertEqual(response.status_code, 500)
        self.assertTemplateUsed(response, 'index/error.html')
        self.assertIn('500 Server Error', response.content.decode('utf-8'))

    @mock.patch('index.views.IndexView.get', raise_(SuspiciousOperation))
    def test_400_page(self):
        """ Should check is 400 page correct """
        response = self.client.get('/')
        self.assertEqual(response.status_code, 400)
        self.assertTemplateUsed(response, 'index/error.html')
        self.assertIn('400 Bad request', response.content.decode('utf-8'))

    @mock.patch('index.views.IndexView.get', raise_(PermissionDenied))
    def test_403_page(self):
        """ Should check is 403 page correct """
        response = self.client.get('/')
        self.assertEqual(response.status_code, 403)
        self.assertTemplateUsed(response, 'index/error.html')
        self.assertIn('403 Permission Denied', response.content.decode('utf-8'))
```

### Расширенные исключения

В [AioHTTP](https://github.com/aio-libs/aiohttp), любой `HttpResponse`
можно выбрасывать как исключение. Это действительно круто. Но если попробовать
такое в `Django`, получаем ошибку типа,
т.к. исключение должно быть наследником базового класса исключений.

```python
>>> from django.http import HttpResponse
>>> raise HttpResponse()
Traceback (most recent call last):
  File "<console>", line 1, in <module>
TypeError: exceptions must derive from BaseException
```

С пакетом [Django web exceptions](https://pypi.python.org/pypi/django-web-exceptions)
можно выбрасывать любой `http` ответ как исключение.

```python
>>> from web_exceptions import exceptions
>>> raise exceptions.HTTPOk()
Traceback (most recent call last):
  File "<console>", line 1, in <module>
web_exceptions.exceptions.HTTPOk: OK
```

#### Быстрый старт

1 Устанавливаем через `pip`

```
pip install django-web-exceptions
```

2 Подключаем `middleware`

```python
# settings.py
MIDDLEWARE = (
    # ...
    'web_exceptions.middleware.WebExceptionsMiddleware',
    # ...
)
```

3 Вызываем исключение

```python
# views.py
from web_exceptions import exceptions

# ...

def index(request):
    """ Simple view raise redirectexception """
    raise exceptions.HTTPMovedPermanently('/foo')
```

#### Как это работает

Для того чтобы выбрасывать `http` ответ как исключение,
в базовом классе наследуемся сразу от `HttpResponse` и `Exception`.

```python
from django.http import HttpResponse

class HTTPException(HttpResponse, Exception):

    """
    Base Web explanation
    In subclasses should set status_code attr
    """

    status_code = None
    empty_body = False
    reason = None

    def __init__(self, *, content=None, headers=None, **kwargs):
        HttpResponse.__init__(self, content or "", status=self.status_code, **kwargs)
        headers = headers or {}
        for key, value in headers.items():
            self[key] = value
        self._reason_phrase = self._reason_phrase or self.reason
        if not (self.content or self.empty_body):
            self.content = "{}: {}".format(self.status_code, self.reason_phrase).encode(self.charset)
        Exception.__init__(self, self.reason_phrase)
```

##### 200x status code

- `200` HTTPOk
- `201` HTTPCreated
- `202` HTTPAccepted
- `203` HTTPNonAuthoritativeInformation
- `204` HTTPNoContent
- `205` HTTPResetContent
- `206` HTTPPartialContent

##### 300x status code

- `300` HTTPMultipleChoices
- `301` HTTPMovedPermanently
- `302` HTTPFound
- `303` HTTPSeeOther
- `304` HTTPNotModified
- `305` HTTPUseProxy
- `307` HTTPTemporaryRedirect
- `308` HTTPPermanentRedirect

##### 400x status code

- `400` HTTPBadRequest
- `401` HTTPUnauthorized
- `402` HTTPPaymentRequired
- `403` HTTPForbidden
- `404` HTTPNotFound
- `405` HTTPMethodNotAllowed
- `406` HTTPNotAcceptable
- `407` HTTPProxyAuthenticationRequired
- `408` HTTPRequestTimeout
- `409` HTTPConflict
- `410` HTTPGone
- `411` HTTPLengthRequired
- `412` HTTPPreconditionFailed
- `413` HTTPRequestEntityTooLarge
- `414` HTTPRequestURITooLong
- `415` HTTPUnsupportedMediaType
- `416` HTTPRequestRangeNotSatisfiable
- `417` HTTPExpectationFailed
- `421` HTTPMisdirectedRequest
- `426` HTTPUpgradeRequired
- `428` HTTPPreconditionRequired
- `429` HTTPTooManyRequests
- `431` HTTPRequestHeaderFieldsTooLarge
- `451` HTTPUnavailableForLegalReasons

##### 500x status code

- `500` HTTPInternalServerError
- `501` HTTPNotImplemented
- `502` HTTPBadGateway
- `503` HTTPServiceUnavailable
- `504` HTTPGatewayTimeout
- `505` HTTPVersionNotSupported
- `506` HTTPVariantAlsoNegotiates
- `510` HTTPNotExtended
- `511` HTTPNetworkAuthenticationRequired

### Web exceptions links

- Source code on GH -- [samael500/web-exceptions](https://github.com/samael500/web-exceptions)
- Pypi package -- [django-web-exceptions](https://pypi.python.org/pypi/django-web-exceptions/0.1.4)
- Docs on readthedocs -- [web-exceptions.readthedocs.io](http://web-exceptions.readthedocs.io/en/latest/readme.html)
- Example usage proj on GH -- [samael500/web-exceptions/example](https://github.com/samael500/web-exceptions/tree/master/example)
