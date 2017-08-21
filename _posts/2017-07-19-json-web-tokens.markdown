---
layout: post
title:  "Implementing JWT in Django"
date:   2017-07-19 07:14:00
author: Ryan Castner
categories: Django
tags: jwt json web token django
cover:  "/assets/crypto.jpeg"
---

# In the beginning there were Sessions

In a typical HTTP Request/Response cycle with Django, the [sessions](https://docs.djangoproject.com/en/1.11/topics/http/sessions/) framework is used to authenticate users. How it works in the normal case is that each visitor to the site is assigned a session id which is saved in the database. A cookie is then generated which contains the unique session id and is returned to the client to be attached to subsequent requests. Django's Session Middlware will intercept the cookie that is passed with future requests and get the corresponding session from the database. In this way, no information from the session is **ever** passed to the client. The session can hold any arbitrary information like whether the user is authenticated or anonymous, their user id in the database, or anything else a developer might want to store about visitors.

The major concept with sessions is that it is a **stateful** form of authentication. Stateful in this context means that the server holds the state of each user at all times. If there was ever a security concern about a user the server can arbitrarily destroy their session or limit a session and its access for a period of time. The downside that comes with this stateful authentication is that it does not scale well. For each user and each request that the user makes the server needs to go through the entire cycle of looking up getting the session id from the cookie, looking up that session id in the database, and using that information to return the appropriate response. In the beginning when each HTTP GET would return an entire page on a website, the scalability wasn't so bad, but in this new age of frontend clients making dozens of asynchronous requests to generate the content for a single page and continuing to make requests while on the same page, the number of requests issued on average by each user has risen dramatically. Indeed, a server might be looking up the same session multiple times for requests happening within milliseconds of each other. While there are solutions to this problem, like setting up a caching server (e.g. Redis), another solution arose that moved away from the concept of stateful sessions entirely, Tokens.

# Token the world by storm

Tokens are a form of **stateless** authentication. A token contains all the necessary information to authenticate a user and any other data that a developer decides to include with it. The idea being that the token is cryptographically encrypted so it doesn't matter if we give this information back to the client. This is a major difference from sessions which avoid the issue entirely by never letting clients have access to the data, encrypted or not. Tokens have solved some of the issues sessions had when working with frontend clients. The token can be given to multiple sources and shared, so a client can communicate from the app on their phone, from the website, or through their own api interface with the token. Where the token comes from doesn't matter, and all the server does is decrypt that token to know who the user is.

This all seems great so far, so what are the downsides? Well, for one, an attacker who gets hold of a token, given enough time could decrypt that token. As a result the developer needs to be careful to never pass sensitive information like billing or personal information in a token. Furthermore, the server has no control over the token and who it is given to. A user could potentially compromise themselves if their token was given to someone else (man in the middle attack, malware, phishing, etc.). The server then cannot invalidate that token like it could with a session that it could just delete. What happens then is that the server needs to keep a list of blacklisted tokens (a much smaller list than the whitelisted tokens, hopefully) and check incoming tokens against the blacklist before proceeding. This defeats some of the scalability benefit of using tokens in the first place. Tokens can also become stale, much like a cache, a token contains a snapshot of encoded information that could change and there is no way to update it because it is not being managed by the server. Imagine a token has 'admin' role access encoded into it and that user's access is revoked, unless you blacklist that token it will still be considered to have 'admin' access until the token expires.

With this in mind, if you are still itching to use JWT in your Django app, continue to read onto implementation. If you are having doubts, check out this great in-depth analysis of the [Security Implications of using JWT](http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/).

## Implementing JWT Auth

The JWT specification tells us how to encode data into a token in a cryptographically secure manner and how to decrypt and verify that token. We luckily don't need to worry about the implementation details as there are already existing libraries that take care of this for us. We will be using the defacto standard [rest framework](https://github.com/encode/django-rest-framework) for our Django REST API and [djangorestframework-jwt](https://github.com/GetBlimp/django-rest-framework-jwt), the suggested third party extension to help manage jwt.

Add `rest_framework` to your `INSTALLED_APPS`:

```
# settings.py

INSTALLED_APPS = [
    # ...,
    'rest_framework',
    # ...,
]
```

Include `djangorestframework-jwt`'s `JSONWebTokenAuthentication` to rest framework's `DEFAULT_AUTHENTICATION_CLASSES`.

```
# settings.py

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
    ),
}
```

In your top level `urls.py` add the following url pattern:

```
# urls.py

from rest_framework_jwt.views import obtain_jwt_token

urlpatterns = [
    '',
    # ...
    url(r'^jwt-auth/', obtain_jwt_token),
]
```

Feel free to rename the pattern to whatever fits your apps needs and makes sense to you. Now we can pass user auth data to this endpoint and have a token returned.

```
$ curl -X POST -H "Content-Type: application/json" -d
  '{"username": "admin","password":"someSecret"}'
  http://localhost:8000/jwt-auth/
```

The token that is returned will be included as a header in subsequent requests in the form `Authorization: JWT <token>`.

Now lets create a view to register a user. This view will, after creating a new user, create a JWT token for that user and return the token.

```
# api.py

from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework_jwt.settings import api_settings

from .serializers import UserSerializer

jwt_payload_handler = api_settings.JWT_PAYLOAD_HANDLER
jwt_encode_handler = api_settings.JWT_ENCODE_HANLDER

@api_view(['POST'])
def jwt_register_user(request):
    serializer = UserSerializer(data=request.data)
    if serializer.is_valid():
        user = serializer.save()
        payload = jwt_payload_handler(user)
        token = jwt_encode_handler(payload)
        return Response({'token': token}, status=status.HTTP_201_CREATE)
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

Let's break down what is happening. `jwt_payload_handler` is a function that takes a user object and generates a jwt payload from it, `jwt_encode_handler` will encode that payload into a json web token. We are taking these functions from `djangorestframework-jwt`'s built-in implementation so we don't have to worry about implementing token creation ourselves. Now we have it so when a `POST` request comes in, we try to serialize a `User` instance and generate a new token that we return. This step saves us from having the API call the `jwt-auth` endpoint after this endpoint returns.

Add our `jwt_register` view to our `urls.py`:

```
# urls.py

from .api import jwt_register_user

urlpatterns = [
    # ...
    url(r'^jwt-auth/', obtain_jwt_token),
    url(r'^register/', jwt_register_user),
]
```

Now we are all set. JWT auth for our backend has been implemented.

## A sample JavaScript API implementation

Below I will describe a simple api implementation to handle authentication. I like to define an `api.js` file that holds all objects that mirror the app's django models and call methods that map to the backend api. We also need to include the `csrf_token` in our requests and as such need to grab this cookie. Django provides an [implementation for us](https://docs.djangoproject.com/en/1.11/ref/csrf/#ajax) but it uses jQuery. Below is a slightly modified version that removes the jQuery dependency.

```
// _getCookie.js

export const getCookie = function getCookie(name) {
  var cookieValue = null;
  if (document.cookie && document.cookie !== '') {
    var cookies = document.cookie.split(';');
    for (var i = 0; i < cookies.length; i++) {
      // trim whitespace around cookie
      var cookie = cookies[i].replace(/(^\s+|\s+$)/g,'');
      // Does this cookie string begin with the name we want?
      if (cookie.substring(0, name.length + 1) === (name + '=')) {
        cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
        break;
      }
    }
  }
  return cookieValue;
};
```

Now onto some basic plumbing for our `api.js` file, requests handling.

```
import { getCookie } from './_getCookie';

// set our API_ROOT
const API_ROOT = window.location.origin + '/api';

// implement a standard function to check for a good or bad response
// throwing an err if bad
const checkStatus = res => {
  if (res.status >= 200 && res.status < 300) {
    return res;
  }
  let error = new Error(`HTTP Error ${res.statusText}`);
  error.status = res.status;
  error.res = res;
  throw error;
}

// make a more readable shorthand
const parseJSON = res => res.json();

// define a null jwt by default
let jwt = null;

// function to set our token to local storage
export const setToken = _token => {
  jwt = _token;
  window.localStorage.setItem('myapps_authtoken', jwt);
}

// function to reduce duplicated code of making fetch requests
const api = (method, url, body) => {
  // default headers needed, pass http method, credentials, and header accept type
  const options = {
    method: method,
    credentials: 'same-origin',
    headers: {
      Accept: 'application/json',
    },
  };

  // if our request has a body, we need to JSON.stringify it
  // add the Content-Type header and add the X-CSRFToken header
  if (body) {
    options.body = JSON.stringify(body);
    options.headers['Content-Type'] = 'application/json';
    options.headers['X-CSRFToken'] = getCookie('csrftoken');
  }

  // if the jwt is available, include it in the request header
  // recall the header needed was Authorization: JWT <token>
  if (jwt) {
    options.headers['Authorization'] = `JWT ${jwt}`;
  }

  // handle special 204 case where no content is returned
  return fetch(url, options)
          .then(checkStatus)
          .then(res => res.status !== 204 ? parseJSON(res) : res);
};

// implement a basic wrapper around our api to make different HTTP requests
const requests = {
  get: url => api('GET', `${API_ROOT}${url}`),
  patch: (url, body) => api('PATCH', `${API_ROOT}${url}`, body),
  put: (url, body) => api('PUT', `${API_ROOT}${url}`, body),
  post: (url, body) => api('POST', `${API_ROOT}${url}`, body),
  delete: (url, body) => api('DELETE', `${API_ROOT}${url}`, body),
};
```

I like to do all this plumbing to keep my ORM-like objects that communciate with the API as simple as possible and really cut down on the duplicated code. Once you try out the abstraction I think you will like it. Lets implement our `User` object in `api.js`.

```
// api.js

// prior code from above omitted

export const User = {
  login: (username, password) =>
    requests.post('/jwt-auth', { username, password })
      .then(res => setToken(res.token)),
  logout: () =>
    setToken('jwt', null),
  register: (username, email, password) =>
    requests.post('/register', { username, email, password })
      .then(res => setToken(res.token)),
  token: () =>
    !!jwt,
};
```

We are using a lot of features of ES6/7 here, like the shorthand of `{ username }` that expands to `{username: username}`, and the es6 arrow syntax for function declarations. We also use a cool trick in javascript to check if the `jwt` is set: `!!jwt`. A single `!` negates, and `!!` provides a double negation. Thus, `!!true === true`, `!!false === false`, and `!!null === false`. If we simply returned `jwt` we could get `true`, `false`, or `null`. The `!!` trick moves a `null` or `undefined` to `false`. Basically, it is just more terse than `jwt !== null && jwt !== undefined && jwt`.

Note that we persist the token by setting it into the browser's `window.localStorage` and that logging out is simply removing the token from `window.localStorage` and setting `jwt = null`. This is because the server has no concept of state and whether the user is logged in or not, it is all managed by the client once it receives the token. The only way the server can manage it is, as previously mentioned, creating a blacklist to check tokens against.

A sample use of this api is now pretty straightforward

```
import { User } from 'api';

User.login('admin', 'someSecret').then(console.log('auth successful'));
User.register('admin2', 'admin2@example.com', 'someSecret2');
User.logout();

if (User.token()) {
  console.log('User has a token so we can proceed to make an authenticated request');
} else {
  console.log('User is not authenticated, redirect to login form perhaps');
}
```

Each request will use the correct verb, encode data passed as a json string, include the correct headers and csrf token as required, and include the jwt token for authorization. The result throws an error if outside the range of acceptable requests, otherwise it parses the json result and passes it back to be used.

Congratulations, you now have a functioning JWT implementation. Some things you may want to consider moving forward:

- Implementing a token blacklist
- Security settings for [djangorestframework-jwt](https://getblimp.github.io/django-rest-framework-jwt/#additional-settings) like expiry time
- [Security implications of using JWT](https://audiolion.github.io/2017/07/18/security-implications-jwt.html)
