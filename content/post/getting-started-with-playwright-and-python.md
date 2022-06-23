---
title: "Getting Started With Playwright using Python"
description: small playwright project with python
date: 2022-06-09T14:16:13+01:00
draft: false
toc: true
image: ""
tags: ["python", "playwright", "test automation"]
categories: ["experiences"]
---

# Getting Started with Playwright using Python

With Playwright already being a mainstream choice in the _test automation world_ and python a great programming language and create automation I've decided to give it a go to create a demo project using Playwright and Python and in the process documenting it in this article.

## Main libraries used
The following software and libs where used:

- Python 3.9 (minimum requirement for playwright is >= python 3.7)
- Playwright 1.22 for python
- Pytest - Playwright pytest plugin

## Installing dependencies

As in any other python project, it is a good practice to create a _virtual environment_ before installing any dependency.
Since I was using PyCharm as soon as I created the project I allowed me also to create the virtual environment directly. In case of an old project that doesn't have yet a virtual environment, using the same IDE its possible to create the same environment following their [tutorial](https://www.jetbrains.com/help/pycharm/creating-virtual-environment.html#python_create_virtual_env).

In the case of using other IDE or creating the virtual environment for the first time making use of the CLI one of the resources that explains it can be found [here](https://virtualenv.pypa.io/en/stable/)

After creating the virtual environment, activate it and upgrade pip to its latest version.

```shell
source ./<the_venv_name>/bin/activate

pip install --upgrade pip
```

In the project root folder create a requirements.txt file with the following contents. _(I'm not forcing any versions of the dependencies but if wanted this can be done as explained [here](https://realpython.com/lessons/using-requirement-files/#:~:text=A%20Beginner's%20Guide%20to%20Pip&text=A%20requirements%20file%20is%20a,current%20projects%20dependencies%20to%20stdout%20.))_

This will allow to create the project with all its dependencies in other environment. (e.g. CI pipeline)

```text
### Project Dependencies ###

pytest
pytest-playwright
pytest-parallel
pytest-html
playwright
PyYAML
dynaconf
flake8
```

After saving the requirements file it is possible to install the project dependencies using `pip` by using the follwing command.

```shell
pip install -r requirements.txt
```

After the project dependencies are installed it is time to install the Playwright browsers (`chromium`, `firefox` and `webkit`) by typing the command

```shell
playwright install
```

For non-default options to browser installation the documentation can be found [here](https://playwright.dev/python/docs/browsers)

## Creating the Automation

There are some different options described both on the Playwright documentation page and throughout the web. 
I will center the article on the usage of the Playwright with python synchronous API and the creation of `Page Objects` that will be used later to implement the tests.

For this demo I will use one of the common web pages for automation training - https://www.saucedemo.com/ 

### Project main structure

Since this is a simple project the folder structure will also be simple. Below I represent the most important files and folders of this structure

```text
playwright-test-automation/
├─ .github/
│  ├─ workflows/
├─ data_model/
├─ page_objects/
├─ tests/
├─ utils/
├─ .secrets.yaml
├─ config.py
├─ pytest.ini
├─ requirements.txt
├─ settings.yaml
```

### Implementation

To mimic multiple environments configurations I've used [dynaconf](https://www.dynaconf.com/) as the configuration manager. To set it up there is the need run the command

```shell
dynaconf init -f yaml
```

The above command will create the files `.secrets.yaml`, `settings.yaml`, `config.py`.

Since the use of multiple environments is not enable by default, to use this feature there is the need to add the flag `environments=True` to the `config.py` file as displayed below.

```python
from dynaconf import Dynaconf

settings = Dynaconf(
    envvar_prefix="DYNACONF",
    settings_files=['settings.yaml', '.secrets.yaml'],
    # To add multiple environments
    environments=True,
)
```

Since I'm only mimic multiple environments and not in fact having multiple environments I will preserve the same setting for each environment, which in a real case would be different. I can achieve the said configuration using the `settings.yaml` and `.secrets.yaml` files. This will allow for each person that uses a _real_ project to have their own set of configurations.

```yaml
# settings.yaml
qa:
  users:
    valid_user: "standard_user"
    invalid_user: "invalid_user"
    locked_user: "locked_out_user"
  reports_path: "target/reports"

preprod:
  users:
    valid_user: "standard_user"
    invalid_user: "invalid_user"
    locked_user: "locked_out_user"
  reports_path: "target/reports"
```

```yaml
# .secrets.yaml
qa:
  passwords:
    valid_user: "secret_sauce"
    invalid_user: "secret_sauce"
    locked_user: "secret_sauce"
preprod:
  passwords:
    valid_user: "secret_sauce"
    invalid_user: "secret_sauce"
    locked_user: "secret_sauce"
```

As I will talk further ahead the `.secrets.yaml` should only be used as a local file or if used in a CI workflow should be treated as very sensitive data having a complete lifecycle of _creation_ and _clean up_ during the same workflow.

There is one final step needed to be able to change the environment and one way is adding a new argument to the pytest that can after be provided using the CLI. To achieve this I've added to the `conftest.py` the following content.

```python
import os
import pytest
from datetime import datetime
from config import settings


def pytest_addoption(parser):
    parser.addoption("--env", action="store", default="qa")


@pytest.fixture(scope='session', autouse=True)
def config_loader(pytestconfig):
    settings.configure(FORCE_ENV_FOR_DYNACONF=pytestconfig.getoption("env"))
```

This will enable to run of the test automation by default using the _QA_ environment but by adding the `--env=<desired_value>` argument it is possible to assign a diffent environment.

It is now time to start creating the data classes that it will be used for the automation. For the purpose of this demo I've chosen to create one very simple data class that will allow to create _User objects_ that can be used in different login tests.

```python
from dataclasses import dataclass

@dataclass
class User:
    username: str
    password: str
```

For page objects I've followed what is stated in the Playwright [documentation](https://playwright.dev/python/docs/pom), so for the landing page (login) the following Page Object class can be created.

```python
from playwright.sync_api import Page, expect
from model.user import User


class LoginPage:

    _logo=".login_logo"
    _username_selector = "#user-name"
    _password_selector = "#password"
    _login_button_selector = "#login-button"
    _mascot_image = ".bot_column"

    def __init__(self, page: Page):
        self.page = page

    def is_loaded(self) -> None:
        expect(self.page.locator(self._logo)).to_be_visible()
        expect(self.page.locator(self._username_selector)).to_be_visible()
        expect(self.page.locator(self._password_selector)).to_be_visible()
        expect(self.page.locator(self._login_button_selector)).to_be_visible()
        expect(self.page.locator(self._mascot_image)).to_be_visible()

    def goto_login_page(self) -> None:
        self.page.goto("/")

    def do_login_with(self, user: User) -> None:
        self.page.locator(self._username_selector).click()
        self.page.locator(self._username_selector).type(user.username)
        self.page.locator(self._password_selector).click()
        self.page.locator(self._password_selector).type(user.password)
        self.page.locator(self._login_button_selector).click()
```

And for catalog a similar class can be also created. For the purpose of this demo the page object class displayed below will be incomplete since most of the actions that mimic a possible user behavior will not be added to this class.

```python
from playwright.sync_api import Page, expect
import logging

class CatalogPage:

    _title = ".title"
    _grid_products = "#inventory_container"
    _item = ".inventory_item"
    _item_name = ".inventory_item_name"

    def __init__(self, page: Page):
        self.page = page

    def is_loaded(self):
        expect(self.page.locator(self._title)).to_be_visible()
        expect(self.page.locator(self._grid_products).nth(0)).to_be_visible()
        expect(self.page.locator(self._item)).to_have_count(6)

    def validate_item_nr_text_is(self, index: int, text: str) -> None:

        expect(self.page.locator(self._item_name).nth(index)).to_contain_text(text)

    def validate_title_is(self, text: str) -> None:

        expect(self.page.locator(self._title)).to_have_text(text)
    
    (...)
```

Since the site allow us to use some distinct users it is possible to create a couple simple tests. This will be quite useful since will allow us to create concurrent python processes making it possible to run tests faster.

```python
import pytest

from model.user import User
from pages.sauce_demo_pages.catalog_page import CatalogPage
from pages.sauce_demo_pages.login_page import LoginPage
from playwright.sync_api import Page
from config import settings


@pytest.fixture
def login_page(page: Page):
    login_page = LoginPage(page)
    yield login_page


@pytest.fixture
def catalog_page(page: Page):
    catalog_page = CatalogPage(page)
    yield catalog_page


def test_valid_user_login(login_page, catalog_page):
    valid_user = User(settings.users.valid_user, settings.passwords.valid_user)
    login_page.goto_login_page()
    login_page.is_loaded()
    login_page.do_login_with(valid_user)
    catalog_page.is_loaded()
    catalog_page.validate_title_is("Products")


def test_invalid_user_login(login_page):
    user = User(settings.users.invalid_user, settings.passwords.invalid_user)
    login_page.goto_login_page()
    login_page.is_loaded()
    login_page.do_login_with(invalid_user)
    login_page.validate_error_message_equals("Epic sadface: Username and password do not match any user in this service")


def test_locked_user_login(login_page):
    user = User(settings.users.locked_user, settings.passwords.locked_user)
    login_page.goto_login_page()
    login_page.is_loaded()
    login_page.do_login_with(locked_user)
    login_page.validate_error_message_equals("Epic sadface: Sorry, this user has been locked out.")
```

Now that some tests are in place its possible to create the pytest.ini making use of options that playwright-pytest plugin offers. (for all the detail on playwright-pytest plugin consult playwright [documentation](https://playwright.dev/python/docs/test-runners))

```text
[pytest]
addopts = --base-url https://www.saucedemo.com/ --headed --browser chromium --browser firefox --browser webkit --screenshot only-on-failure --html=target/reports/report.html
```

The above options will allow the tests to run in headed mode (by default playwright runs in headless mode) in all the browser playwright offers, saving a screenshot in case a test fails and in the end generating a html report.

Up until where I could experiment, with the use of `pytest-parallel` dependency it is possible to create parallel tests runs. **There is although a very important note from the playwright developers that is Playwright is not [thread-safe](https://playwright.dev/python/docs/intro#threading)** (at least at the time of the writing of this article). To be safer I make use of the `workers` (process) option instead of `tests-per-worker` (threads) option, allowing each test to run in a distinct python process.
To enable this one only needs to append a new option (argument) to the _pytest.ini addopts_ as displayed below.

```text
[pytest]
addopts = --base-url https://www.saucedemo.com/ --headed --browser chromium --browser firefox --browser webkit --screenshot only-on-failure --html=target/reports/report.html --workers auto
```

As a comparison between a single process and concurrent processes for this simple demo the following times were achieved:

in total 9 test (3 test in 3 browser)

headed mode:
- sequential time:  14.29s
- concurrent time (3 distinct processes): 10.24s
- concurrent time (auto): 8.72s

headless mode:
- sequential time:  10.68s
- concurrent time (3 distinct processes): 7.04s
- concurrent time (auto): 6.00s

This means when using concurrency the automation is around **39%** less time when running with the `auto` flag enabled in headed mode and around **44%** when running in headless mode.

Other good particularity of _python-parallel_ library is that the synchronization of results is performed internally, so the test report in the end is not affected.

## CI Workflow

To create a CI workflow I've used Github Actions. Similar workflows can be created for other CI solution providers.

For this demo I've experimented with two workflows with both considering that the environment to be used is _QA_. 
For the first workflow, instead of the usage `.secrets.yaml` file I will provide the contents of the file by using `github secrets` while in the second there will be a small lifecycle around the stated file.

- Workflow considering the usage of github secrets

```yaml
# This workflow will install Python dependencies, run tests and lint with a single version of Python
# For more information see: https://help.github.com/actions/language-and-framework-guides/using-python-with-github-actions

name: Playwright Python UI testing

env:
  REQUIREMENTS_FILENAME: requirements-test.txt
  # Defining the contents of the .secret.yaml file as env variables using github secrets.
  DYNACONF_PASSWORDS.VALID_USER: ${{ secrets.DYNACONF_PASSWORDS_VALID_USER }}
  DYNACONF_PASSWORDS.INVALID_USER: ${{ secrets.DYNACONF_PASSWORDS_INVALID_USER }}
  DYNACONF_PASSWORDS.LOCKED_USER: ${{ secrets.DYNACONF_PASSWORDS_LOCKED_USER }}

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  test_job:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up Python 3.10
      uses: actions/setup-python@v3
      with:
        python-version: "3.10"
    - name: Install dependencies
      run: |
        cd playwright-python-ui
        python -m pip install --upgrade pip
        if [ -f ${{env.REQUIREMENTS_FILENAME}} ]; then pip install -r ${{env.REQUIREMENTS_FILENAME}}; fi
    - name: Ensure browsers are installed
      run: |
        cd playwright-python-ui
        python -m playwright install --with-deps
    - name: Lint with flake8
      run: |
        # stop the build if there are Python syntax errors or undefined names
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

    - name: Test with pytest
      run: |
        cd playwright-python-ui
        pytest tests/test_demo.py
```

- Workflow considering the usage of .secrets.yaml file

```yaml

# Workflow that considers the download of the .secrets.yaml or .env file from a safe repository.

name: Playwright Python UI testing

env:
  REQUIREMENTS_FILENAME: requirements-test.txt

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  test_job:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up Python 3.10
      uses: actions/setup-python@v3
      with:
        python-version: "3.10"
    - name: Install dependencies
      run: |
        cd playwright-python-ui
        python -m pip install --upgrade pip
        if [ -f ${{env.REQUIREMENTS_FILENAME}} ]; then pip install -r ${{env.REQUIREMENTS_FILENAME}}; fi
    - name: Ensure browsers are installed
      run: |
        cd playwright-python-ui
        python -m playwright install --with-deps
    - name: Download .secrets.yaml
      
      # Considering that there was a prior step to retrive of an api token that would allow the download of the .secrets.yaml file securily 
      run: | 
        curl -iL "https://thisURLdoesntexist.com" |grep -A 100 "qa" >> .secrets.yaml
    - name: Lint with flake8
      run: |
        # stop the build if there are Python syntax errors or undefined names
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

    - name: Test with pytest
      run: |
        cd playwright-python-ui
        pwd
        pytest tests/test_demo.py

    - name: Delete .secrets.yaml
      run: |
        echo "Clean up!"
        rm -rf .secrets.yaml
```
## Other Links and References

### Demo project

[GitHub Repo](https://github.com/O2F/qa-frwks-bootstraps/tree/main/playwright-python-ui)
### References
- dataclasses: https://realpython.com/python-data-classes/
- Playwright python documentation: https://playwright.dev/python/docs/intro
### Links
- pytest-parallel: https://pypi.org/project/pytest-parallel/
- pytest-html: https://pypi.org/project/pytest-html/
- ascii tree generator: https://ascii-tree-generator.com/
