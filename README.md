heroku-django-s3
================

A clean project of a Django web-framework running on Heroku with static files served from Amazon S3.


(heroku, git, virtualenv and virtualenvwrapper)

mkvirtualenv django
mkdir herokuproject
cd herokuproject
git init
git commit -a -m 'First commit of generic heroku/django/s3 project'
pip install django psycopg2 gunicorn dj-database-url (https://github.com/kennethreitz/dj-database-url to use a 12factor inspired DATABSE_URL env var to configure django)
django-admin.py startproject djangoproject . **********  full stop added after failure
<!-- This is differing from https://devcenter.heroku.com/articles/django  -->

echo "web: gunicorn djangoproject.wsgi" > Procfile
foreman start (it works!)

cd ..
added .gitignore
git add .
git commit -m "Pip installed, django project started"

[added database url bit to end of settings]

heroku create [name]

git push heroku master
[broken].

[Copied the django project up on level ********** and did necessary git commits]
http://fathomless-ravine-5669.herokuapp.com/  working, standard django project!!



Uncommented admin, admin docs in settings.py INSTALLED_APPS.
Uncommented lines 4,5 13 and 16 from urls to enable admin

#local
 "settings.DATABASES is improperly configured. Please supply the ENGINE value. Check settings documentation for more details.""
so added a default to that we did earlier: 
	import os
	DATABASES['default'] =  dj_database_url.config(default=os.environ.get('DATABASE_URL'))

and set $VIRTUAL_ENV/django/bin/postactivate shell script to include
	#!/bin/zsh
	# This hook is run after this virtualenv is activated.
	export DATABASE_URL=sqlite:////Users/jordanburgess/Projects/herokuproject/djangoproject/sqlite.db

syncdb (have kept settings in git, not included sqlite db)
**working admin! (if using runserver, css works. if using foreman, css not showing)**

#remote
http://fathomless-ravine-5669.herokuapp.com/admin/ 
	"DoesNotExist at /admin/
	Site matching query does not exist. Lookup parameters were {'pk': 1}"

Removed     # 'django.contrib.sites', form INSTALLED_APPS
Working but no statics served!



#Statics
Django-storages (custom storage backends for django, best supported for S3 if you install boto) http://django-storages.readthedocs.org/en/latest/backends/amazon-S3.html
Boto (Python interface to Amazon Web Services, simplifies the AWS connection to just your access keys)
	pip install django-storages boto 

Added 'storages' to INSTALLED_APPS in settings.py

and the follwing to the bottom of settings.py
	#Storage on S3 settings are stored as os.environs to keep settings.py clean 
	AWS_STORAGE_BUCKET_NAME = os.environ['AWS_STORAGE_BUCKET_NAME']
	STATICFILES_STORAGE = 'storages.backends.s3boto.S3BotoStorage'
	S3_URL = 'http://%s.s3.amazonaws.com/' % AWS_STORAGE_BUCKET_NAME
	STATIC_URL = S3_URL

set env variable on heroku by 
	heroku config:add AWS_STORAGE_BUCKET_NAME='static-server'

and locally (added to postactivate script)
	export AWS_STORAGE_BUCKET_NAME=static-server
	export AWS_ACCESS_KEY_ID=AKIAIM3UG6KGL46ROKSA
	export AWS_SECRET_ACCESS_KEY=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

collectstatic and it plops it all on s3
and it works. 