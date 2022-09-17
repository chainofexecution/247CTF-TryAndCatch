# 247CTF "Try And Catch"
A writeup for 247CTF's "Try and Catch" challenge.

This challenge is a great showcase of what can happen when a developer does not disable/remove features that were never intended to be included with production build software.

## Tools used:
- Debian(Buster)
- Firefox
- Atom

## The challenge:

![challenge](https://user-images.githubusercontent.com/92492482/190871980-0840a1a9-b7f3-488b-9970-892196535845.png)

We start off in a plaintext file that is the source code for the app we will be exploiting in this challenge.
![target_app](https://user-images.githubusercontent.com/92492482/190872004-3be6a474-0c78-47e0-b9c4-ad15ed88f89f.png)

A quick glance over the source code gleans that:
- They are using Python.
- The app is a Flask web application:
```python
from flask import Flask, request
```
- Appending /calculator to the URL will run the app instead of showing it's source:
```python
@app.route('/calculator')
```
- There are references to debugging:
```python
app.wsgi_app = DebuggedApplication(app.wsgi_app, True)
```
...
```python
if __name__ == "__main__":
    app.run(debug=True)
```
