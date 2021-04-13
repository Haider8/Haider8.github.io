---
layout: post
title: Ditching pip install for poetry install
date: 2021-04-13 21:00
summary: Complete guide for migrating pip project to poetry
categories: python
---

The following is a complete guide for migrating your existing project to [poetry](https://python-poetry.org/).

Most people who write python code are still using pip for python packaging and dependency installation and up until 5th April I was too, but after that day I migrated to Poetry. Why? Because of this error/warning from pip:

```
ERROR: requests 2.25.1 has requirement idna<3,>=2.5, but you'll have idna 3.1 which is incompatible.
```

So, as you can see barebones pip is not capable of resolving dependencies by itself. In my project [oscarine-api](https://github.com/oscarine/oscarine-api), both `requests`, as well as `email-validator` package, has the required dependency of another package `idna` but `requests` needed `idna >=2.5,<3` and `email-validator` needed `idna >=2.0.0`. 

And finally, here comes Poetry, Python packaging and dependency management made easy. I'll be showing in the rest of the article how I completely remove pip from my existing project and set up Poetry for a local project,  Heroku deployment, and CircleCI tests. If you're curious, you can check the [before](https://github.com/oscarine/oscarine-api/tree/c9d3729f22094b3a2dceb5d7efffff614e4c37e4) and [after](https://github.com/oscarine/oscarine-api/tree/35d039078d3d90b08ad5ba5cfcd4e315e56ba8f2) on GitHub.

[Poery Installation Link](https://python-poetry.org/docs/#installation)

## Local Project Changes

This is the part in which we are going to simply delete our `requirements.txt` file and use the `pyproject.toml` file for everything related to dependencies.

So in my case, I deleted my `requirements.txt` file which I was using for the Heroku Python buildpack and also deleted the `requirements` directory in which I've declared my dev, tests, and production dependencies. After that, I run the `poetry init` command which will create the `pyproject.toml` file if not created already and will also help us in filling the basic info related to our project like project name, version, description et cetera. Now what can we do is, declare our dependencies in `[tool.poetry.dependencies]` and `[tool.poetry.dev-dependencies]` sections. In dev-dependencies section you can declare your dependencies which are not needed in production environment like for example, black, isort and also testing dependencies like pytest can come in this section. Your `pyproject.toml` file should now look something like this:

```
[tool.poetry]
name = "oscarine-api"
version = "0.1.0"
description = ""
authors = ["Haider Ali <haider.lee23@gmail.com>"]
license = "MIT"

[tool.poetry.dependencies]
python = "3.8.5"
fastapi = "0.63.0"
uvicorn = {extras = ["standard"], version = "0.13.4"}
SQLAlchemy = "<1.4.0"
alembic = "1.5.8"

[tool.poetry.dev-dependencies]
black = "20.8b1"
isort = "5.8.0"
# For testing
pytest = "6.2.3"
requests = "2.25.1"

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

And finally, we can run the `poetry install` command but before running it make sure your virtual environment is deactivated and then also delete your virtual environment directory. In my case, I deleted `venv` directory. Now we can strike! `poetry install` will install all your dependencies while resolving the nested dependencies and saving all that info to `poetry.lock` file. You will commit this lock file to git (or any other VCS) so that every developer who is working on this project can have the exact same version of a particular dependency as you have.

## Heroku Deployment Changes

Heroku doesn't understand our dependencies declared in the `pyproject.toml` file since it does not have Poetry support as of now. You can track the current status [here](https://github.com/heroku/heroku-buildpack-python/issues/796). So how can we deploy our app and declare dependencies? By creating a `requirements.txt` file from `pyproject.toml` and for that, I will be using an unofficial Poetry buildback [python-poetry-buildpack](https://github.com/moneymeets/python-poetry-buildpack). Before using this buildpack, please make sure you have deleted your `runtime.txt` file (if you have one), because we will now be decalaring our Python version in  `pyproject.toml` and python-poetry-buildpack will automatically generate this `runtime.txt` file when we push our app to Heroku. Now, quickly run these command using Heroku CLI 

```
heroku buildpacks:clear
heroku buildpacks:add https://github.com/moneymeets/python-poetry-buildpack.git
heroku buildpacks:add heroku/python
```

As you can see above, we have first cleared all the buildpacks because we want `python-poetry-buildpack` to come before `heroku/python` buildpack. Otherwise, we will not be able to deploy or app. You should now be able to deploy your app! If you still face any errors, make sure you have deleted your `requirements.txt` file also.

## CircleCI Changes

This part was very simple I just needed to change `pip install -r requirements/tests.txt` to `poetry install` and that's all! 

In case you are caching dependency modules, you will need to change your checksum to use `poetry.lock` for generating the hash, because `poetry.lock` will never change until you change any of the project dependencies. So now my caching part will look something like this:

{% raw %}
```
- restore_cache:
          keys:
            - v1.1-dependencies-{{ checksum "poetry.lock" }}
            # fallback to using the latest cache if no exact match is found
            - v1.1-dependencies-
```
{% endraw %}

{% raw %}
```
- save_cache:
          paths:
            - ./venv
          key: v1.1-dependencies-{{ checksum "poetry.lock" }}
```
{% endraw %}

Please make sure your Python docker image version declared in CircleCI config is same as that of `pyproject.toml` file. That was it!

I hope it helps and if not, [say hello](/contact/) and please write down your query!

Thank you for reading.

---
