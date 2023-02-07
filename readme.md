<!-- !/usr/bin/env markdown
-*- coding: utf-8 -*-
region header
Copyright Torben Sickert (info["~at~"]torben.website) 16.12.2012

License
-------

This library written by Torben Sickert stand under a creative commons naming
3.0 unported license. See https://creativecommons.org/licenses/by/3.0/deed.de
endregion -->

Project status
--------------

[![npm](https://img.shields.io/npm/v/reverse-proxy-middleware?color=%23d55e5d&label=npm%20package%20version&logoColor=%23d55e5d)](https://www.npmjs.com/package/reverse-proxy-middleware)
[![npm downloads](https://img.shields.io/npm/dy/reverse-proxy-middleware.svg)](https://www.npmjs.com/package/reverse-proxy-middleware)

[![<LABEL>](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/build.yaml/badge.svg)](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/build.yaml)
[![<LABEL>](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/test.yaml/badge.svg)](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/test.yaml)
[![<LABEL>](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/test-coverage-report.yaml/badge.svg)](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/test-coverage-report.yaml)
[![<LABEL>](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/check-types.yaml/badge.svg)](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/check-types.yaml)
[![<LABEL>](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/lint.yaml/badge.svg)](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/lint.yaml)
[![<LABEL>](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/build-image-periodically-2-branches.yaml/badge.svg)](https://github.com/thaibault/reverse-proxy-middleware/actions/workflows/build-image-periodically-2-branches.yaml)

[![code coverage](https://coveralls.io/repos/github/thaibault/reverse-proxy-middleware/badge.svg)](https://coveralls.io/github/thaibault/reverse-proxy-middleware)

[![documentation website](https://img.shields.io/website-up-down-green-red/http/torben.website/reverse-proxy-middleware.svg?label=documentation-website)](http://torben.website/reverse-proxy-middleware)

Use case
--------

Easy and dynamically configurable reverse proxy. This tool can act as a
middleware to check incoming requests via external bot filtering services such
as ReCaptcha or friendlycaptcha.
Adding state via session token with external authentication apis is possible as
applying any external api given data to the incoming and conditionally
forwarded requests.

### Quick start

#### Simple forwarding

Simple reverse proxy request from `http://localhost:8080` to
`https://www.google.com` without modifying the entire request.

```
{
  "forwarders": {
    "google": {
      "host": "www.google.com"
    }
  }
}
```

Since the proxy starts at localhost on port 8080 as a default configuration you
can check the configuration via a simple curl command:

```curl --verbose http://localhost:8080```

Behind there are a some commonly use defaults configured under key
"configuration" in [package.json](package.json). Please have a look.

#### Distributing requests to different backends

Here is how to distribute incoming requests randomly between google and bing:

```
{
  "forwarders": {
    "bing": {
      "host": "www.bing.com",
      "useExpression": "Math.random() < 0.5"
    },

    "google": {
      "host": "www.google.com"
    }
  }
}
```

Forwarders are iterated in alphabetical order of their name and `useExpression`
are getting evaluated. Since "google" is always "used" it depends on the random
outcome of bing's expression if it is beeing used or not.

Since this proxy just streams the whole request through it could be used as
basic load balancer with this configuration.

### Rewriting headers for underlying backend

Headers can be replaced in both directions. Client-Request to forward or
retrieved responses given from configured backend:

```
{
  "forwarders": {
    "bing": {
      "host": "www.bing.com",

      "headerTransformations": {
        "send": {
          "source": "/X-Special-Client-Header-Name: (.+)/gi",
          "target": "'New-Header-Name: $1'"
        },

        "retrieve": {
          "source": "/set-cookie: (.+)/gi",
          "target": "'X-Prevented-Cookie: $1'"
        }
      }
    }
  }
}
```

Test via:

```
  curl \
    --header 'X-Special-Client-Header-Name: value' \
    --verbose \
    http://localhost:8080 \
      1>/dev/null
```

You should see a lot of cookie header ("set-cookie: ..." replaced by
"X-Prevented-Cookie: ...".
Note that muting the standart output ("1>/dev/null") enables you to focus on
retrieved headers printed via secondary error output.

### State-APIs

State-APIs enables you to trigger requests conditionally to third party
endpoints and use responses for further expressions.
Expression which transform subsequent api requests, decide which backend to use
or transform the final backend request.

Here is an example:

```
{
  "forwarders": {
    "bing": {
      "host": "www.bing.com",

      "stateAPIs": {
        "name": "sense",
        "url": "https://www.google.com/search?q=${data.question}",

        "data": {
          "question": "What is the meaning of life?"
        },

        "expressions": {
          "pre": "request.headers['ask-for-sense-of-live'] ? true : null",
          "post": "response.statusCode >= 200 && response.statusCode < 300 ? null : response.statusCode"
        }
      }
    }
  }
}
```

What is going on here?
We generally forward requests to "www.bing.com" but if there is a header called
"ask-for-sense-of-live" present in client request we will first ask google for
the meaning of life. If google answers with a "negative" response code not
between 200 and 300 we will just do the final forwarding to bing. If google
responses in a positive manner the resulting status code we will be used for
answering client and no forwarding to "www.bing.com" happens.

Here es a test curl command:

```
  curl \
    --header 'ask-for-sense-of-live: value' \
    --verbose \
    http://localhost:8080 \
      1>/dev/null
```

This should result in a simple response but:

```curl --verbose http://localhost:8080 1>/dev/null```

will finally request "www.bing.com".

Pre and post evaluations can have various results. The meanings of them are
described here:

#### Pre-Evaluation Results

| Result            | Meaning                                                                                                                                                                |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| break (string)    | Do not evaluate subsequent pre evaluations.                                                                                                                            |
| null or undefined | Just jump to the next evaluation to run.                                                                                                                               |
| true (boolean)    | Use this state api configuration. Run the configured request.                                                                                                          |
| false (boolean)   | Do not use this state api and to not run subsequent pre evaluations.                                                                                                   |
| code (number)     | Answer client request with provided http status code and do not run any subsequent pre-evaluations, state-api request or request forwarding to the underlying backend. |

#### Post-Evaluation Results

| Result            | Meaning                                                                                                                                                                 |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| break (string)    | Do not evaluate subsequent post evaluations.                                                                                                                            |
| null or undefined | Just jump to the next evaluation to run.                                                                                                                                |
| code (number)     | Answer client request with provided http status code and do not run any subsequent post-evaluations, state-api request or request forwarding to the underlying backend. |

### Smart configurations

Whenever you can configure list of items you can either use just one or a list
of them. Consider this configuration example:

```
{
  "forwarders": {
    "bing": {
      "host": "www.bing.com",

      "headerTransformations": [{...}, {...}, ...],

      "stateAPIs": {
        "expressions": {
          "pre": ["...", "...", ...],
          "post": ["...", "...", ...]
        }
      }
    }
  }
}
```

If only one item is needed please consider that:

```
{
  "forwarders": {
    "bing": {
      "host": "www.bing.com",

      "headerTransformations": [{...}],

      "stateAPIs": {
        "expressions": {
          "pre": ["..."],
          "post": ["..."]
        }
      }
    }
  }
}
```

is equivalent to:

```
{
  "forwarders": {
    "bing": {
      "host": "www.bing.com",

      "headerTransformations": {...},

      "stateAPIs": {
        "expressions": {
          "pre": "...",
          "post": "..."
        }
      }
    }
  }
}
```

#### Use environment variables

While some configuration values are interpret as expression to be evalued at
runtime e.g. to decide which endpoint to use:

```
{
  "forwarders": {
    "endpoint": {
      ...
      "useExpression": "..."
    }
  }
}
```

Every item can utilize expression to dynamically derive intial configurations:

```
{
  "port": {
    "__evaluate__": "environment.PORT ?? 8080"
  },
  ...
}
```

If an environment variabel "PORT" is set it will be used or "8080" as a
fallback.

#### Use Base Forwarder

Base forwarder are inherited by every specific forwarder. This configuration:

```
{
  "forwarders": {
    "base: {
      "headerTransformations": {
        "send": {
          "source": "/(GET|POST) \\/.* (.*)/",
          "target": "'$1 /sub/path/to/forward/to $2'"
        }
      }
    },

    "bing": {
      "host": "www.bing.com",
      "useExpression": "Math.random() < 0.5"
    },

    "google": {
      "host": "www.google.com"
    }
  }
}
```

is equivalent to:

```
{
  "forwarders": {
    "bing": {
      "host": "www.bing.com",

      "useExpression": "Math.random() < 0.5",

      "headerTransformations": {
        "send": {
          "source": "/(GET|POST) \\/.* (.*)/",
          "target": "'$1 /sub/path/to/forward/to $2'"
        }
      }
    },

    "google": {
      "host": "www.google.com",

      "headerTransformations": {
        "send": {
          "source": "/(GET|POST) \\/.* (.*)/",
          "target": "'$1 /sub/path/to/forward/to $2'"
        }
      }
    }
  }
}
```

#### Use base State-APIs

As we support generic base forwarder confgurations there are also base state
api configuraton sections. Consider the follwing configuration example:

```
{
  "forwarders": {
    "bing": {
      "host": "www.bing.com",

      "stateAPIs": [
        {
          "name": "base",
          "url": "https://www.google.com/search?q=${data.query}",

          "expressions": {
            "pre": "request.headers['ask-google'] ? true : null",
            "post": "response.statusCode >= 200 && response.statusCode < 300 ? null : response.statusCode"
          }
        }

        {
          "name": "sense",

          "data": {
            "query": "What is the meaning of life?"
          }
        },

        {
          "name": "nonesense",

          "data": {
            "query": "baby cats"
          }
        }
      ]
    }
  }
}
```

Note that url and expression are inherited to the state apis "sense" and
"nonsense".
The Data field can save various configuration items to be used in runtime
expressions.
Please also note that it is possible to access configuration, request or
response informations from other state apis in every runtime expression.

Every expression can access the following environment:

| Name      | Meaning                                                |
|-----------|--------------------------------------------------------|
| data      | Generic configuration items.                           |
| error     | Error object if some occured.                          |
| request   | Client request informations.                           |
| response  | Response informations (available in post evaluations). |
| stateAPIs | Access oher state api informations.                    |
| Tools     | [see](https://www.npmjs.com/package/clientnode)        |

<!-- region modline
vim: set tabstop=4 shiftwidth=4 expandtab:
vim: foldmethod=marker foldmarker=region,endregion:
endregion -->
