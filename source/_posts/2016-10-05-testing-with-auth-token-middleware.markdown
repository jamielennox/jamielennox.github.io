---
layout: post
title: "Testing with auth_token middleware"
date: 2016-10-05 16:28:21 +1000
comments: true
categories: openstack
---

For most OpenStack services the auth\_token middleware component is the only direct interaction the service will have with keystone.
It is the piece of the service that validates the token a user presents and relays the stored information.

When testing a service we want to ensure that the middleware is working and presenting the correct information but not actually talking to keystone.
To do this keystonemiddleware provides a fixture that will stub out the interaction with keystone with an existing token.

If all that makes sense to you then at this point I think you mostly just want a code example and you can research anything else you need.

```python
import json

from keystoneauth1 import fixture as ksa_fixture
from keystonemiddleware import auth_token
from keystonemiddleware import fixture as ksm_fixture
from oslo_context import context
import testtools
import webob.dec
import webtest


@webob.dec.wsgify
def app(request):
    """A really simple WSGI application that returns some context info."""

    # don't try to figure out what AuthToken sets, just use a context
    ctxt = context.RequestContext.from_environ(request.environ, overwrite=False)

    # return some information from context so we can verify the test
    resp_body = {'project_id': ctxt.tenant,
                 'user_id': ctxt.user,
                 'auth_token': ctxt.auth_token}

    return webob.Response(json.dumps(resp_body),
                          status_code=200,
                          content_type='application/json')


class Tests(testtools.TestCase):

    @staticmethod
    def load_app():
        # load your wsgi app here, wrapped in auth_token middleware
        return auth_token.AuthProtocol(app, {})

    @staticmethod
    def create_token():
        """Create a fake token that will be used in testing"""

        # Creates a project scoped V3 token, with 1 entry in the catalog
        token = ksa_fixture.V3Token()
        token.set_project_scope()

        s = token.add_service('identity')
        s.add_standard_endpoints(public='http://example.com/identity/public',
                                 admin='http://example.com/identity/admin',
                                 internal='http://example.com/identity/internal',
                                 region='RegionOne')

        return token

    def setUp(self):
        super(Tests, self).setUp()

        # create our app. webtest gives us a callable interface to it
        self.app = webtest.TestApp(self.load_app())

        # stub out auth_token middleware
        self.auth_token_fixture = self.useFixture(ksm_fixture.AuthTokenFixture())

        # create a token, mock it and save it and the ID for later use
        self.token = self.create_token()
        self.token_id = self.auth_token_fixture.add_token(self.token)

    def test_auth_token_params(self):
        # make a request with the stubbed out token_id and unpack the response
        body = self.app.get('/', headers={'X-Auth-Token': self.token_id}).json

        # ensure that the information in our stubbed token made it to the app
        self.assertEqual(self.token_id, body['auth_token'])
        self.assertEqual(self.token.project_id, body['project_id'])
        self.assertEqual(self.token.user_id, body['user_id'])
```

The important pieces are:

* [Webtest](http://docs.pylonsproject.org/projects/webtest/en/latest/) is a useful testing library emulating a wsgi layer so you can make requests.
* [ksa\_fixture.V3Token](http://docs.openstack.org/developer/keystoneauth/api/keystoneauth1.fixture.html#keystoneauth1.fixture.v3.Token) is a helper that builds the correct raw data layout for a V3 token.
* [AuthTokenFixture](http://docs.openstack.org/developer/keystonemiddleware/api/keystonemiddleware.html#keystonemiddleware.fixture.AuthTokenFixture) is doing the mocking and returning the token data.
* [RequestContext.from\_environ](http://docs.openstack.org/developer/oslo.context/api/context.html#oslo_context.context.RequestContext.from_environ) takes the information auth\_token sets into headers and loads it into the standard place in a RequestContext.
