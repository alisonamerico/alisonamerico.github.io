---
title: Testing GitHub APIs with Pytest - Practice
date: 2026-02-12
draft: false
tags: ["python", "tests", "pytest", "github", "mock", "flask"]
categories: ["testing"]
description: "Complete practical guide to testing GitHub APIs with Pytest. Master unit tests, integration tests, mocking, fixtures, and performance testing with real-world examples. Learn professional testing patterns and build robust test suites for your applications."
cover: testing-with-pytest-practice.png
---

<!-- ![](testing-with-pytest-practice.png) -->

## Introduction

In the [previous post]({{< relref "posts/post-1-testing-with-pytest-fundamentals-best-practices-and-strategy/index.md" >}}), we covered the theoretical foundations of Pytest.

Now we will apply everything in practice using a real example that
consumes the GitHub public API:

- Build a small real-world project
- Consume the public GitHub API
- Apply unit tests
- Apply integration tests
- Use fixtures
- Use mocks
- Use markers
- Apply the AAA pattern
- Use pytest plugins

This guide is intentionally very detailed. Every important line will be
explained so that even beginners can follow.


------------------------------------------------------------------------

## Project Structure

``` text
github_api_tests/
├── app.py                     # Flask web application
├── services/
│   └── github_service.py      # GitHub API service layer
├── tests/
│   ├── conftest.py            # Shared fixtures
│   ├── test_users_unit.py     # Unit tests with mocks
│   ├── test_users_integration.py  # Integration tests (real API)
│   ├── test_parametrize.py    # Parameterized tests
│   ├── test_skip.py           # Skip marker examples
│   ├── test_fail.py           # xfail marker examples
│   ├── test_errors.py         # Error handling tests
│   ├── test_functional.py     # Functional tests (Flask routes)
│   ├── test_performace.py     # Performance benchmarks
│   └── test_regression.py     # Regression tests
├── pytest.ini                 # Pytest configuration
└── requirements.txt           # Project dependencies
```

We organize the project following best practices:

- **`app.py`** - Main Flask application (entry point)
- **`services/`** - Business logic layer (separates concerns)
- **`tests/`** - All test files organized by type
- **`pytest.ini`** - Global test configuration
- **`requirements.txt`** - Dependency management

This architecture provides:
- Clear separation of concerns
- Easy maintenance and scalability
- Industry-standard structure
- Better test organization

---

## Project Setup

Before running the tests, we need to prepare our local development environment properly.

---

### Creating a Virtual Environment

A virtual environment allows us to isolate project dependencies from the global Python installation.

This prevents version conflicts between different projects and ensures reproducibility.

```bash
python -m venv .venv
```


#### What this command does:

`python` → Runs the Python interpreter.

`-m venv` → Executes the built-in venv module.

`.venv` → Creates a new virtual environment folder named `.venv`.

After running this command, a directory called .venv/ will be created containing:

- A local Python interpreter

- Its own `pip`

- An isolated site-packages directory

This ensures that any package installed inside this environment will not affect other projects.

---

### Activating the Virtual Environment

Once created, the virtual environment must be activated:

```bash
source .venv/bin/activate
```

#### What this does:

`source` → Executes the script in the current shell.

`.venv/bin/activate` → Activates the virtual environment.

After activation:

Your terminal prompt usually changes (e.g., `(.venv)` appears).

`python` and `pip` now point to the virtual environment versions.

All installed packages will be isolated inside `.venv`.

---

### Dependencies

All project dependencies are listed in a `requirements.txt` file:

```
flask
requests
pytest
pytest-mock
pytest-cov
pytest-benchmark
```

What each dependency does:

- **flask** → Web framework for building the API endpoints.
- **requests** → HTTP client used to interact with the GitHub API.
- **pytest** → Testing framework.
- **pytest-mock** → Provides integration between pytest and `unittest.mock`.
- **pytest-cov** → Adds test coverage reporting.
- **pytest-benchmark** → Performance testing and benchmarking tools.

---

### Installing Dependencies

To install all required packages:

```bash
pip install -r requirements.txt
```

What this command does:

- `pip` → Python package installer.
- `-r requirements.txt` → Reads the dependency list from the file.
- Installs all listed packages inside the active virtual environment.

---

### Pytest Configuration

The project includes a pytest.ini file to configure pytest behavior globally.

`pytest.ini`

```ini
[pytest]
addopts = -ra -q --cov=app --cov-report=term-missing
markers =
    unit: marks tests as unit tests
    integration: marks tests as integration tests
    slow: marks tests as slow tests
    regression: marks tests as regression tests
```

---

#### Line-by-line explanation

`[pytest]`

Declares that this file contains configuration for pytest.

`addopts = -ra -q --cov=app --cov-report=term-missing`

Defines default command-line options that will always be applied when running `pytest`.

Breaking it down:

- `-r a` → Shows a summary for all test outcomes (passed, skipped, xfailed, etc.).
- `-q` → Runs pytest in quiet mode (less verbose output).
- `--cov=app` → Measures test coverage for the app package.
- `--cov-report=term-missing` → Displays missing lines directly in the terminal coverage report.

This ensures that every test run automatically includes coverage analysis.

`markers =`

Registers custom test markers to avoid warnings like:

```makefile
PytestUnknownMarkWarning: Unknown pytest.mark.integration
```

Each marker must be declared explicitly.

---

`unit`

```ini
unit: marks tests as unit tests
```

Marks tests as unit tests.

These tests:

- Validate small pieces of logic
- Avoid external dependencies
- Run very fast

You can run them using:

```bash
pytest -m unit
```

`integration`

```ini
integration: marks tests as integration tests
```

Marks integration tests.

These tests:

- Validate interaction between components
- May call real services or external APIs
- Are usually slower than unit tests

Run only integration tests:

```bash
pytest -m integration
```

`slow`

```ini
slow: marks tests as slow tests
```

Marks slower tests.

You can exclude them:

```bash
pytest -m "not slow"
```


## Production Code

### Flask Application (app.py)

```python
# app.py

from flask import Flask, jsonify
from services.github_service import fetch_users

app = Flask(__name__)

@app.route('/users')
def github_users():
    try:
        users = fetch_users()
        return jsonify(users)
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(debug=True)
```


### Service Layer (services/github_service.py)

```python
# services/github_service.py

import requests

GITHUB_API_URL = 'https://api.github.com/users'

def fetch_users(per_page=10):
    """
    Fetches users from the public GitHub API.
    """
    response = requests.get(
        GITHUB_API_URL, params={'per_page': per_page}, timeout=5
    )
    response.raise_for_status()
    return response.json()
```


### Production Code (Line-by-Line Explanation)

#### Flask Application (app.py)

``` python
# app.py

from flask import Flask, jsonify
from services.github_service import fetch_users
```

We import:
- `Flask` - Web framework to create our API
- `jsonify` - Flask helper for JSON responses
- `fetch_users` - Service function that handles GitHub API calls

---

``` python
app = Flask(__name__)
```

Creates the Flask application instance.

---

``` python
@app.route('/users')
def github_users():
    try:
        users = fetch_users()
        return jsonify(users)
    except Exception as e:
        return jsonify({"error": str(e)}), 500
```

This is a Flask route with error handling:
- `@app.route('/users')` → Maps HTTP GET `/users` to this function
- `try/except` → Handles API failures gracefully
- `jsonify(users)` → Returns the JSON response to the client
- `return jsonify({"error": str(e)}), 500` → Returns error response when GitHub API fails

---

``` python
if __name__ == '__main__':
    app.run(debug=True)
```

Allows running the app directly with `python app.py`.

#### Service Layer (services/github_service.py)

``` python
# services/github_service.py

import requests

GITHUB_API_URL = 'https://api.github.com/users'
```

- Import requests for HTTP calls
- Define constant for GitHub API base URL

---

``` python
def fetch_users(per_page=10):
    """
    Fetches users from the public GitHub API.
    """
    Fixture que cria um cliente HTTP de teste para o Flask.
    """
    with app.test_client() as client:
        yield client

@pytest.fixture
def sample_username():
    return 'octocat'
```

This fixture provides reusable test data.

### Fixtures (Explanation)

``` python
# tests/conftest.py

import pytest
from app import app
```

We import pytest and the Flask app.

---

#### Flask Test Client Fixture

``` python
@pytest.fixture
def client():
    """
    Fixture that creates HTTP test client for Flask.
    """
    with app.test_client() as client:
        yield client
```

Line-by-line breakdown:
- `@pytest.fixture` → Registers this function as a fixture
- `client()` → Function name becomes injectable
- `app.test_client()` → Creates Flask test client
- `with ... as client:` → Context manager for cleanup
- `yield client` → Provides client to tests, then cleans up

This fixture allows testing Flask routes without starting a real server.

---

#### Sample Data Fixture

``` python
@pytest.fixture
def sample_username():
    return 'octocat'
```

Line-by-line:
- `@pytest.fixture` → Registers as fixture
- `sample_username` → becomes injectable parameter
- `return 'octocat'` → Provides test data

When tests include these parameters:
``` python
def test_example(client, sample_username):
```

Pytest automatically injects both fixtures. This is **dependency injection**.

---

## Unit Test with Mock

```python
# tests/test_users_unit.py

from services.github_service import fetch_users

def test_fetch_users_with_pytest_mock(mocker):
    """
    Same unit test, using pytest-mock.
    """
    # Arrange
    fake_users = [{'login': 'pytest-mock'}]

    mocker.patch(
        'services.github_service.requests.get',
        return_value=mocker.Mock(
            json=lambda: fake_users, raise_for_status=lambda: None
        ),
    )

    # Act
    users = fetch_users()

    # Assert
    assert users == fake_users
```

### Unit Test (Line-by-Line Explanation)

``` python
# tests/test_users_unit.py

from services.github_service import fetch_users
```

We import the service function to test it in isolation.

---

``` python
def test_fetch_users_with_pytest_mock(mocker):
```

- `mocker` comes from pytest-mock plugin
- No explicit marker needed, but could use `@pytest.mark.unit`

---

#### Arrange Section

``` python
fake_users = [{'login': 'pytest-mock'}]
```

Creates expected test data.

---

``` python
mocker.patch(
    'services.github_service.requests.get',
    return_value=mocker.Mock(
        json=lambda: fake_users, 
        raise_for_status=lambda: None
    ),
)
```

This is the mocking setup:
- `mocker.patch()` → Replaces `requests.get` in our service
- `return_value=mocker.Mock()` → Creates mock response
- `json=lambda: fake_users` → Mock method that returns our fake data
- `raise_for_status=lambda: None` → Mock that does nothing (no exception)

**Key point**: No real HTTP call is made!

---

#### Act Section

``` python
users = fetch_users()
```

Calls the service function, which uses the mocked `requests.get`.

---

#### Assert Section

``` python
assert users == fake_users
```

Verifies the service returns exactly what our mock provided.

---

## Integration Test (Real API Call)

```python
# tests/test_users_integration.py

import pytest
from services.github_service import fetch_users

@pytest.mark.integration
@pytest.mark.skip(reason="GitHub API rate limit exceeded - skip for now")
def test_fetch_users_integration():
    """
    Calls REAL GitHub API
    """
    # Act
    users = fetch_users(5)

    # Assert
    assert isinstance(users, list)
    assert len(users) > 0
    assert 'login' in users[0]
```

This test makes a real HTTP call to GitHub.

### Integration Test (Line-by-Line Explanation)

``` python
@pytest.mark.integration
def test_fetch_users_integration():
```

- `integration` marker categorizes it as integration test
- No `slow` marker here, but could be added

---

``` python
users = fetch_users(5)
```

This performs a real request to GitHub API, asking for 5 users.

---

``` python
assert isinstance(users, list)
assert len(users) > 0
assert 'login' in users[0]
```

Multiple assertions validate:
- Response is a list
- List is not empty  
- First user has expected structure

**Important**: This test requires internet connection and can be slow.

---

## Parametrize

```python
# tests/test_parametrize.py

import pytest
from services.github_service import fetch_users

@pytest.mark.parametrize('qty', [1, 3, 5])
def test_fetch_users_parametrize(qty, mocker):
    """
    Same test with multiple values.
    Using mocked requests to avoid rate limiting.
    """
    # Arrange - mock response to avoid rate limiting
    fake_users = [{'login': f'user{i}'} for i in range(qty)]
    
    mocker.patch(
        'services.github_service.requests.get',
        return_value=mocker.Mock(
            json=lambda: fake_users, 
            raise_for_status=lambda: None
        ),
    )
    
    # Act
    users = fetch_users(qty)
    
    # Assert
    assert len(users) <= qty
    assert len(users) == qty  # Should match exactly with our mock
```

Pytest runs this test three times with different quantities.

## Parametrize (Detailed)

``` python
@pytest.mark.parametrize('qty', [1, 3, 5])
```

Pytest will:
- Run the test with `qty = 1`
- Run it with `qty = 3`
- Run it with `qty = 5`

---

``` python
def test_fetch_users_parametrize(qty, mocker):
    fake_users = [{'login': f'user{i}'} for i in range(qty)]
    mocker.patch('services.github_service.requests.get', ...)
    users = fetch_users(qty)
    assert len(users) == qty
```

The parameters are automatically injected into the test function.

This approach:
- Eliminates code duplication
- Tests edge cases (1, 3, 5 users)
- Uses mocking to avoid API rate limiting
- Makes tests more maintainable

**Pro tip**: You can also combine multiple parameters:
```python
@pytest.mark.parametrize('qty,expected', [(1, 1), (5, 5), (100, 100)])
```


---

## Testing Errors

```python
# tests/test_errors.py

import pytest
import requests
from services.github_service import fetch_users

def test_timeout_error(mocker):
    """
    Tests how service handles timeouts.
    """
    # Arrange
    mocker.patch(
        'services.github_service.requests.get',
        side_effect=requests.Timeout("Request timed out")
    )

    # Act & Assert
    with pytest.raises(requests.Timeout):
        fetch_users()

def test_http_error_handling(mocker):
    """
    Tests HTTP error handling.
    """
    # Arrange
    mocker.patch(
        'services.github_service.requests.get',
        side_effect=requests.HTTPError("404 Not Found")
    )

    # Act & Assert
    with pytest.raises(requests.HTTPError):
        fetch_users()
```

This ensures errors are handled correctly.

### Testing Errors (Line-by-Line Explanation)

#### Timeout Error Test

``` python
mocker.patch(
    'services.github_service.requests.get',
    side_effect=requests.Timeout("Request timed out")
)
```

- `side_effect` → Instead of returning, raises exception
- `requests.Timeout` → Specific timeout exception
- Tests how service handles network timeouts

---

``` python
with pytest.raises(requests.Timeout):
    fetch_users()
```

- Asserts that `fetch_users()` raises `Timeout`
- Test passes if correct exception is raised
- Test fails if no exception or wrong exception

#### HTTP Error Test

``` python
side_effect=requests.HTTPError("404 Not Found")
```

Simulates HTTP errors like 404, 500, etc.

**Key benefits**:
- Tests error handling without real failures
- Ensures proper exception propagation
- Verifies service robustness

---

## Skip vs Xfail

```python
# tests/test_skip.py

import pytest

@pytest.mark.skip(reason='Feature under development')
def test_future_feature():
    assert True
```

```python
# tests/test_fail.py

import pytest
from services.github_service import fetch_users

@pytest.mark.xfail(reason='Known bug')
def test_known_bug():
    assert False

@pytest.mark.xfail(reason='Known bug when per_page > 100')
def test_expected_failure():
    fetch_users(200)
```

### Skip vs Xfail (Detailed)

#### Skip Marker

``` python
@pytest.mark.skip(reason='Feature under development')
```

- **Test does not run at all**
- Shows as "skipped" in test report
- Useful for incomplete features
- Saves execution time

#### Xfail Marker

``` python
@pytest.mark.xfail(reason='Known bug')
```

- **Test runs but failure is expected**
- Shows as "xfail" (expected failure) if it fails
- Shows as "xpass" (unexpected pass) if it passes
- Useful for known bugs or edge cases

**When to use each**:

- `@pytest.mark.skip`: Feature not ready, environment issues
- `@pytest.mark.xfail`: Known limitations, expected failures

**Conditional skipping**:
```python
@pytest.mark.skipif(sys.version_info < (3, 8), reason="Requires Python 3.8+")
def test_python_38_feature():
    pass
```

------------------------------------------------------------------------

## Functional Tests (Flask Routes)

```python
# tests/test_functional.py

from http import HTTPStatus

def test_users_page(client):
    """
    Simulates access to Flask route
    """
    # Act
    response = client.get('/users')

    # Assert - check for either successful response or rate limit error
    assert response.status_code in [HTTPStatus.OK, HTTPStatus.INTERNAL_SERVER_ERROR]
    
    # Check if response is valid JSON
    try:
        data = json.loads(response.data)
        if response.status_code == HTTPStatus.OK:
            assert isinstance(data, list)
        else:
            # Should be an error response
            assert isinstance(data, dict)
            assert 'error' in data
    except json.JSONDecodeError:
        assert False, "Response is not valid JSON"
```

### Functional Tests (Line-by-Line Explanation)

``` python
def test_users_page(client):
```

- `client` is the Flask test client fixture
- Tests the web route end-to-end
- Handles both success and error scenarios

------------------------------------------------------------------------

``` python
response = client.get('/users')
```

- Simulates HTTP GET request to `/users`
- No real server needed (test client)

------------------------------------------------------------------------

``` python
assert response.status_code in [HTTPStatus.OK, HTTPStatus.INTERNAL_SERVER_ERROR]
```

- Accepts both success (200) and error (500) responses
- 500 occurs when GitHub API rate limits the request

------------------------------------------------------------------------

``` python
data = json.loads(response.data)
if response.status_code == HTTPStatus.OK:
    assert isinstance(data, list)
else:
    assert isinstance(data, dict)
    assert 'error' in data
```

- Validates that response is always valid JSON
- Success: should be a list of users
- Error: should be a dict with 'error' key

## Performance Tests

```python
# tests/test_performace.py

def test_users_endpoint_performance(benchmark, client):
    """
    Measures response time of /users route.
    """
    benchmark(lambda: client.get('/users'))
```

### Performance Testing (Line-by-Line Explanation)

``` python
def test_users_endpoint_performance(benchmark, client):
```

- `benchmark` fixture from pytest-benchmark
- `client` fixture for Flask testing

------------------------------------------------------------------------

``` python
benchmark(lambda: client.get('/users'))
```

- Measures execution time of the lambda
- Provides detailed performance metrics
- Establishes baseline for future comparisons

**Current benchmark results from the project**:
```
------------------------------------------------------- benchmark: 1 tests ------------------------------------------------------
Name (time in ms)                        Min       Max      Mean  StdDev    Median      IQR  Outliers     OPS  Rounds  Iterations
---------------------------------------------------------------------------------------------------------------------------------
test_users_endpoint_performance     246.0832  264.0037  254.0807  8.1997  252.4519  15.2874       1;0  3.9358       5           1
---------------------------------------------------------------------------------------------------------------------------------

Legend:
  Outliers: 1 Standard Deviation from Mean; 1.5 IQR (InterQuartile Range) from 1st Quartile and 3rd Quartile.
  OPS: Operations Per Second, computed as 1 / Mean
```

**Benchmark metrics explained**:
- **Min**: Fastest execution time observed
- **Max**: Slowest execution time observed  
- **Mean**: Average execution time across all rounds
- **StdDev**: Standard deviation (measure of consistency)
- **Median**: Middle value when sorted (less affected by outliers)
- **IQR**: Interquartile range (middle 50% of values)
- **Outliers**: Values that deviate significantly from the norm
- **OPS**: Operations per second (how many times it could run in one second)
- **Rounds**: Number of test iterations performed
- **Iterations**: Number of executions per round

**Run with**: `pytest --benchmark-only`

## Regression Tests

```python
# tests/test_regression.py

from http import HTTPStatus
import pytest

@pytest.mark.regression
def test_users_endpoint_handles_gracefully(client):
    """
    Test that route handles errors gracefully.
    Even if external API fails, route should not crash.
    """
    # Act
    response = client.get('/users')

    # Should return proper JSON response even when GitHub API fails
    assert response.status_code in [HTTPStatus.OK, HTTPStatus.INTERNAL_SERVER_ERROR]
    
    # Response should always be valid JSON
    import json
    try:
        data = json.loads(response.data)
        assert isinstance(data, (list, dict))
    except json.JSONDecodeError:
        assert False, "Response is not valid JSON"
```

### Regression Testing (Line-by-Line Explanation)

``` python
@pytest.mark.regression
```

- Custom marker for regression tests
- Can be run with: `pytest -m regression`

------------------------------------------------------------------------

``` python
assert response.status_code in [HTTPStatus.OK, HTTPStatus.INTERNAL_SERVER_ERROR]
```

- Ensures the endpoint always returns proper responses
- Even when external API fails, returns structured JSON
- Prevents crashes and unhandled exceptions

**Run regression tests only**: `pytest -m regression`

## Coverage

```bash
pytest --cov=app --cov-report=term-missing
```

This:

- Tracks executed lines
- Shows missing lines in coverage report
- Integrated into pytest.ini for automatic coverage

**Current coverage report from the project**:
```
==================== tests coverage ====================
Name     Stmts   Miss  Cover   Missing
--------------------------------------
app.py      12      2    83%   10, 15
--------------------------------------
TOTAL       12      2    83%
```

**Coverage metrics explained**:
- **Name**: File name being measured
- **Stmts**: Total statements (lines of code) in the file
- **Miss**: Number of statements not executed by tests
- **Cover**: Percentage of code covered by tests
- **Missing**: Specific line numbers not covered

⚠️ **Important**: High coverage does not mean high-quality tests. Focus on testing behavior, not just lines.

------------------------------------------------------------------------

## Running Different Test Types

```bash
# Run only unit tests
pytest -m unit

# Run only integration tests
pytest -m integration

# Run regression tests
pytest -m regression

# Skip slow tests
pytest -m "not slow"

# Run performance benchmarks
pytest --benchmark-only

# Run with coverage (configured in pytest.ini)
pytest

# Run with verbose output
pytest -v

# Run specific test file
pytest tests/test_users_unit.py
```

## Best Practices Summary

1. **Test Organization**
   - Separate production code from tests
   - Use descriptive test names
   - Group related tests in files

2. **AAA Pattern**
   - Arrange: Prepare test data and mocks
   - Act: Execute the function under test
   - Assert: Verify the outcome

3. **Mocking Strategy**
   - Mock external dependencies in unit tests
   - Use real calls in integration tests
   - Mock only what you need (specific methods)

4. **Fixtures Usage**
   - Reuse common test setup
   - Keep fixtures focused and simple
   - Use `yield` for cleanup operations

5. **Test Categories**
   - Unit: Fast, isolated, business logic
   - Integration: Real external calls
   - Functional: Full user scenarios
   - Performance: Benchmark critical paths
   - Regression: Prevent bug recurrence

## Advanced Tips

### Custom Markers
```python
# pytest.ini
markers =
    unit: marks tests as unit tests
    integration: marks tests as integration tests
    regression: marks tests as regression tests
    slow: marks tests as slow tests
    network: marks tests requiring internet
```

### Test Configuration
```python
# conftest.py
@pytest.fixture(scope="session")
def api_client():
    """Client shared across all tests"""
    return SomeApiClient()

@pytest.fixture(autouse=True)
def setup_test_environment():
    """Auto-used fixture for all tests"""
    # Setup code here
    yield
    # Cleanup code here
```

## Conclusion

You now understand:

- **Test Organization**: Proper structure and separation
- **Unit Tests**: Isolated testing with mocks
- **Integration Tests**: Real API calls
- **Functional Tests**: End-to-end scenarios
- **Performance Tests**: Benchmarking with pytest-benchmark
- **Regression Tests**: Preventing bug recurrence
- **Mocking Strategy**: When and how to mock
- **Fixtures**: Reusable test components
- **Markers**: Categorizing and filtering tests
- **Parametrize**: Reducing test duplication
- **AAA Pattern**: Clear test structure
- **Coverage**: Measuring test completeness

This is **professional-level test architecture** that will scale with your project.

Full project on **[Github](https://github.com/alisonamerico/github_api_tests)**
