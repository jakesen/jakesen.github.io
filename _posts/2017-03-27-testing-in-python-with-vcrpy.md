---
layout: post
title:  "Testing in Python with VCR.py"
date: 2017-03-27 08:05:00 +0600
categories: python, testing
---
[VCR.py][vcrpy] is a python version of the [Ruby VCR library][vcr]. When testing code that makes HTTP requests, VCR.py can make your tests faster, more reliable and more secure, and in general just make your life a lot easier.

VCR works by recording HTTP requests and responses generated while running tests, and serializing them into a cassette file. Future HTTP requests are intercepted and the recorded reponse is returned withouth ever connecting to the original HTTP host. This means that once the cassette is recorded, the tests can still be executed even if there is no internet connection, or the host is unavailable for some reason.

Another useful feature of VCR is that it can obscure sensitive API tokens and passwords in the cassette, and a real token or password is no longer needed once the cassette has been recorded. This means that developers can run the tests without even having access to a valid token or password!

[vcrpy]:https://github.com/kevin1024/vcrpy
[vcr]: https://github.com/vcr/vcr

<!--description-->

I found a great excuse to try out VCR.py while working on [pyhatchbuck][pyhatchbuck], a wrapper library for the [Hatchbuck][hatchbuck] API. By using VCR.py, the tests for pyhatchbuck will not require permanent and continuous access to the test API account, nor will they depend on a particular state of data being represented by the API.

[pyhatchbuck]:https://github.com/jakesen/pyhatchbuck
[hatchbuck]:http://www.hatchbuck.com


### Demonstration

To demonstrate how one might use VCR.py in their python tests, I'll create a little project for accessing the [GitHub API][github-api] with an OAuth token. To keep things simple, we'll just have a class to store the token along with a method to return a list of all the repositories for a GitHub user. You can view all the code in this example in the [vcrpy-demo respository][vcrpy-demo-repo].

First a little setup:

```sh
➜ mkdir vcr-test
➜ cd vcr-test
➜ virtualenv env
➜ source env/bin/activate
(env)➜ pip install nose
(env)➜ pip install requests
```

Now, a python module with some code we want to test: `gitrepos.py`

```py
import requests
import json

class GitRepos(object):
    def __init__(self, oauth_token):
        self.oauth_token = oauth_token

    def list_for_user(self, username):
        request_url = "https://api.github.com/users/%s/repos" % username
        request_url += "?access_token="+self.oauth_token
        response = requests.get(request_url)
        if response.status_code == 200:
            return [repo.get('name') for repo in json.loads(response.content)]
        else:
            raise Exception(response.content)
```

And finally, a test for our tiny GitHub lib: `test.py`

```py
import os
import unittest
from gitrepos import GitRepos

class GitReposTest(unittest.TestCase):
    def setUp(self):
        self.test_api_token = os.environ.get("GITHUB_API_TOKEN", "ABC123")

    def test_list_for_user(self):
        repos = GitRepos(self.test_api_token)
        self.assertTrue('nwalsdev' in repos.list_for_user('jakesen'))
```

As you can see, the test attempts to retrieve the API token from an environment variable. If that's not available it just uses a dummy token. Of course, if the dummy token is used to make a request to GitHub it will fail and our code will raise an exception. Let's try it just to see what happens.

```sh
(env)➜ nosetests
E
======================================================================
ERROR: test_list_for_user (test.GitReposTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/jacobsenecal/Documents/Jakesen/vcr-tests/test.py", line 11, in test_list_for_user
    self.assertTrue('nwalsdev' in repos.list_for_user('jakesen'))
  File "/Users/jacobsenecal/Documents/Jakesen/vcr-tests/gitrepos.py", line 15, in list_for_user
    raise Exception(response.content)
Exception: {"message":"Bad credentials","documentation_url":"https://developer.github.com/v3"}
-------------------- >> begin captured logging << --------------------
requests.packages.urllib3.connectionpool: DEBUG: Starting new HTTPS connection (1): api.github.com
requests.packages.urllib3.connectionpool: DEBUG: https://api.github.com:443 "GET /users/jakesen/repos?access_token=ABC123 HTTP/1.1" 401 83
--------------------- >> end captured logging << ---------------------

----------------------------------------------------------------------
Ran 1 test in 0.353s

FAILED (errors=1)
```

But if I set the environment variable and then run the test we see that everything passes.

```sh
(env)➜  export GITHUB_API_TOKEN=<secret stuff>
(env)➜  nosetests
.
----------------------------------------------------------------------
Ran 1 test in 0.464s

OK
```

Now we have a test that depends on a successful HTTP request with a valid API token. That means if the API token changes or is not available in a future test environment this test will fail. It will also fail if for some reason GitHub's API server is unavailable or the test is run without an internet connection (maybe while working on an airplane or in a mountain cabin). We've added all these dependencies to our test even though we just want to check the function of our code: that it structures the request properly and handles the response appropriately. Out intent here is not to test GitHub's API or a token's validity!

## VCR.py to the Rescue!

Fortunately, it is easy to remove these dependencies and ship our code with reliable, portable test coverage.

First we need to install vcrpy:

```sh
(env)➜  pip install vcrpy
```

Then we add the `use_cassette` decorator to the test we want to record a cassette for:

```py
import os
import unittest
import vcr
from gitrepos import GitRepos

class GitReposTest(unittest.TestCase):
    def setUp(self):
        self.test_api_token = os.environ.get("GITHUB_API_TOKEN", "ABC123")

    @vcr.use_cassette(
        'fixtures/cassettes/test_list_for_user.yml',
        filter_query_parameters=['access_token']
    )
    def test_list_for_user(self):
        repos = GitRepos(self.test_api_token)
        self.assertTrue('nwalsdev' in repos.list_for_user('jakesen'))
```

Notice how we are designating `access_token` as a query parameter that will be filtered. The filtering will ensure that the secret API token does not get recorded in the cassette. That way we can safely commit the cassette with the rest of our code and ship it out, even to a public repository, without fear of our credentials falling into the wrong hands.

Now when we run the nosetests again we'll see that a cassette file has been created at `fixtures/cassettes/test_list_for_user.yml`:

```yml
interactions:
- request:
    body: null
    headers:
      Accept: ['*/*']
      Accept-Encoding: ['gzip, deflate']
      Connection: [keep-alive]
      User-Agent: [python-requests/2.13.0]
    method: GET
    uri: https://api.github.com/users/jakesen/repos
  response:
    body:
      string: !!binary |
        H4sIAAAAAAAAA+2dbW/ruJWA/4rgTy02ji05thMBi+5tp0BbTItpb4oFdrEIZJuJdSNLriQnN2Pc
        /95DShZfRMoyeTLI7NWXmcQhHx69UBafe0j+73EUb0bh4jYIltNgcTVKox0ZhaNos4vTuCjzqCSj
        ===============================================
        LOTS-MORE-BINARY-LINES-REMOVED-HERE-FOR-BREVITY
        ===============================================
        woCtxqVO2VyMpTI6gcmKTJf3kG43A4HZd8UuY9R9deY5QE+5eQ7TS3UaIbL4pENmWCFULk1nfk5K
        stsnMPoeZzCB6SUmr8WE67sJUOoV5CG77M5KmWr2WvyO0vP+79/aTViVm/oAAA==
    headers:
      access-control-allow-origin: ['*']
      access-control-expose-headers: ['ETag, Link, X-GitHub-OTP, X-RateLimit-Limit,
          X-RateLimit-Remaining, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes,
          X-Poll-Interval']
      cache-control: ['private, max-age=60, s-maxage=60']
      content-encoding: [gzip]
      content-security-policy: [default-src 'none']
      content-type: [application/json; charset=utf-8]
      date: ['Sat, 25 Mar 2017 13:28:33 GMT']
      etag: [W/"9490d33002f09fb3cc09ccf9f8c76c7f"]
      server: [GitHub.com]
      status: [200 OK]
      strict-transport-security: [max-age=31536000; includeSubdomains; preload]
      vary: ['Accept, Authorization, Cookie, X-GitHub-OTP', Accept-Encoding]
      x-accepted-oauth-scopes: ['']
      x-content-type-options: [nosniff]
      x-frame-options: [deny]
      x-github-media-type: [github.v3; format=json]
      x-github-request-id: ['94E7:3545:28E5E1A:33454DF:58D67081']
      x-oauth-scopes: [repo]
      x-ratelimit-limit: ['5000']
      x-ratelimit-remaining: ['4995']
      x-ratelimit-reset: ['1490449232']
      x-served-by: [a6882e5cd2513376cb9481dbcd83f3a2]
      x-xss-protection: [1; mode=block]
    status: {code: 200, message: OK}
version: 1
```

Another benefit of VCR.py is that it gives us an easy way to inspect the request and response details, all we have to do is look in the cassette!

Now we can get rid of the environment variable and see what happens when we run the test again using just the invalid dummy token:

```sh
(env)➜  unset GITHUB_API_TOKEN
(env)➜  nosetests
.
----------------------------------------------------------------------
Ran 1 test in 0.214s

OK
```

As you can see, the test passed with flying colors, even though it was not using a valid API key. The results would have been exactly the same if the test had been run with no internet connection. This is just a simple example but it demonstrates the power and usefullness of VCR.py.

[github-api]:https://developer.github.com/v3/
[vcrpy-demo-repo]:https://github.com/jakesen/vcrpy-demo

## Conclusion

After using VCR.py to develop and release software that integrates with API services, I can hardly imagine doing this kind of work in the future without using VCR.py or some similar tool. It provides speed and freedom for the test-driven development process, and security and reliability for sharing code and continuous integration. In short it's an invaluable tool that every developer should know how to leverage for their projects! 

