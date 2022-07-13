---
layout: post
title: "How to create virtual env in python"
subtitle: "How to create virtual env in python"
category: studies
tags: Debugging
# image: 
---

# *Python Virtual environment*

This page describe *__`how to set virtual environment in Windows/Ubuntu.`__*   

<br>

## *__Step by step__*
*** 
<br>

### *__1. Install virtualenv__*

```
python3.7 -m pip install virtualenv
```

### *__2. Make virtual env.__*
```bash
python3.7 -m virtualenv `venv-folder-name`
```

### *__3. Activate Virtualenv__*

change directory into `venv-folder-name`(in example, "githubio") and type activate.

```cmd
# change directory
cd githubio
```

```shell
# Ubuntu
$ source bin/activate
```

```cmd
# Windows
call scripts\activate
```

If activated successfully, (`venv-folder-name`) is shown at front of command. (in example, (githubio)). 
![venv](/assets/img/posts/studies/debugging/md-img-paste-2020-12-26-11-34-00.png)

It's better to double check python version typing like
```python
python -V
```
![python-version](/assets/img/posts/studies/debugging/md-img-paste-2020-12-26-12-00-00.png)

<br>

## *__Disable virutal environment__*
***

To deactivate, just type *`deactivate`*

<br>


## *__Tips__*
***

Also, you could specify python version like belows.  
(mind you should install wanted-python-version in advance.)
```python
# python3.6
python3 -m virtualenv venv 
virtualenv venv --python=python3.6
```

