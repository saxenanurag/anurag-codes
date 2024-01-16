---
title: Integrate Azure AD SSO with SAML for your FastAPI Application using PySAML2
description: How to add Azure AD SSO with SAML for your FastAPI Application
date: 2024-01-15
draft: false
tags: [python,sso,saml,fastapi,azure]
---

Recently I had to work on a Azure AD SAML integration for a FastAPI application.
I looked around the internet for a how to and stitched together a solution from
multiple sources. I am writing this post as a how to on integrating Azure AD SAML
SSO into a FastAPI application so that there is a single source for others to use.
I am using PySAML2 and I am sure there are better ways to do this but this is how
I did it. This integration is written for a FastAPI application but since PySAML2 is
a library seperate from the Web Framework being used, you could use this as a guide
to integrate PySAML2 with Flask etc.

You will need to get a metadata file from your Azure AD Administrator.
It will be an XML file built for your application.

Install PySAML2 as decsribed in the [docs](https://pysaml2.readthedocs.io/en/latest/).

Also install xmlsec1 as stated in the [prerequisites](https://pysaml2.readthedocs.io/en/latest/install.html#prerequisites).
If you are using homebrew you can simply install by running `brew install libxmlsec1`.

## Setup the class

All the SSO work can be done using one class.

```python

import base64

from saml2 import BINDING_HTTP_POST, BINDING_HTTP_REDIRECT
from saml2.client import Saml2Client
from saml2.config import Config
from saml2.saml import name_id_from_string

@dataclass
class SAMLService:
    saml_config = load_saml_config()
    saml_client = Saml2Client(config=saml_config)

    def prepare_for_authenticate(self, redirect_url=None):
        relay_state = ""
        if redirect_url:
            relay_state = base64.b64encode(redirect_url.encode("utf-8"))
        request_id, info = self.saml_client.prepare_for_authenticate(
            relay_state=relay_state
        )
        for key, value in info["headers"]:
            if key == "Location":
                sso_redirect_url = value
                return sso_redirect_url
        return None

    def process_saml_response(self, saml_response_data):
        # Parse and process the SAML response
        authn_response = self.saml_client.parse_authn_request_response(
            saml_response_data,
            BINDING_HTTP_POST,
        )
        authn_response.get_identity()
        authn_response.get_subject()
        return authn_response

    def get_user_info(self, saml_response):
        user_info = {
            "email": saml_response.ava["Email"][0],
            "first_name": saml_response.ava["Firstname"][0],
            "last_name": saml_response.ava["Lastname"][0],
        }
        return user_info

    def is_logged_in(self, sso_name_id):
        return self.saml_client.is_logged_in(name_id=name_id_from_string(sso_name_id))

    def logout(self, sso_name_id):
        # local logout. does not logout from azure
        return self.saml_client.local_logout(name_id=name_id_from_string(sso_name_id))

    def authenticate_user(self, user_attributes):
        # placeholder to verify user roles etc
        return True

class RequiresLoginException(Exception):
    def __init__(self, redirect_url: str = None):
        self.redirect_url = redirect_url
        super().__init__()

def get_saml_service():
    return SAMLService()

def is_user_logged_in(request: Request):
    saml_service = SAMLService()
    if not saml_service.is_logged_in(request.session["saml_name_id"]):
        redirect_url = str(request.url)
        raise RequiresLoginException(redirect_url=redirect_url)
    else:
        return request.session["user_info"]
```

And the method to load the `saml_config`:

```python

def load_saml_config():
    settings = {
        "metadata": {"local": [SSO_METADATA_FILEPATH]},
        "service": {
            "sp": {
                "endpoints": {
                    "assertion_consumer_service": [
                        (SSO_ASSERTION_CONSUMER_SERVICE_URL, BINDING_HTTP_REDIRECT),
                        (SSO_ASSERTION_CONSUMER_SERVICE_URL, BINDING_HTTP_POST),
                    ],
                },
                "allow_unsolicited": True,
                "authn_requests_signed": False,
                "logout_requests_signed": True,
                "want_assertions_signed": True,
                "want_response_signed": False,
            },
        },
    }
    saml_config = Config()
    saml_config.load(settings)
    saml_config.allow_unknown_attributes = True
    saml_config.entityid = SSO_ENTITY_ID
    return saml_config
```

The `load_saml_config` method returns a saml config class instance with the settings dict loaded.
You may find the Assertion Service Url (ACS) in the metadata file. The `SSO_ENTITY_ID` will be the
entity id created during the setup on the Azure AD Admin account.

The `prepare_for_authenticate` method uses the `prepare_for_authenticate` method on the saml client class
to get the `request_id` and `info` dictionary back. The `header` key in the `info` dict has a `Location` key
which has the redirect url with the SAML request. This redirect url is where you will redirect the user to login.
If you want to preserve the state of the url where the authentication exception originated in the application,
you can encode it and pass it as a `relay_state` string in the `prepare_for_authenticate` method on the saml
client class. This will be added to the redirect url to azure as a `RelayState` named param in the redirect url
which is simply returned back as `RelayState` paramter in the SAML Response from Azure AD.

The `process_saml_response` processes the SAML response recieved from Azure AD after successful login
and returns a dictionary with the SAML attributes that are returned from Azure. This can include user name,
email address, etc.

Other methods are described as defined above. The `is_user_logged_in` method is used as a check to see
wether the user is present in the SAML cache and if its still valid. If the user is not valid in the cache,
a `RequiresLoginException` is raised which gets bubbled up to the FastAPI app level and then gets handled
as described after the FastAPI route section below.

## FastAPI Routes

On the FastAPI side, you can define routes and import the SAMLService class as a dependency using the
[FastAPI Dependency Injection](https://fastapi.tiangolo.com/tutorial/dependencies/) feature. The `starlette.status`
class has the http status code literals used below so import those if you want to use them.

The callback route is where the SAMLResponse comes in after a successful login in Azure AD.

We also rely on a signed session cookie from the client to store saml attributes and user info.
Since FastAPI is written on top of Starlette, you can use
[Starlette's Session Middleware](https://www.starlette.io/middleware/#sessionmiddleware)
feature for this.

```python

@router.get("/login")
def saml_login(
    redirect_url: str = None, saml_service: SAMLService = Depends(get_saml_service)
):
    authn_request_url = saml_service.prepare_for_authenticate(redirect_url=redirect_url)
    if authn_request_url:
        headers = {"Cache-Control": "no-cache, no-store", "Pragma": "no-cache"}
        return RedirectResponse(url=authn_request_url, headers=headers)
    else:
        raise HTTPException(status_code=HTTP_400_BAD_REQUEST, detail="Unable to login")

@router.post("/login/callback")
async def saml_callback(
    request: Request, saml_service: SAMLService = Depends(get_saml_service)
):
    form_data = await request.form()
    saml_response_data = form_data.get("SAMLResponse")
    if not saml_response_data:
        raise HTTPException(
            status_code=HTTP_400_BAD_REQUEST, detail="SAMLResponse not found"
        )
    saml_response = saml_service.process_saml_response(saml_response_data)
    request.session["saml_attributes"] = saml_response.ava
    request.session["saml_name_id"] = str(saml_response.name_id)
    request.session["user_info"] = saml_service.get_user_info(saml_response)
    if form_data.get("RelayState"):
        return_path = base64.b64decode(form_data.get("RelayState")).decode("utf-8")
        return RedirectResponse(return_path, status_code=HTTP_303_SEE_OTHER)
    return RedirectResponse("/api/login/user-info", status_code=HTTP_303_SEE_OTHER)


@router.get("/login/user-info")
async def check_user(request: Request):
    return is_user_logged_in(request)


@router.get("/login/logout")
async def logout_user(
    request: Request, saml_service: SAMLService = Depends(get_saml_service)
):
    saml_name_id = request.session["saml_name_id"]
    if saml_service.logout(saml_name_id):
        request.session.pop("saml_attributes")
        request.session.pop("saml_name_id")
        request.session.pop("user_info")
        return Response(status_code=HTTP_205_RESET_CONTENT)
```

Handling `RequiresLoginException``:

```python
# in main.py or where ever you define the
# fastapi app
@app.exception_handler(RequiresLoginException)
async def exception_handler(request: Request, exc: RequiresLoginException) -> Response:
    login_path = "/api/login"
    if exc.redirect_url:
        login_path = (
            f"{login_path}?redirect_url={urllib.parse.quote_plus(exc.redirect_url)}"
        )
    return RedirectResponse(url=login_path)
```

SAML authenticaiton does have a drawback of the inability to authenticate with the Identity Provider
(Azure AD in this case) without user redirection. If the user has to be redirected for re-authentication
the request state has to be reserved and can be set using the `RelayState` parameter.
