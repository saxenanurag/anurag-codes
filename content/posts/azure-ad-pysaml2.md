---
title: Enable Azure AD SSO for your FastAPI Application using PySAML2
description:
date:
draft: true
tags: [python,sso,saml,fastapi,azure]
---

```python
import base64

from fastapi import APIRouter, Depends, Request, Response
from fastapi.responses import RedirectResponse
from starlette.exceptions import HTTPException
from starlette.status import (
    HTTP_200_OK,
    HTTP_202_ACCEPTED,
    HTTP_205_RESET_CONTENT,
    HTTP_303_SEE_OTHER,
    HTTP_400_BAD_REQUEST,
    HTTP_404_NOT_FOUND,
    HTTP_422_UNPROCESSABLE_ENTITY,
)

from app.modules.user_auth import SAMLService, get_saml_service, is_user_logged_in

router = APIRouter()


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
    # return RedirectResponse("/api/login/user-info", status_code=HTTP_303_SEE_OTHER)
    return RedirectResponse("/auth/login/callback", status_code=HTTP_303_SEE_OTHER)


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

```python
import base64
from dataclasses import dataclass
from typing import Optional

from fastapi import Request
from fastapi.responses import RedirectResponse
from loguru import logger
from requests.models import PreparedRequest
from saml2 import BINDING_HTTP_POST, BINDING_HTTP_REDIRECT
from saml2.client import Saml2Client
from saml2.config import Config
from saml2.saml import name_id_from_string

from app.core.config import (
    SSO_ASSERTION_CONSUMER_SERVICE_URL,
    SSO_ENTITY_ID,
    SSO_METADATA_FILEPATH,
)


class RequiresLoginException(Exception):
    def __init__(self, redirect_url: str = None):
        self.redirect_url = redirect_url
        super().__init__()


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


@dataclass
class SAMLService:
    saml_config = load_saml_config()
    saml_client = Saml2Client(config=saml_config)

    def prepare_for_authenticate(self, redirect_url=None):
        """
        Create saml_config based on the latest parameters
        Then, get the redirect url based on the saml config
        """
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
            "employee_id": saml_response.ava["EmpID"][0],
        }
        return user_info

    def is_logged_in(self, sso_name_id):
        return self.saml_client.is_logged_in(name_id=name_id_from_string(sso_name_id))

    def logout(self, sso_name_id):
        # local logout. does not logout from azure
        return self.saml_client.local_logout(name_id=name_id_from_string(sso_name_id))

    def authenticate_user(self, user_attributes):
        return True


def get_saml_service():
    return SAMLService()


def is_user_logged_in(request: Request):
    saml_service = SAMLService()
    if not saml_service.is_logged_in(request.session["saml_name_id"]):
        # redirect_url = request.headers.get("referer")
        redirect_url = str(request.url)
        raise RequiresLoginException(redirect_url=redirect_url)
    else:
        return request.session["user_info"]
```

```python
@app.exception_handler(RequiresLoginException)
async def exception_handler(request: Request, exc: RequiresLoginException) -> Response:
    login_path = "/api/login"
    if exc.redirect_url:
        login_path = (
            f"{login_path}?redirect_url={urllib.parse.quote_plus(exc.redirect_url)}"
        )
    return RedirectResponse(url=login_path)
```
