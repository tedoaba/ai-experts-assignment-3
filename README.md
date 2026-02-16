# AI Experts Assignment Solution

This repository contains the solution for the AI Experts Python assignment. It demonstrates a reliable setup using Docker, pinned dependencies, and a fix for a subtle logic bug in the HTTP client.

## Solution Overview

### 1. Robust Containerization (Dockerfile)

To ensure the test suite runs in a consistent, non-interactive CI-style environment, I implemented a `Dockerfile`. usage of a slim Python image minimizes the footprint while ensuring all necessary tools are available.

- **Consistency**: It installs the dependencies defined in `requirements.txt` in a clean environment.
- **Automation**: The container is configured to automatically execute the full test suite using `pytest -v` upon startup, providing immediate feedback on code health.

### 2. Reproducible Environment (requirements.txt)

To prevent "it works on my machine" issues, I created a `requirements.txt` file that strictly pins all necessary libraries—`requests`, `python-dateutil`, and `pytest`—to specific versions. This guarantees that the development and testing environments satisfy the exact same dependency constraints.

### 3. Verification & Testing Steps

I have updated the documentation to include clear steps for verification.

**Running Tests Locally:**
To verify the fix on your machine:

1.  Install the pinned dependencies:
    ```bash
    pip install -r requirements.txt
    ```
2.  Execute the test suite:
    ```bash
    python -m pytest -v
    ```

**Running Tests via Docker:**
To verify the fix in an isolated environment:

1.  Build the Docker image:
    ```bash
    docker build -t app-tests .
    ```
2.  Run the validation container:
    ```bash
    docker run --rm app-tests
    ```

### 4. The Bug Fix: Token Refresh Logic

**The Problem:**
I identified a critical bug in the `http_client.py` module. The `Client` class failed to refresh the OAuth2 token when the token was provided as a raw dictionary (e.g., `{'access_token': '...', 'expires_at': 0}`) instead of a structured `OAuth2Token` object. The original conditional logic simply skipped the refresh step for dictionaries because they didn't match the expected class type, leading to the use of invalid or stale tokens.

**The Solution:**
I corrected the conditional check in the `request` method to be more robust.

- **Original Logic**: Only refreshed if the token was `None` or an _expired_ `OAuth2Token` object.
- **New Logic**: Now forces a refresh if the token is _anything other than_ a valid `OAuth2Token` object (handling `None`, `dict`, etc.) OR if the valid object is expired.

This minimal change ensures that the client automatically recovers from invalid states by fetching a fresh token, satisfying the failing test case (`test_api_request_refreshes_when_token_is_dict`).

### 5. Design Considerations & Limitations

**Root Cause:**
The bug stemmed from an overly specific type check (`isinstance`) that didn't account for other "truthy" types like dictionaries. Python's dynamic nature requires careful handling of such types when defining control flow.

**Remaining Edge Case:**
While the fix addresses the immediate crash/failure, the system does not yet handle race conditions. If a valid token expires strictly _after_ the check but _before_ the request reaches the server, the API call will still fail with a 401 error. A production-grade solution would likely implement a retry mechanism that catches 401 responses and attempts one token refresh before failing.
