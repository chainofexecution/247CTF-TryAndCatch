# 247CTF "Try And Catch"
A writeup for 247CTF's "Try and Catch" challenge.

This challenge is a great showcase of what can happen when a developer does not disable/remove features that were never intended to be included with production build software.

## Tools used:
- Debian 11
- Firefox ESR (for interface with the webapp)
- Atom (for easy viewing/editing of Python scripts)
- Terminator (To use Python's package installer)

## The challenge:

![challenge](https://user-images.githubusercontent.com/92492482/190871980-0840a1a9-b7f3-488b-9970-892196535845.png)

We start off in a plaintext file that is the source code for the app we will be exploiting in this challenge.
![target_app](https://user-images.githubusercontent.com/92492482/190872004-3be6a474-0c78-47e0-b9c4-ad15ed88f89f.png)

A quick glance over the source code gleans that:
- The app is written in Python.
- The app is a [Flask](https://pythonbasics.org/what-is-flask-python/) WSGI web app:
```python
from flask import Flask, request
```
- Appending /calculator to the URL will run the app instead of showing it's source:
```python
@app.route('/calculator')
```
- The app is using the [Werkzeug](https://werkzeug.palletsprojects.com/en/2.2.x/) WSGI library and the debugger for that library is turned on:
```python
from werkzeug.debug import DebuggedApplication
```
...
```python
app.wsgi_app = DebuggedApplication(app.wsgi_app, True)
```
...
```python
if __name__ == "__main__":
    app.run(debug=True)
```

We could try and get the calculator app to crash and get us our juicy exception, but there is error handling present and we would have to trigger an error other than a Value or Type error to get an unmanaged exception instead of the `safe_cast()` function just returning a None type object:
```python
def safe_cast(val, to_type):
    try:
        return to_type(val)
    except (ValueError, TypeError):
        return None
```

If we don't get arround the error handling, the None type object returned from `safe_cast()` will signal the `flag()` function to simply give us an error message through a return, not through the exception handler:
```python
if None in (number_1, number_2, operation) or not operation in calculate:
        return "Invalid calculator parameters"
```

Instead, we can just try to get access to the debugger and use it to help us produce an exception.

We noted earlier the debugger was left running and a quick look at this page from Werkzeug's documentation provides us with a good indicator we are on the right track:
![danger](https://user-images.githubusercontent.com/92492482/190875292-d2df541b-c99b-4b10-9db1-52c3feeb6241.png)

We want to look over Werkzeug's code to find out more about how the debugger works as the online documentation only shows ways of interaction through the source code.
We can install Werkzeug using pip:
![pip](https://user-images.githubusercontent.com/92492482/190876152-d410d797-45e5-4aa1-842f-06c416d9cf11.png)

To find out where pip has stored Werkzeug's module files we can use the Python interpreter to print out the contents of `sys.path`, which is the object Python uses to store various paths that are important to the Python installation:
![pip_werkzeug](https://user-images.githubusercontent.com/92492482/190876534-1ced9e2c-48d7-4103-aa0d-9af12cc12915.png)

We are interested in the `__init__.py` inside the debug directory:
![werkzeug_files](https://user-images.githubusercontent.com/92492482/190876779-9506f38d-f7d6-497f-a801-7e460a86e375.png)

We can see in a comment at the start of the `DebuggedApplication()` that there is a variable for a default URL path to the console:
```python
class DebuggedApplication:
    """Enables debugging support for a given application::"""
```
...
```python
""":param console_path: the URL for a general purpose console."""
```

If we scroll further down we see the `console_path` variable is by default set to `/console`:
```python
def __init__(
        self,
        app: "WSGIApplication",
        evalex: bool = False,
        request_key: str = "werkzeug.request",
        console_path: str = "/console",
        console_init_func: t.Optional[t.Callable[[], t.Dict[str, t.Any]]] = None,
        show_hidden_frames: bool = False,
        pin_security: bool = True,
        pin_logging: bool = True,
```

