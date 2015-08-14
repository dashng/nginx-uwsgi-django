This document has now been incorporated into the uWSGI documentation

	http://uwsgi-docs.readthedocs.org/en/latest/tutorials/Django_and_nginx.html


******************************
Set up Django, nginx and uwsgi
******************************

Steps with explanations to set up a server using:

* virtualenv
* Django
* nginx
* uwsgi

Concept
=======

nginx will face the outside world. It will serve media files (images, CSS, etc) directly from the file system. However, it can't talk directly to Django applications; it needs something that will run the application, feed it requests from the web, and return responses.

That's uwsgi's job. uwsgi will create a Unix socket, and serve responses to nginx via the uwsgi protocol - the socket passes data in both directions::

    the outside world <-> nginx <-> the socket <-> uwsgi

Before you start
==================

virtualenv
----------

Make sure you are in a virtualenv - you will install a system-wide uwsgi later.

Django
------

I am assuming Django 1.4. It automatically creates a wsgi module when you create a project. If you're using an earlier version, you will have to find a Django wsgi module for your project.

Note that I'm also assuming a Django 1.4 project structure, in which you see paths like::

	/path/to/your/project/project/

(i.e. it creates nested directories with the name of your project). Adjust the examples if you using an earlier Django.

It will also be helpful if you are in your Django project's directory. If you don't have one ready, just create a directory for now.

About the domain and port
-------------------------

I'll call your domain example.com. Substitute your own FQDN or IP address.

Throughout, I'm using port 8000. You can use whatever port you want of course, but I have chosen this one so it doesn't conflict with anything a web server might be doing already.

Basic uwsgi intallation and configuration
=========================================

Install uwsgi
-------------

::

pip install uwsgi

Basic test
----------

Create a file called test.py::

	# test.py
	def application(env, start_response):
	    start_response('200 OK', [('Content-Type','text/html')])
	    return "Hello World"

Run::

	uwsgi --http :8000 --wsgi-file test.py

The options mean:

http :8000
	use protocol http, port 8000 

wsgi-file test.py
	load the specified file

This should serve a hello world message directly to the browser on port 8000. Visit::

	http://example.com:8000

to check.                       

Test your Django project
------------------------

Now we want uwsgi to do the same thing, but to run a Django site instead of the test.py module.

But first, make sure that your project actually works! Now you need to be in your Django project directory.

::

	python manage.py runserver 0.0.0.0:8000

Now run it using uwsgi::

	uwsgi --http :8000 --chdir /path/to/your/project --module project.wsgi --virtualenv /path/to/virtualenv

The options mean:

chdir /path/to/your/project
	use your Django project directory as a base
module project.wsgi
	i.e. the Python wsgi module in your project
virtualenv /path/to/virtualenv
	the virtualenv

There is an alternative to using the `--module` option, by referring instead to the wsgi *file*::

wsgi-file /path/to/your/project/project/wsgi.py
	i.e. the system file path to the wsgi.py file


Point your browser at the server; if the site appears, it means uwsgi can serve your Django application from your virtualenv. Media/static files may not be served properly, but don't worry about that.

Now normally we won't have the browser speaking directly to uwsgi: nginx will be the go-between.

Basic nginx
===========

Install nginx
-------------

The version of Nginx from Debian stable is rather old. We'll install from backports.

::

	sudo pico /etc/apt/sources.list     # edit the sources list

Add::

	# backports
	deb http://backports.debian.org/debian-backports squeeze-backports main

Run::

	sudo apt-get -t squeeze-backports install nginx	# install nginx
	sudo /etc/init.d/nginx start	# start nginx

And now check that the server is serving by visiting it in a web browser on port 80 - you should get a message from nginx: "Welcome to nginx!"

Configure nginx for your site
-----------------------------

Check that your nginx has installed a file at `/etc/nginx/uwsgi_params`. If not, copy http://projects.unbit.it/uwsgi/browser/nginx/uwsgi_params to your directory, because nginx will need it. Easiest way to get it::

	wget http://projects.unbit.it/uwsgi/export/3fab63fcad3c77e7a2a1cd39ffe0e50336647fd8/nginx/uwsgi_params

Create a file called nginx.conf, and put this in it::

	# nginx.conf
	upstream django {
	    # connect to this socket
	    # server unix:///tmp/uwsgi.sock;	# for a file socket
	    server 127.0.0.1:8001;	# for a web port socket 
	    }
 
	server {
	    # the port your site will be served on
	    listen      8000;
	    # the domain name it will serve for
	    server_name .example.com;	# substitute your machine's IP address or FQDN
	    charset     utf-8;
   
	    #Max upload size
	    client_max_body_size 75M;	# adjust to taste

	    # Django media
	    location /media  {
			alias /path/to/your/project/project/media;	# your Django project's media files
	    }
   
		location /static {
			alias /path/to/your/project/project/static;	# your Django project's static files
		}
   
	    # Finally, send all non-media requests to the Django server.
	    location / {
	        uwsgi_pass  django;
	        include     /etc/nginx/uwsgi_params; # or the uwsgi_params you installed manually 
	        }
	    }

Symlink to this file from /etc/nginx/sites-enabled so nginx can see it::

	sudo ln -s ~/path/to/your/project/nginx.conf /etc/nginx/sites-enabled/

Basic nginx test
----------------

Restart nginx::

	sudo /etc/init.d/nginx restart

Check that media files are being served correctly:

Add an image called media.png to the /path/to/your/project/project/media directory

Visit 

http://example.com:8000/media/media.png     

If this works, you'll know at least that nginx is serving files correctly.

nginx and uwsgi and test.py
===========================

Let's get nginx to speak to the hello world test.py application.

::

	uwsgi --socket :8001 --wsgi-file test.py

This is nearly the same as before, except now we are not using http between uwsgi and nginx, but the (much more efficient) uwsgi protocol, and we're doing it on port 8001. nginx meanwhile will pass what it finds on that port to port 8000. Visit:

http://example.com:8000/

to check.

Meanwhile, you can try to have a look at the uswgi output at:

http://example.com:8001/

but quite probably, it won't work because your browser speaks http, not uwsgi.

Using sockets instead of ports
==============================

It's better to use Unix sockets than ports - there's less overhead.

Edit nginx.conf. 

uncomment
	server unix:///tmp/uwsgi.sock;
comment out
	server 127.0.0.1:8001;

and restart nginx.

Runs uwsgi again::

	uwsgi --socket /tmp/uwsgi.sock --wsgi-file test.py

Try http://example.com:8000/ in the browser.

If that doesn't work
--------------------

Check your nginx error log(/var/log/nginx/error.log). If you see something like::

	connect() to unix:///path/to/your/project/uwsgi.sock failed (13: Permission denied)

then probably you need to manage the permissions on the socket (especially if you are using a file not in /tmp as suggested).

Try::

	uwsgi --socket /tmp/uwsgi.sock --wsgi-file test.py --chmod-socket=644 # 666 permissions (very permissive)

or::

	uwsgi --socket /tmp/uwsgi.sock --wsgi-file test.py --chmod-socket=664 # 664 permissions (more sensible) 

You may also have to add your user to nginx's group (probably www-data), or vice-versa, so that nginx can read and write to your socket properly.                                         

Running the Django application with uswgi and nginx
===================================================

Let's run our Django application::

	uwsgi --socket /tmp/uwsgi.sock --chdir /path/to/your/project --module project.wsgi --virtualenv /path/to/virtualenv --chmod-socket=664

Now uwsgi and nginx should be serving up your Django application.


a uwsgi .ini file for our Django application
============================================

Deactivate your virtualenv::

	deactivate

and install uwsgi system-wide::

	sudo pip install uwsgi
                                                             
We can put the same options that we used with uwsgi into a file, and then ask uwsgi to run with that file::

	# django.ini file
	[uwsgi]

	# master
	master			= true

	# maximum number of processes
	processes 		= 10

	# the socket (use the full path to be safe)
	socket          = /tmp/uwsgi.sock 

	# with appropriate permissions - *may* be needed
	# chmod-socket    = 664

	# the base directory
	chdir           = /path/to/your/project 

	# Django's wsgi file
	module          = project.wsgi

	# the virtualenv 
	home            = /path/to/virtualenv

	# clear environment on exit
	vacuum          = true           


And run uswgi using the file::

	uwsgi --ini django.ini

Note:

--ini django.ini
	use the specified .ini file

Test emperor mode
=================

uwsgi can run in 'emperor' mode. In this mode it keeps an eye on a directory of uwsgi config files, and spawns instances ('vassals') for each one it finds. 

Whenever a config file is amended, the emperor will automatically restart the vassal.

::

	# symlink from the default config directory to your config file
	sudo ln -s /path/to/your/project/django.ini /etc/uwsgi/vassals/

	# run the emperor as root
	sudo uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data --master

The options mean:

emperor /etc/uwsgi/vassals
	look there for vassals (config files)
uid www-data
	run as www-data once we've started
gid www-data
	run as www-data once we've started

Check the site; it should be running. 

Make uwsgi startup when the system boots
========================================

The last step is to make it all happen automatically at system startup time.

Edit /etc/rc.local and add::

	/usr/local/bin/uwsgi --emperor /etc/uwsgi/vassals --uid www-data --gid www-data --master

before the line "exit 0".

And that should be it!
