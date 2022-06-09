---
title: "Getting Started With Playwright and Python"
description: small playwright project with python
date: 2022-06-09T14:16:13+01:00
draft: false
toc: true
image: ""
tags: ["python", "playwright", "test automation"]
categories: ["experiences"]
---

# Getting Started with Playwright and Python

With Playwright being more and more accepted in the _world of test automation_ and python a great programming language to create automation I've decided to give it a go to create a demo project using Playwright and Python and in the process documenting my experiment in this article.

Some of what is found in the article can also be found throughout [Playwright documentation](https://playwright.dev/python/docs/intro), although in a bit more condensed manner.

## Software versions
The following software and libs where used:

- Python 3.9 (minimum requirement for playwright is >= python 3.7)
- Pip
- Playwright 1.22 for python
- Pytest - Playwright pytest plugin
- PyCharm Community Edition

## Installing dependencies

As in any other python project, it is a good practice to create a _virtual environment_ (venv) before start installing any dependency.
Since I was using PyCharm as soon as I created the project I could create the virtual env directly but if you are reusing an old project or your project don't have a venv yet, and you're working with PyCharm you can follow their [tutorial](https://www.jetbrains.com/help/pycharm/creating-virtual-environment.html#python_create_virtual_env) on how to create a virtual environment.

In the case of other IDE is being used, or the use of CLI is preferred consider checking the documentation on the topic [here](https://virtualenv.pypa.io/en/stable/)

After creating the virtual environment, activate it and upgrade pip to its latest version.

```shell
source ./<the_venv_name>/bin/activate

pip install --upgrade pip
```

For windows, follow [this article](https://mothergeo-py.readthedocs.io/en/latest/development/how-to/venv-win.html) that will help to create and activate the virtual environment.

In the project root folder create a requirements.txt file with the following contents. I'm not forcing any versions of the dependencies but if wanted this can be done as explained [here](https://realpython.com/lessons/using-requirement-files/#:~:text=A%20Beginner's%20Guide%20to%20Pip&text=A%20requirements%20file%20is%20a,current%20projects%20dependencies%20to%20stdout%20.)

This will allow you to create the project with all its dependencies in other environment. (e.g. CI pipeline)

```text
### Project Dependencies ###

pytest
playwright
pytest-playwright
pytest-parallel
pytest-html
```

After saving the requirements file it is possible to install the project dependencies using `pip`. For that open a terminal in the project root folder and type:

```shell
pip install -r requirements.txt
```

After the project dependencies are installed it is time to install the Playwright browsers (`chromium`, `firefox` and `webkit`) using the following command in the CLI.

```shell
playwright install
```

For non-default options to browser installation the documentation can be found [here](https://playwright.dev/python/docs/browsers)

## Creating the Automation

There are some different options described both on the Playwright documentation page and throughout the web. 
I will center the article on the usage of the Playwright synchronous python API and the creation of `Page Objects` that will be used to implement the tests.

For this demo I will use one of the common web pages for automation training - https://www.saucedemo.com/ 

### Project structure

Since this is a simple project the folder structure will also be simple. Below I represent the most important files and folders of this structure

```text
project-root/
├─ data_model/
├─ page_objects/
├─ tests/
├─ utils/
├─ .gitignore
├─ pytest.ini
├─ README.md
```

### Implementation

For this demo it will only be created one very simple class that will allow to create a User objects that can be used in different login tests

```python
from dataclasses import dataclass, asdict

@dataclass
class User:
    username: str
    password: str
```

Following the Playwright [documentation](https://playwright.dev/python/docs/pom), for the landing page (login) the following Page Object class can be created.

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

And for catalog a similar class can be created. For this demo this class will be incomplete since it is where you can do a big part of the actions (if it was a real shop) so the same class would have more methods.

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

Since the site allow us to use some distinct users it is possible to create a couple simple tests. This will be quite useful ahead since will allow us to create concurrent python processes creating the possibility of running the tests faster.

```python
import pytest

from model.user import User
from pages.sauce_demo_pages.catalog_page import CatalogPage
from pages.sauce_demo_pages.login_page import LoginPage
from playwright.sync_api import Page


@pytest.fixture
def login_page(page: Page):
    login_page = LoginPage(page)
    yield login_page


@pytest.fixture
def catalog_page(page: Page):
    catalog_page = CatalogPage(page)
    yield catalog_page


def test_valid_user_login(login_page, catalog_page):
    valid_user = User("standard_user", "secret_sauce")
    login_page.goto_login_page()
    login_page.is_loaded()
    login_page.do_login_with(valid_user)
    catalog_page.is_loaded()
    catalog_page.validate_title_is("Products")


def test_invalid_user_login(login_page):
    invalid_user = User("invalid_user", "secret_sauce")
    login_page.goto_login_page()
    login_page.is_loaded()
    login_page.do_login_with(invalid_user)
    login_page.validate_error_message_equals("Epic sadface: Username and password do not match any user in this service")


def test_locked_user_login(login_page):
    locked_user = User("locked_out_user", "secret_sauce")
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
To enable this it is only needed to add the option to the _pytest.ini addopts_ as displayed below.

```text
[pytest]
addopts = --base-url https://www.saucedemo.com/ --headed --browser chromium --browser firefox --browser webkit --screenshot only-on-failure --html=target/reports/report.html --workers auto
```

For this simple demo the following times were achieved:

in total 9 test (3 test in 3 browser)

headed mode:
- sequential time:  14.29s
- concurrent time (3 distinct processes): 10.24s
- concurrent time (auto): 8.72s

headless mode:
- sequential time:  10.68s
- concurrent time (3 distinct processes): 7.04s
- concurrent time (auto): 6.00s

This means around **39%** less time when running with the `auto` flag enabled in headed mode and around **44%** when running in headless mode.

Other good particularity of _python-parallel_ is the synchronization of results is performed internally, so the test report is not affected.

## Other Links and References

### References
- dataclasses: https://realpython.com/python-data-classes/

### Links
- pytest-parallel: https://pypi.org/project/pytest-parallel/
- pytest-html: https://pypi.org/project/pytest-html/
- ascii tree generator: https://ascii-tree-generator.com/
