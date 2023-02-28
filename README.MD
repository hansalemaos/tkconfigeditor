# Dynamic tkinter config editor for files created with ConfigParser

If you have a friend or a client who doesn’t know anything about python, but wants to change some settings without touching the py file, 
you can use tkconfigeditor. Just add it on top of your python file. 

## pip install tkconfigeditor

### Tested against Python 3.9.15 / Windows 10 / Anaconda 


```python
[mysql]
host = localhost
user = user7
passwd = s$cret
db = ydb

[postgresql]
host = localhost
user = user8
passwd = mypwd$7
db = testdb
```

![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/1.png "Example")

```python
[dev]
api_url = https://gorest.co.in/public/v2/users
save_dir = /Users/dradecic/Desktop/users_data
save_file = users.json

[prod]
api_url = https://gorest.co.in/public/v2/users
save_dir = /tmp/users_data
save_file = users.json
```

![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/2.png "Example")

```python
[openweather]
api_key = mt732wbt5kh5ea4bkw7qytbb9gdkaew4

[twitter]
consumer_key = d53Qe4uf5LMWuQeQFFG66n6BN
consumer_secret = Yxf5n2BwnRELRrkkVjWgpTu48sz5KwGD52324PuwkFWdmVGP4m
access_token = 9922431-d53Qe4uf5LMWuQeQFFG66n6BN
access_token_secret = ubfWMR8WYucrzeaQdrqkm6SrhYTMVQSsxZWpNbtUCMX5u

[google]
api_key = ubfWMR8WYucrzeaQdrqkm6SrhYTMVQSsxZWpNbtUCMX5u
```

![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/3.png "Example")


```python
[book]
title: The Python Standard Library
author: Fredrik Lundh
email: fredrik@pythonware.com
version: 2.0-001115

[ematter]
pages: 250

[hardcopy]
pages: 350
```

![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/4.png "Example")


```python
[DEFAULT]
serverthreads = 45
content_type = text/html
content_length = 1024
autoscaling = no

[website_name.org]
username = your_name
login_mail = your_mail

[private.data.com]
password = secret_password
autoscaling = yes
```

![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/5.png "Example")

```python
[DEFAULT]
serverthreads = 45
content_type = text/html
content_length = 1024
autoscaling = no

[website_name.org]
username = your_name
login_mail = your_mail
login_date = new_date

[private.data.com]
password = new_password
autoscaling = yes

[new_section]
user_port = 4098
user_request = yes
```

![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/6.png "Example")

```python
[Common]
home_dir: /Users
library_dir: /Library
system_dir: /System
macports_dir: /opt/local

[Frameworks]
Python: 3.2
path: ${Common:system_dir}/Library/Frameworks/

[Arthur]
nickname: Two Sheds
last_name: Jackson
my_dir: ${Common:home_dir}/twosheds
my_pictures: ${my_dir}/Pictures
python_dir: ${Frameworks:path}/Python/Versions/${Frameworks:Python}
```


![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/7.png "Example")


### The first time you run it with a new config file, it will create a backup for the default settings:


![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/backupfiles.png "Example")


```python
from tkconfigeditor import start_config_and_load_vars, start_config

# This function opens the editor and adds the variables of the config file to the global dict. 

start_config_and_load_vars(
    cfgfile=r"C:\Users\Gamer\anaconda3\envs\dfdir\initest\cfg7.ini", # the config file, when you run it the first time, it will create a backup for the default settings
    title="Configuration", # window title
    icon=r"C:\Users\Gamer\Pictures\5.png", # needs to be png or None 
    res="1920x1080", 
    addbuttons=(), # you can add some additional buttons 
    mainlabelfont=("Helvetica", 15, "underline bold italic"),
    sublabelfont=("Helvetica", 14),
    varfont=("Helvetica", 10),
    buttonfont=("Helvetica", 12, "bold italic"),
    onezeroasboolean=True,  # 1 will be added as True to the global dict 
)

```



```python
# if you want to add a button
start_config_and_load_vars(
    cfgfile=r"C:\Users\Gamer\anaconda3\envs\dfdir\initest\cfg7.ini",
    title="Configuration",
    icon=r"C:\Users\Gamer\Pictures\5.png",
    res="1920x1080",
    addbuttons=(('Another button',lambda :os.system('start cmd')),),
    mainlabelfont=("Helvetica", 15, "underline bold italic"),
    sublabelfont=("Helvetica", 14),
    varfont=("Helvetica", 10),
    buttonfont=("Helvetica", 12, "bold italic"),
    onezeroasboolean=True,
)
```

![Example](https://github.com/hansalemaos/screenshots/raw/main/tkconfigedit/anotherbutton.png "Example")

```python

# If you don't want to add the variables of the config file to the global dict 
start_config(
    cfgfile=r"C:\Users\Gamer\anaconda3\envs\dfdir\initest\cfg7.ini",
    title="Configuration",
    icon=None,
    res="1024x768",
    addbuttons=(),
    mainlabelfont=("Helvetica", 15, "underline bold italic"),
    sublabelfont=("Helvetica", 14),
    varfont=("Helvetica", 10),
    buttonfont=("Helvetica", 12, "bold italic"),
)
```