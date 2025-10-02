---
title: "OAuth 2.0 - Tutorial 1 - Localhost Python WebApp Auth Code Flow with Entra ID"
author: Nathanael Santschi
date: 2025-09-28T05:10:21+01:00
draft: true
tags:
  - OAuth
categories:
  - OAuth

  
---

## Introduction - "Server Side App" - Auth Code Flow

This guide demonstrates how to create a "server-side" **Python web application** running locally (for development, in second part we will deploy it to Azure) that authenticates users with **Microsoft Entra ID** and authorizes access to the **Microsoft Graph API** using the **Authorization Code Flow** as a **"confidential client"** with a client secret.

The client is considered confidential because the app runs solely on the server, and users do not have access to the client secret.

The following image illustrates the Authorization Code Flow with the corresponding OAuth Roles based on [section 4.1 of the OAuth 2.0 specification: RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1)
![Authorization Code Flow](/images/oauth-auth-code-grant-entraid.png)

> **Note:** The lines illustrating steps (A), (B), and (C) are broken into
   two parts as they pass through the user-agent.

Here is another high-level view of the authentication flow:
- Although this image shows a native app, in this guide we are only using a Python web app not a native Desktop app.
- The authorization and token endpoints are provided by Microsoft Entra ID
- The web API being accessed is Microsoft Graph API

[source: Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/media/v2-oauth2-auth-code-flow/convergence-scenarios-native.svg)
![OAuth2 Flow](https://learn.microsoft.com/en-us/entra/identity-platform/media/v2-oauth2-auth-code-flow/convergence-scenarios-native.svg)


## Prerequisites
- Microsoft Azure Tenant
- UV installed: [UV Installation](https://docs.astral.sh/uv/getting-started/installation/)

## Python Flask App
You can find tutorials for this scenario using the MSAL library here: [Microsoft: Python Flask MSAL Tutorial](https://learn.microsoft.com/en-us/entra/identity-platform/tutorial-web-app-python-flask-sign-in-out?tabs=workforce-tenant)

Open a console window, createn and avigate to your Flask web app folder:
````bash
cd Flask_App
````

Create a new UV project and install the necessary dependencies:
````bash
uv init
uv add flask msal python-dotenv
# activate the virtual environment with those dependencies installed:
source .venv/bin/activate
````

Then, add the following code into **main.py**
````python
import logging

logging.basicConfig(level=logging.DEBUG)

from flask import Flask, session, redirect, request, url_for
from dotenv import load_dotenv
import msal
import uuid
import os
import requests

app = Flask(__name__)
app.secret_key = os.urandom(24)  # Needed for session

# Specify the path to the .env file
dotenv_path = ".env"  # Update this with the actual path

# Load the .env file
load_dotenv(dotenv_path=dotenv_path)

# Config
config = {
    "client_id": os.getenv("CLIENT_ID"),
    "authority": f"https://login.microsoftonline.com/{os.getenv('TENANT_ID')}",
    "scope": ["User.Read"],
    "redirect_uri": os.getenv("REDIRECT_URI"),
    "client_secret": os.getenv("CLIENT_SECRET"),
}


def build_msal_app():
    return msal.ConfidentialClientApplication(
        client_id=config["client_id"],
        authority=config["authority"],
        client_credential=config["client_secret"],
    )


@app.route("/")
def index():
    if "user" in session:
        user = session["user"]
        return f"""
        <h2>Welcome {user['displayName']}</h2>
        <p>Email: {user.get('mail') or user.get('userPrincipalName')}</p>
        <a href="/logout">Logout</a>
        """
    return '<a href="/login">Login with Microsoft</a>'


@app.route("/login")
def login():
    msal_app = build_msal_app()
    flow = msal_app.initiate_auth_code_flow(
        scopes=config["scope"],
        redirect_uri=config["redirect_uri"],
        state=str(uuid.uuid4()),
    )
    session["auth_flow"] = flow
    return redirect(flow["auth_uri"])


@app.route("/callback")
def callback():
    msal_app = build_msal_app()
    try:
        result = msal_app.acquire_token_by_auth_code_flow(
            session.get("auth_flow", {}), request.args
        )
    except ValueError:
        return "Authentication failed", 400

    if "access_token" in result:
        session["access_token"] = result["access_token"]

        # Call Microsoft Graph to get user info
        graph_resp = requests.get(
            "https://graph.microsoft.com/v1.0/me",
            headers={"Authorization": f"Bearer {result['access_token']}"},
        )
        if graph_resp.ok:
            session["user"] = graph_resp.json()
            return redirect(url_for("index"))
        else:
            return f"Graph API error: {graph_resp.text}", 500

    return f"Error: {result.get('error_description')}", 400


@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("index"))


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001, debug=True)

````

Start the app (the login link will not work yet):
````bash
flask --app main.py run --host=0.0.0.0 --port=5001
````

Now, we need to register our app in Entra ID to enable login and API access. 

## Create new App Registration
Go to Entra ID and create a new App Registration: 
- **Name:** Choose a name that helps you identify your app.
- **Redirect URI:** The authentication response will be redirected to this URI.

![register application](/images/register-confidential-serversideapp.png "Preview")

After creating the application, complete the following steps:
- Create a **Client Secret** (We can use one for server side apps)
- Add Micrsoft Graph API **User.Read** permissions (if not already present).

Copy the secret immediately and store it securely.

![create client secret](/images/CreateClientSecret.png "Preview")

Verify that your app has the User.Read permission for the Microsoft Graph API:

![graph api user read](/images/app-registration-api-permission-confidentialserver.png "Preview")

Create a **.env** file and add in there your Azure tenant id, client id, your created client secret and the redirect uri:  

`````
TENANT_ID=******************************************
CLIENT_ID=be4099ac-8a21-****************************
CLIENT_SECRET=LFp8Q~********************************
REDIRECT_URI=http://localhost:5001/callback
`````

## Login
Restart your Flask app and try clicking the login button.
You should be redirected to a URL similar to the following:

````bash
https://login.microsoftonline.com/YOUR-TENANT-ID/oauth2/v2.0/authorize?
  client_id=YOUR-CLIENT-ID&
  response_type=code&
  redirect_uri=http%3A%2F%2Flocalhost%3A5001%2Fcallback&
  scope=User.Read+offline_access+openid+profile&
  state=ab75d9bc-3e73-435d-8f6e-12bec5dce5c3&
  code_challenge=2D2aoAkngNH37eJPvjndhjyvJYhlPdcBUXzfPpQXAuQ&
  code_challenge_method=S256&
  nonce=40c7a26b0d3c1baac6473580566a7b390459646ccaa8b8be249aa160eee32aaa&
  client_info=1
````
As you can see, the login link includes a code challenge, indicating that the PKCE extension is also being used. The main purpose of the PKCE extension is for mobile or native apps where you cannot store a secret but still need to prove the identity of your application. 
> **Note:** With OAuth 2.1, it is recommended to use PKCE (Proof Key for Code Exchange) for all Authorization Code flows, including confidential clients. PKCE enhances security by mitigating authorization code interception attacks, even when a client secret is used.

You will need to consent to the requested permissions:

![permission-requested](/images/confidential-client-permission-requested.png "Preview")

After consenting, you will be redirected back to the Flask app, where you will see a welcome page with your Users Name and email. This information is retrieved using the Graph API:

![welcome-callback](/images/welcome-callback-page.png "Preview")

