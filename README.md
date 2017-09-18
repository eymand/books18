Books Concierge (2017 version)
==============================

* [What is this?](#what-is-this)
* [Assumptions](#assumptions)
* [What's in here?](#whats-in-here)
* [Bootstrap the project](#bootstrap-the-project)
* [Hide project secrets](#hide-project-secrets)
* [Save media assets](#save-media-assets)
* [Add a page to the site](#add-a-page-to-the-site)
* [Run the project](#run-the-project)
* [COPY configuration](#copy-configuration)
* [COPY editing](#copy-editing)
* [Load books and covers](#load-books-and-covers)
* [Arbitrary Google Docs](#arbitrary-google-docs)
* [Run Python tests](#run-python-tests)
* [Run Javascript tests](#run-javascript-tests)
* [Compile static assets](#compile-static-assets)
* [Test the rendered app](#test-the-rendered-app)
* [Deploy to S3](#deploy-to-s3)
* [Report analytics](#report-analytics)

What is this?
-------------

[A snappy looking presentation of NPR contributors' favorite books of the year.](http://apps.npr.org/best-books-2017/)

This code is open source under the MIT license. See LICENSE for complete details.

Assumptions
-----------

The following things are assumed to be true in this documentation.

* You are running OSX.
* You are using Python 2.7. (Probably the version that came OSX.)
* You have [virtualenv](https://pypi.python.org/pypi/virtualenv) and [virtualenvwrapper](https://pypi.python.org/pypi/virtualenvwrapper) installed and working.
* You have NPR's AWS and other credentials stored as environment variables locally.

For more details on the technology stack used with the app-template, see our [development environment blog post](http://blog.apps.npr.org/2013/06/06/how-to-setup-a-developers-environment.html).

What's in here?
---------------

The project contains the following folders and important files:

* ``confs`` -- Server configuration files for nginx and uwsgi. Edit the templates then ``fab <ENV> servers.render_confs``, don't edit anything in ``confs/rendered`` directly.
* ``data`` -- Data files, such as those used to generate HTML.
* ``fabfile`` -- [Fabric](http://docs.fabfile.org/en/latest/) commands for automating setup, deployment, data processing, etc.
* ``etc`` -- Miscellaneous scripts and metadata for project bootstrapping.
* ``jst`` -- Javascript ([Underscore.js](http://documentcloud.github.com/underscore/#template)) templates.
* ``less`` -- [LESS](http://lesscss.org/) files, will be compiled to CSS and concatenated for deployment.
* ``templates`` -- HTML ([Jinja2](http://jinja.pocoo.org/docs/)) templates, to be compiled locally.
* ``tests`` -- Python unit tests.
* ``www`` -- Static and compiled assets to be deployed. (a.k.a. "the output")
* ``www/assets`` -- A symlink to an S3 bucket containing binary assets (images, audio).
* ``www/live-data`` -- "Live" data deployed to S3 via cron jobs or other mechanisms. (Not deployed with the rest of the project.)
* ``www/test`` -- Javascript tests and supporting files.
* ``app.py`` -- A [Flask](http://flask.pocoo.org/) app for rendering the project locally.
* ``app_config.py`` -- Global project configuration for scripts, deployment, etc.
* ``copytext.py`` -- Code supporting the [Editing workflow](#editing-workflow)
* ``crontab`` -- Cron jobs to be installed as part of the project.
* ``public_app.py`` -- A [Flask](http://flask.pocoo.org/) app for running server-side code.
* ``render_utils.py`` -- Code supporting template rendering.
* ``requirements.txt`` -- Python requirements.
* ``static.py`` -- Static Flask views used in both ``app.py`` and ``public_app.py``.

Project lifecycle
-----------------

TODO: Since this is an annual project, document how to get this project up and running each year.  Give some indication of when these steps should happen, e.g. "one month before launch".

* Copy project code?  TODO: Define whether this means copying files or forking the repo or something else.
* Push copy to a new GitHub repository.
* Create a new version of the copy spreadsheet.
* Create a new version of the books spreadsheet.
* Create a testing version of the books spreadsheet that includes ISBN values for book records.
* ???

Bootstrap the project
---------------------

Node.js is required for the static asset pipeline. If you don't already have it, get it like this:

```
brew install node
curl https://npmjs.org/install.sh | sh
```

Then bootstrap the project:

```
cd books17
mkvirtualenv --no-site-packages books17
pip install -r requirements.txt
npm install
```

**Problems installing requirements?** You may need to run the pip command as ``ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future pip install -r requirements.txt`` to work around an issue with OSX.

Next, we will download and parse the text used for copy editing. If we don't do this, subsequent update commands fail:

```
fab text.update
```

After that we will need to download and parse our book list and covers:

```
fab data.update
```

_Note: The `DATA_GOOGLE\_DOC\_KEY` spreadsheet pointed in `app\_config.py` has to be published as a csv in order for the above command to work_

_Note: In order for cover images to be loaded, each book record has to have an ISBN value.  The books team enters this data, but doesn't get to it until later in the process.  If you want to test out the rig, you'll probably want to make a test version of the spreadsheet that includes more complete data, including ISBNs.  You can specify the test spreadsheet key by overriding the value of `DATA\_GOOGLE\_DOC\_KEY` in a `local\_settings` module._

Once that has been run, then we can do a full update that will sync also fonts & copy:

```
fab update
```

Hide project secrets
--------------------

Project secrets should **never** be stored in ``app_config.py`` or anywhere else in the repository. They will be leaked to the client if you do. Instead, always store passwords, keys, etc. in environment variables and document that they are needed here in the README.

Save media assets
-----------------

Large media assets (images, videos, audio) are synced with an Amazon S3 bucket specified in ``app_config.ASSETS_S3_BUCKET`` in a folder with the name of the project. (This bucket should not be the same as any of your ``app_config.PRODUCTION_S3_BUCKETS`` or ``app_config.STAGING_S3_BUCKETS``.) This allows everyone who works on the project to access these assets without storing them in the repo, giving us faster clone times and the ability to open source our work.

Syncing these assets requires running a couple different commands at the right times. When you create new assets or make changes to current assets that need to get uploaded to the server, run ```fab assets.sync```. This will do a few things:

* If there is an asset on S3 that does not exist on your local filesystem it will be downloaded.
* If there is an asset on that exists on your local filesystem but not on S3, you will be prompted to either upload (type "u") OR delete (type "d") your local copy.
* You can also upload all local files (type "la") or delete all local files (type "da"). Type "c" to cancel if you aren't sure what to do.
* If both you and the server have an asset and they are the same, it will be skipped.
* If both you and the server have an asset and they are different, you will be prompted to take either the remote version (type "r") or the local version (type "l").
* You can also take all remote versions (type "ra") or all local versions (type "la"). Type "c" to cancel if you aren't sure what to do.

Unfortunantely, there is no automatic way to know when a file has been intentionally deleted from the server or your local directory. When you want to simultaneously remove a file from the server and your local environment (i.e. it is not needed in the project any longer), run ```fab assets.rm:"www/assets/file_name_here.jpg"```

Adding a page to the site
-------------------------

A site can have any number of rendered pages, each with a corresponding template and view. To create a new one:

* Add a template to the ``templates`` directory. Ensure it extends ``_base.html``.
* Add a corresponding view function to ``app.py``. Decorate it with a route to the page name, i.e. ``@app.route('/filename.html')``
* By convention only views that end with ``.html`` and do not start with ``_``  will automatically be rendered when you call ``fab render``.

Run the project
---------------

A flask app is used to run the project locally. It will automatically recompile templates and assets on demand.

```
workon books17
fab app
```

Visit [localhost:8000](http://localhost:8000) in your browser.

COPY configuration
------------------

This app uses a Google Spreadsheet for a simple key/value store that provides an editing workflow.

To access the Google doc, you'll need to create a Google API project via the [Google developer console](http://console.developers.google.com).

Enable the Drive API for your project and create a "web application" client ID.

For the redirect URIs use:

* `http://localhost:8000/authenticate/`
* `http://127.0.0.1:8000/authenticate`
* `http://localhost:8888/authenticate/`
* `http://127.0.0.1:8888/authenticate`

For the Javascript origins use:

* `http://localhost:8000`
* `http://127.0.0.1:8000`
* `http://localhost:8888`
* `http://127.0.0.1:8888`

You'll also need to set some environment variables:

```
export GOOGLE_OAUTH_CLIENT_ID="something-something.apps.googleusercontent.com"
export GOOGLE_OAUTH_CONSUMER_SECRET="bIgLonGStringOfCharacT3rs"
export AUTHOMATIC_SALT="jAmOnYourKeyBoaRd"
```

Note that `AUTHOMATIC_SALT` can be set to any random string. It's just cryptographic salt for the authentication library we use.

Once set up, run `fab app` and visit `http://localhost:8000` in your browser. If authentication is not configured, you'll be asked to allow the application for read-only access to Google drive, the account profile, and offline access on behalf of one of your Google accounts. This should be a one-time operation across all app-template projects.

It is possible to grant access to other accounts on a per-project basis by changing `GOOGLE_OAUTH_CREDENTIALS_PATH` in `app_config.py`.

COPY editing
------------

This app uses a Google Spreadsheet for a simple key/value store that provides an editing workflow.

View the [sample copy spreadsheet](https://docs.google.com/spreadsheet/pub?key=0AlXMOHKxzQVRdHZuX1UycXplRlBfLVB0UVNldHJYZmc#gid=0).

This document is specified in ``app_config`` with the variable ``COPY_GOOGLE_DOC_KEY``. To use your own spreadsheet, change this value to reflect your document's key (found in the Google Docs URL after ``&key=``).

A few things to note:

* If there is a column called ``key``, there is expected to be a column called ``value`` and rows will be accessed in templates as key/value pairs
* Rows may also be accessed in templates by row index using iterators (see below)
* You may have any number of worksheets
* This document must be "published to the web" using Google Docs' interface

The app template is outfitted with a few ``fab`` utility functions that make pulling changes and updating your local data easy.

To update the latest document, simply run:

```
fab text.update
```

Note: ``text.update`` runs automatically whenever ``fab render`` is called.

At the template level, Jinja maintains a ``COPY`` object that you can use to access your values in the templates. Using our example sheet, to use the ``byline`` key in ``templates/index.html``:

```
{{ COPY.attribution.byline }}
```

More generally, you can access anything defined in your Google Doc like so:

```
{{ COPY.sheet_name.key_name }}
```

You may also access rows using iterators. In this case, the column headers of the spreadsheet become keys and the row cells values. For example:

```
{% for row in COPY.sheet_name %}
{{ row.column_one_header }}
{{ row.column_two_header }}
{% endfor %}
```

When naming keys in the COPY document, pleaseattempt to group them by common prefixes and order them by appearance on the page. For instance:

```
title
byline
about_header
about_body
about_url
download_label
download_url
```

Load books and covers
---------------------

To run the app, you'll need to load books and covers from a Google Spreadsheet.
First, see `DATA_GOOGLE_DOC_KEY` in `app_config.py`.

View a [sample data spreadsheet](https://docs.google.com/spreadsheets/d/10XgwGi631PgWrKwMp_OcFnVa7m4v9LXfwTDB5aXNUDw/edit?usp=sharing)


In order to get the covers for our books we are using an external service from BAKER & TAYLOR, in order to use it you will need your own credentials stored in these environment variables.

```
books17_BAKER_TAYLOR_USERID="SampleUser"
books17_BAKER_TAYLOR_PASSWORD="SamplePassword"
```

Then run the loader:

```
fab data.load_books
fab data.load_images
```

Alternatively, you can update copy and social media along with books with a
single command:

```
fab update
```

Arbitrary Google Docs
----------------------
Sometimes, our projects need to read data from a Google Doc that's not involved with the COPY rig. In this case, we've got a class for you to download and parse an arbitrary Google Doc to a CSV.

This solution will download the uncached version of the document, unlike those methods which use the "publish to the Web" functionality baked into Google Docs. Published versions can take up to 15 minutes up update!

First, export a valid Google username (email address) and password to your environment.

```
export APPS_GOOGLE_EMAIL=foo@gmail.com
export APPS_GOOGLE_PASS=MyPaSsW0rd1!
```

Then, you can load up the `GoogleDoc` class in `etc/gdocs.py` to handle the task of authenticating and downloading your Google Doc.

Here's an example of what you might do:

```
import csv

from etc.gdoc import GoogleDoc

def read_my_google_doc():
    doc = {}
    doc['key'] = '0ArVJ2rZZnZpDdEFxUlY5eDBDN1NCSG55ZXNvTnlyWnc'
    doc['gid'] = '4'
    doc['file_format'] = 'csv'
    doc['file_name'] = 'gdoc_%s.%s' % (doc['key'], doc['file_format'])

    g = GoogleDoc(**doc)
    g.get_auth()
    g.get_document()

    with open('data/%s' % doc['file_name'], 'wb') as readfile:
        csv_file = list(csv.DictReader(readfile))

    for line_number, row in enumerate(csv_file):
        print line_number, row

read_my_google_doc()
```

Google documents will be downloaded to `data/gdoc.csv` by default.

You can pass the class many keyword arguments if you'd like; here's what you can change:
* gid AKA the sheet number
* key AKA the Google Docs document ID
* file_format (xlsx, csv, json)
* file_name (to download to)

See `etc/gdocs.py` for more documentation.

Run Python tests
----------------

Python unit tests are stored in the ``tests`` directory. Run them with ``fab tests``.

Run Javascript tests
--------------------

With the project running, visit [localhost:8000/test/SpecRunner.html](http://localhost:8000/test/SpecRunner.html).

Compile static assets
---------------------

Compile LESS to CSS, compile javascript templates to Javascript and minify all assets:

```
workon books17
fab render
```

(This is done automatically whenever you deploy to S3.)

Test the rendered app
---------------------

If you want to test the app once you've rendered it out, just use the Python webserver:

```
cd www
python -m SimpleHTTPServer
```

Deploy to S3
------------

```
fab staging deploy
```

If you have already loaded books and cover images, you can skip this time-consuming step when
deploying by running:

```
fab staging deploy:quick
```


Analytics
---------

The Google Analytics events tracked in this application are:

|Category|Action|Label|Value|Notes|
|--------|------|-----|-----|-----|
|best-books-2017|tweet|`location`|||
|best-books-2017|facebook|`location`|||
|best-books-2017|pinterest|`location`|||
|best-books-2017|email|`location`|||
|best-books-2017|open-share-discuss|||
|best-books-2017|close-share-discuss|||
|best-books-2017|summary-copied|||
|best-books-2017|view-review|`book_slug`|||
|best-books-2017|navigate|`next` or `previous`|||
|best-books-2017|toggle-view|`list` or `grid`|||
|best-books-2017|clear-tags||||
|best-books-2017|selected-tags|`comma separated list of tags`|||
|best-books-2017|library|`book_slug`||Book slug of library click|
|best-books-2017|amazon|`book_slug`||Book slug of amazon click|
|best-books-2017|ibooks|`book_slug`||Book slug of ibooks click|
|best-books-2017|indiebound|`book_slug`||Book slug of indiebound click|
