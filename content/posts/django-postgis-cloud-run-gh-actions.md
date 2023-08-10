---
title: "Notes: Django w/ PostGIS on GCP Cloud Run and Github Actions"
date: 2023-08-10T17:58:22-04:00
draft: false
tags: ["django", "python", "python3"]
categories: ["software"]
keywords: ["django", "python", "python3", "postgis", "cloud run", "gh actions", "github actions", "github", "gdal", "geodjango", "postgres"]
description: "This a short post about the configuration steps I took to get django running with gdal extension and postgres/postgis on cloud run and github actions (for testing). Pretty straight forward, but I had to iteratae a few times to get it right so hopefully this helps someone else."
---
*TL;DR: this a short post about the configuration steps I took to get django
running with the GDAL extension and postgreSQL/postGIS backend. These notes
are what it took to get it deployed on GCP Cloud Run, and to use Github Actions
to run automated tests on push or pull request. Pretty straight forward, but I
had to iterate a few times to get it right so hopefully this helps someone
else.*

# Notes from Test and Deploy Django with PostGIS
Running a Django app in a Docker container is pretty straightforward, but some
minor extra configuration is required to install the dependencies when using
a Postgres backend with the PostGIS extension to support storing GIS data. I 
recently set up a Django REST API server on Google Cloud Platform's Cloud Run
to deploy continuously from Github. I used Github Actions to run automated
tests on push or pull request as well. Both of these setups required some
iteration on the configuration to get right, so I thought I'd share. This is
not meant to be a long detailed post, I know it’s a pretty specific application
but I hope it helps someone in the same boat as I was.

# Deploying on GCP Cloud Run
Cloud Run demonstrates how to set up your project to build the image, push it
to a cloud container registry, and then deploy it on Cloud Run in their docs.
We can use pretty much exactly the procedure that their
[documentation describes.](https://cloud.google.com/build/docs/deploying-builds/deploy-cloud-run#continuous_deployment)
The part that required a little bit of configuration was installing the
GIS dependencies in the container. Here's the resulting [Dockerfile](https://gist.github.com/heathhenley/87961a8b58fb476bd3ca55f2e3a6ff46):

```dockerfile
# Use an official lightweight Python image.
# https://hub.docker.com/_/python
FROM python:3.10-slim

ENV APP_HOME /app
WORKDIR $APP_HOME


RUN apt-get update && apt-get install -y binutils gdal-bin

# Removes output stream buffering, allowing for more efficient logging
ENV PYTHONUNBUFFERED 1

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# Copy local code to the container image.
COPY . .

RUN python manage.py collectstatic

# Run the web service on container startup. Here we use the gunicorn
# webserver, with one worker process and 8 threads.
# For environments with multiple CPU cores, increase the number of workers
# to be equal to the cores available.
# Timeout is set to 0 to disable the timeouts of the workers to allow Cloud Run to handle instance scaling.
CMD exec gunicorn --bind 0.0.0.0:$PORT --workers 1 --threads 8 --timeout 0 myapp.wsgi:application
```

# Running Automated Testing using Github Actions
Django has a nice testing framework already built in, once you add your unit
tests you can run them with “python manage.py test”. The documentation has some
[great examples.](https://docs.djangoproject.com/en/4.2/topics/testing/overview/)
Once you have your tests written, you probably want to run them on each new 
push or pull request to catch any regressions. In my case, as mentioned above,
we’re using Postgres with the PostGIS extension as our backend. Because Django
makes a complete database for unit testing and then cleans it up, we need to
get postgres running and configured with the PostGIS extension so that Django
can make the testing database and then clean it up on the Github runner. In the
end, we have something like [this](https://gist.github.com/heathhenley/d0227f07461f3eb63fe823d70832b371):

```yaml
name: test_django
on: [pull_request, push]
jobs:
  test_project:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up python 3.10
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"
          cache: "pip"
      - name: Install deps
        run: |
          pip install -r requirements.txt 
          sudo apt-get update
          sudo apt-get install -y binutils gdal-bin
          sudo apt install postgresql-postgis
      - name: Start and initialize postgres service (database, gis)
        run: |
          sudo systemctl start postgresql.service
          sudo su - postgres -c "createdb djangotest"
          sudo su - postgres -c "psql  --echo-all -U postgres -d postgres --command \"ALTER USER postgres PASSWORD 'somepasswordhere';\""
          sudo systemctl restart postgresql.service
          sudo su - postgres -c "psql  --echo-all -U postgres --command \"create extension postgis\""
      - name: Run django tests
        run: python manage.py test
```

The tricky things to figure out were related to creating the DB, adding
password (security not an issue this db is temporary and for testing),
adding the GIS extension, and also making sure that the postgres service was
running on the runner. That is basically everything under the
"Start and initialize postgres services" step in the .yaml file above.

You also need to make sure that the `settings.py` in your Django application has
information about the test database you're going to use. For example, it can get
the credentials from environment variables in production, but default to a set
of test credentials when running tests when those environment variables are not
there.

That’s all I got. Hopefully this helps someone save a few pushes and some time
and / or frustration.

Let me know if you have any questions!

