## Django WSGI Request Lifecycle (Step-by-Step + Diagram)

Below is a step-by-step request lifecycle diagram for Django’s `WSGI(Web Server Gateway Interface)` flow, including all major components

### WSGI Server Receives Request: 
Gunicorn/uWSGI receives the HTTP request from the browser.

### Calls Django’s WSGI Callable
it calls
```python
application(environ, start_response)
```
where:
- environ = request metadata
- start_response = callback to begin the HTTP response

### Django Creates HttpRequest Object

Django converts the WSGI environ dict into a Python `HttpRequest`.

### Request Middlewares Run (top → bottom)

Examples:
- SecurityMiddleware
- AuthenticationMiddleware
- SessionMiddleware etc


Each middleware can:
- modify request
- return a response early (short-circuit)

### URL Routing
Django matches the URL pattern in urls.py:
```
URLconf → resolver → selected view
```

### View Processing + ORM Queries

View executes:
- Business logic
- Database queries (via Django ORM)
- Calls serializers/templates

### Template Rendering (if needed)
If view renders a template:
```python
return render(request, "home.html", context)
```
Django loads template, applies context → generates HTML.

### Response Middlewares Run (bottom → top)

Examples:
- GZipMiddleware
- ETagMiddleware
- MessageMiddleware etc

Each middleware can modify the outgoing response.

### Django Returns Response to WSGI Server

Django returns:
- Status line
- Headers
- Body as iterable of bytes

WSGI server calls:
```
start_response(status, headers)
```
### WSGI Server Sends Response to Browser

Gunicorn writes the response to the client socket.

### Here is the full example

wsgi.py
```python
application = get_wsgi_application()
```
core/wsgi.py
```python
def get_wsgi_application():
    """
    The public interface to Django's WSGI support. Return a WSGI callable.

    Avoids making django.core.handlers.WSGIHandler a public API, in case the
    internal WSGI implementation changes or moves in the future.
    """
    django.setup(set_prefix=False)
    return WSGIHandler()
```
handlers/wsgi.py
```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        request = self.request_class(environ)
        response = self.get_response(request)

        response._handler_class = self.__class__

        status = "%d %s" % (response.status_code, response.reason_phrase)
        response_headers = [
            *response.items(),
            *(("Set-Cookie", c.output(header="")) for c in response.cookies.values()),
        ]
        start_response(status, response_headers)
        if getattr(response, "file_to_stream", None) is not None and environ.get(
            "wsgi.file_wrapper"
        ):
            # If `wsgi.file_wrapper` is used the WSGI server does not call
            # .close on the response, but on the file wrapper. Patch it to use
            # response.close instead which takes care of closing all files.
            response.file_to_stream.close = response.close
            response = environ["wsgi.file_wrapper"](
                response.file_to_stream, response.block_size
            )
        return response
```
### In a Diagram

                ┌──────────────────────────────┐
                │      Client (Browser)        │
                └───────────────┬──────────────┘
                                │ HTTP Request
                                ▼
                ┌──────────────────────────────┐
                │      WSGI Server             │
                │  (Gunicorn / uWSGI / mod_wsgi)│
                └───────────────┬──────────────┘
                       Calls application(environ, start_response)
                                │
                                ▼
                ┌──────────────────────────────┐
                │      WSGI Application        │
                │ get_wsgi_application()       │
                └───────────────┬──────────────┘
                                │
                                ▼
                ┌──────────────────────────────┐
                │     Django Middleware Stack   │
                │  (from top → bottom)          │
                └───────────────┬──────────────┘
                                │ Request object created
                                ▼
                ┌──────────────────────────────┐
                │       URL Resolver            │
                │   (urls.py, matching route)   │
                └───────────────┬──────────────┘
                                │
                                ▼
                ┌──────────────────────────────┐
                │        View Function/Class    │
                │   (process logic, ORM query)  │
                └───────────────┬──────────────┘
                                │
                                ▼
                ┌──────────────────────────────┐
                │ Template Rendering (optional) │
                │   context → HTML              │
                └───────────────┬──────────────┘
                                │ Response object created
                                ▼
                ┌──────────────────────────────┐
                │    Middleware (Response Phase)│
                │   bottom → top                │
                └───────────────┬──────────────┘
                                │
                                ▼
                ┌──────────────────────────────┐
                │      WSGI Application         │
                │  returns iterable of bytes    │
                └───────────────┬──────────────┘
                                │ start_response(status, headers)
                                ▼
                ┌──────────────────────────────┐
                │         WSGI Server           │
                │  (writes response to socket)  │
                └───────────────┬──────────────┘
                                │ HTTP Response
                                ▼
                ┌──────────────────────────────┐
                │        Client Browser         │
                └──────────────────────────────┘

## WSGI is a single callable—what does that mean?
In WSGI, the entire web application must be exposed as one callable object—a function or a class implementing `__ call __`.
The WSGI server invokes this callable for every HTTP request, passing environ (request data) and start_response (to begin the response).
All the application logic—middleware, routing, views—happens inside this single callable.
This simple contract allows any WSGI server to run any WSGI-compliant framework.

- WSGI is `Sysnchronus` and `Blocking` in nature. So it serves one request at a time.
- Parallel request handling is not possible.
