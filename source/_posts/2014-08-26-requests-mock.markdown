---
layout: post
title: requests-mock
date: 2014-08-26 12:26:23 +1000
comments: true
categories: openstack python requests-mock
---

Having just release v0.5 of [requests-mock](https://pypi.python.org/pypi/requests-mock) and having it used by both keystoneclient and novaclient with others in the works I thought I'd finally do a post explaining what it is and how to use it.

Motivation
----------

I was the person who brought [HTTPretty](http://falcao.it/HTTPretty/) into the OpenStack requirements.

The initial reason for this was that keystoneclient was transitioning from the [httplib](https://docs.python.org/2/library/httplib.html) library to [requests](http://docs.python-requests.org/en/latest/) and I needed to prove that there was no changes to the HTTP requests during the transition.
HTTPretty is a way to mock HTTP responses at the socket level, so it is not dependant on the HTTP library you use and for this it was fairly successful.

As part of that transition I converted all the unit tests so that they actually traversed through to the requesting layer and found a number of edge case bugs because the responses were being mocked out above this point.
I have therefore advocated that the clients convert to mocking at the request layer rather than stubbing out returned values.
I'm pretty sure that this doesn't adhere strictly to the unit testing philosophy of testing small isolated changes, but our client libraries aren't that deep and I'd honestly prefer to just test the whole way through and find those edge cases.

Having done this has made it remarkably easier to transition to using sessions in the clients as well, because we are testing the whole path down to making HTTP requests for all the resource tests so again have assurances that the HTTP requests being sent are equivalent.

At the same time we've had a number of problems with HTTPretty:

 - It was the lingering last requirement for getting Python 3 support. Thanks to Cyril Roelandt for finally getting that fixed.
 - For various reasons it is difficult for the distributions to package.
 - It has a bad habit of doing backwards incompatible, or simply broken releases. The current requirements string is: `httpretty>=0.8.0,!=0.8.1,!=0.8.2,!=0.8.3`
 - Because it acts at the socket layer it doesn't always play nicely with other things using the socket. For example it has to be disabled for live memcache tests.
 - It pins its requirements on pypi.

Now I feel like I'm just ranting.
There are additional oddities I found in trying to fix these upstream but this is not about bashing HTTPretty.

requests-mock
-------------

requests-mock follows the same concepts allowing users to stub out responses to HTTP requests, however it specifically targets the requests library rather than stubbing the socket.
All the OpenStack clients have been converted to requests at this point, and for the general audience if you are writing HTTP code in Python you should be using requests.

Note: a lot of what is used in these examples is only available since the 0.5 release.
The current OpenStack requirements still have 0.4 so you'll need to wait for some of the new syntax.

The intention of requests-mock is to work in as similar way to requests itself as possible.
Hence all the variable names and conventions should be as close to a `requests.Response` as possible.
For example:

{% codeblock lang:python %}

>>> import requests
>>> import requests_mock
>>> url = 'http://www.google.com'
>>> with requests_mock.mock() as m:
...     m.get(url, text='Not really google', status_code=218)
...     r = requests.get(url)
...
>>> r.text
u'Not really google'
>>> r.status_code
218

{% endcodeblock %}

So `text` in the mock equates to `text` in the response and similarly for `status_code`.
Some more advanced usage of the requests library:

{% codeblock lang:python %}

>>> with requests_mock.mock() as m:
...     m.get(url, json={'hello': 'world'}, headers={'test': 'header'})
...     r = requests.get(url)
...
>>> r.text
u'{"hello": "world"}'
>>> r.json()
{u'hello': u'world'}
>>> r.status_code
200
>>> r.headers
{'test': 'header'}
>>> r.headers['test']
'header'

{% endcodeblock %}

You can also use callbacks to create responses dynamically:

{% codeblock lang:python %}

>>> def _request_callback(request, context):
...     context.status_code = 201
...     context.headers['test'] = 'header'
...     return {'request': request.body}
...
>>> with requests_mock.mock() as m:
...     m.post(url, json=_request_callback)
...     r = requests.post(url, data='data')
...
>>> r.status_code
201
>>> r.headers
{'test': 'header'}
>>> r.json()
{u'request': u'data'}

{% endcodeblock %}

Note that because the callback was passed as the `json` parameter the return type is expected to be the same as if you had passed it as a predefined `json=blob` value.
If you wanted to return `text` the callback would be on the `text` parameter.

Cool tricks
-----------

So rather than give a lot of examples i'll just highlight some of the interesting things you can do with the library and how to do it.

 - Queue mutliple responses for a url, each element of the list is interpreted as if they were `**kwargs` for a response.
   In this case every request other than the first will get a 401 error:

{% codeblock lang:python %}
m.get(url, [{'json': _request_callback},
            {'text': 'Not Allowed', 'status_code': 401}])
{% endcodeblock %}

 - See the history of requests:

{% codeblock lang:python %}
m.request_history  # all requests
m.last_request  # the last request
m.call_count  # number of requests
m.called  # boolean, True if called
{% endcodeblock %}

 - match on the only the URL path:

{% codeblock lang:python %}
m.get('/path/only')
{% endcodeblock %}

 - match on any method:

{% codeblock lang:python %}
m.request(requests_mock.ANY, url)
{% endcodeblock %}

 - or match on any URL:

{% codeblock lang:python %}
m.get(requests_mock.ANY)
{% endcodeblock %}

 - match on headers that are part of the request (useful for distinguishing between multiple requests to the same URL):

{% codeblock lang:python %}
m.get(url, request_headers={'X-Auth-Token': 'XXXXX'})
{% endcodeblock %}

 - be used as a function decorator

{% codeblock lang:python %}
@requests_mock.mock()
def test_a_thing(m):
   m.get(requests_mock.ANY, text='resp')
   ...
{% endcodeblock %}

Try it!
-------

There is a lot more it can do and if you want to know more you can check out:

 - [Read the Docs](http://requests-mock.readthedocs.org/)
 - [PyPi](https://pypi.python.org/pypi/requests-mock)
 - [git repository](https://git.openstack.org/cgit/stackforge/requests-mock)

As a final selling point because it was built particularly around OpenStack needs it is:

 - Easily integrated with the [fixtures](https://pypi.python.org/pypi/fixtures) library.
 - Hosted on stackforge and reviewed via [Gerrit](https://review.openstack.org/#/q/project:stackforge/requests-mock+is:open,n,z).
 - Continuously tested against at least keystoneclient and novaclient to prevent backwards incompatible changes.
 - Accepted as part of OpenStack requirements.

Patches and [bug reports](https://bugs.launchpad.net/requests-mock) are welcome.
