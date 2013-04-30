heroku-django-s3
================

####*A clean project of a Django web-framework running on Heroku with static files served from Amazon S3, attempting to follow the [12-factor](http://www.12factor.net/) design pattern.*

**Django** is a web framework that simplifies the work required to make a web app in python.

**Heroku** is touted as one of the best 'platform as a service' providers. You can host apps without having to get too involved with installing your own serving software etc.

**Amazon S3** is a quick, cheap way to host static files (images, js, css etc. that doesn't change) because heroku doesn't do that.

This collection is touted as one fo the best ways to serve up django apps, that's free initially and can scale up easily.

### Pre-requisites
This setup using the excellent virtualenvwrapper to isolate the installed dependencies and environmental variables.

- heroku account and [toolbelt](https://toolbelt.heroku.com/) installed on your computer
- git installed
- [virtualenv](https://pypi.python.org/pypi/virtualenv) and [virtualenvwrapper](https://bitbucket.org/dhellmann/virtualenvwrapper)
- an amazon web services account

## Installation

1. First make a new virtualenv
		
		```bash
		mkvirtualenv [name-of-your-project]
		```

2. Now git clone this repo 

	    ```bash
    	git clone https://github.com/jordn/heroku-django-s3 [name-of-your-project]
    	```

3. Now 	need to install all the dependencies (django, psycopg2, gunicorn dj-database-url boto and django-storages) with pip. These are specified in requirements.txt (if you edit this to remove the version numbers it will install the latest versions available)

    	```bash
    	pip install -r requirements.txt
		```

4. The django settings is kept general and all the private or environment-dependant settings are kept as environmental variables (see http://www.12factor.net/config for why).
	We need to set these settings to come into effect everytime we enter this virtual environment. virtualenvwrapper does this with a `postactivate` script

		```bash
    	vim $VIRTUAL_ENV/bin/postactivate 
    	```

	use `vim`, `nano`, or `subl` (sublime text) or whatever to edit the file. Add set the following variables:

		```bash
    	#!/bin/zsh
    	# This hook is run after this virtualenv is activated.
    	# Django database
    	export DATABASE_URL=sqlite:////[path to whereever you'd like to store the sqlite (easiest) database for local dev]

    	# Django static file storage
    	export AWS_STORAGE_BUCKET_NAME=[YOUR AWS S3 BUCKET NAME]
    	export AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXXXXXX
    	export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXX-XXXXXXXXXX
    
    	# Django debug setting
    	export DJ_DEBUG=True
    	export DJ_SECRET_KEY=[Any random sequence of 40ish characters  - django uses it for added security]
    	```

5. These changes won't come into affect until this script is run. Easiest way is to reopen the virtualenv

    	```bash
    	workon [name-of-your-project]
    	```

6. Everything should now work for local development.

    	```bash
    	python manage.py syncdb
    	...
    	python manage.py runserver
    	```

7. Should be able to see the admin pages at `http://127.0.0.1:8000/admin/`

8. Create the app on heroku

    	```bash
    	heroku create [name-of-your-project]
    	```

9. Push everything to heroku and it will detect we're making a python web app and install everything in requirements.txt (update this file with `pip freeze > requirements.txt`)

    	```bash
    	git push heroku master
    	```

10. Heroku will fail because the django app's settings aren't available as environmental variables so set them (`DATABASE_URL` is set by default on heroku):

        ```bash
    	heroku config:add AWS_STORAGE_BUCKET_NAME=[YOUR AWS S3 BUCKET NAME]
    	heroku config:add AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXXXXXX
    	heroku config:add AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXX-XXXXXXXXXX
    	heroku config:add DJ_SECRET_KEY=[Any random sequence of around 40 characters django uses for added security]
    	```

    You can turn debug on/off by changing the DJ_DEBUG setting (only do something has gone wrong):

    	```bash
    	heroku config:add DJ_DEBUG=True
    	``` 
    	
    *Note: static files aren't served from S3 in debug mode*

11. Should now be pretty much setup and can get on with building your django app

    	```bash
    	python manage.py syncdb
    	...
    	heroku open
    	```

    Lastly, go into settings.py and change the trusted hosts to be specifically your domains (for added security)

    	```python
    	ALLOWED_HOSTS = ['[your-project-name].herokuapp.com']
    	```



### What's different from a fresh django install
	
#####Programs installed
 - django
 - psycopg2 (to be able to talk to postgreSQL databases)
 - gunicorn (python HTTP server to use from heroku)
 - [dj-database-url](https://github.com/kennethreitz/dj-database-url) (to use a URL environmental variable to reference the location of the database)
 - [django-storages](http://django-storages.readthedocs.org/en/latest/backends/amazon-S3.html) (custom storage backends for django, best S3 support makes use of 'boto'...)
 - Boto (Python interface to Amazon Web Services, simplifies the AWS connection to just the access keys)

#####Changes Made

`settings.py`

Removed sites add-on that can enable multiple sites to use the same back-end but confuses matters here.

	```python
     # 'django.contrib.sites' #REMOVED
     ```

As part of the plan to make the settings.py file to be transferable and secure did the following:

Make DEBUG an environmental variable

	```python
	import os 

	# Added to help use env variables
	def env_var(key, default=None):
	    """Retrieves env vars and makes Python boolean replacements"""
	    val = os.environ.get(key, default)
	    if val == 'True':
	        val = True
	    elif val == 'False':
	        val = False
	    return val

	DEBUG = env_var('DJ_DEBUG', False) #Unless env var is set to True, debug is off
	````

SECRET_KEY removed form the file to be stored as env var:

	SECRET_KEY = os.environ['DJ_SECRET_KEY']

Allowed hosts (which domains can host the site when debug is off) set to allow only subdomains of heroku. **Remember to change this to be most specific!**
	
	```python
	# SET TO THE  SUBDOMAIN ON HEROKU, ANYWHERE ELSE IT'S HOSTED (INSECURE PRESENTLY)    <--------!!!!!
	ALLOWED_HOSTS = ['.herokuapp.com']
	```

Added database settings as env var
	
	```python	
	# Parse database configuration from $DATABASE_URL
	import dj_database_url
	DATABASES['default'] =  dj_database_url.config(default=os.environ.get('DATABASE_URL'))

	# Honor the 'X-Forwarded-Proto' header for request.is_secure()
	SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
	```

Uncommented admin, admindocs and `'storages'` to `INSTALLED_APPS`

Added settings to store statics on S3
	
	```python
	#Storage on S3 settings are stored as os.environs to keep settings.py clean 
	if not DEBUG:
		AWS_STORAGE_BUCKET_NAME = os.environ['AWS_STORAGE_BUCKET_NAME']
		AWS_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
		AWS_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']
		STATICFILES_STORAGE = 'storages.backends.s3boto.S3BotoStorage'
		S3_URL = 'http://%s.s3.amazonaws.com/' % AWS_STORAGE_BUCKET_NAME
		STATIC_URL = S3_URL
	```

`urls.py`
Uncommented lines 4, 5, 13 and 16 from urls to enable admin urls

