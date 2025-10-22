---
title: "OAuth 2.0 - Tutorial 1 - Localhost Python WebApp Auth Code Flow with Entra ID"
author: Nathanael Santschi
date: 2025-09-28T05:10:21+01:00
draft: false
tags:
  - OAuth
categories:
  - OAuth

---

## Introduction - "Server Side App" - Auth Code Flow

This guide demonstrates how to create a "server-side" **Python web application** running locally (for development) that authenticates users with **Microsoft Entra ID** and authorizes access to the **Microsoft Graph API** using the **Authorization Code Flow** as a **"confidential client"** with a client secret.

> **Info:**
> In the second tutorial, we will deploy this app to Azure

The client is considered confidential because the app runs solely on the server, and users do not have access to the client secret.

You may first want to check out my previous article on OAuth, which offers a MindMap to give a broad overview of some important OAuth concepts: [OAuth 2.0 - MindMap]({{< relref "posts/2025-09-27-OAuth20-0-Overview.md" >}})

The following image illustrates the Authorization Code Flow with the corresponding OAuth roles, based on [section 4.1 of the OAuth 2.0 specification: RFC6749](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1):
![Authorization Code Flow](/images/oauth-auth-code-grant-entraid.png)

> **Note:** The lines illustrating steps (A), (B), and (C) are broken into
   two parts as they pass through the user-agent.

Here is another high-level view of the authentication flow:
- Although this image shows a native app, in this guide we are only using a Python web app, not a native desktop app.
- The authorization and token endpoints are provided by Microsoft Entra ID.
- The web API being accessed is Microsoft Graph API.

[source: Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/media/v2-oauth2-auth-code-flow/convergence-scenarios-native.svg) 

![OAuth2 Flow](https://learn.microsoft.com/en-us/entra/identity-platform/media/v2-oauth2-auth-code-flow/convergence-scenarios-native.svg)


## Prerequisites
- Microsoft Azure Tenant
- UV installed: [UV Installation](https://docs.astral.sh/uv/getting-started/installation/)

## Python Flask App

You can find tutorials for this scenario using the MSAL library here: [Microsoft: Python Flask MSAL Tutorial](https://learn.microsoft.com/en-us/entra/identity-platform/tutorial-web-app-python-flask-sign-in-out?tabs=workforce-tenant)

Open a console window, create and navigate to your Flask web app folder:
````bash
cd Flask_App
````

Create a new UV project and install the necessary dependencies:
````bash
uv init
uv add flask msal python-dotenv
# Activate the virtual environment with those dependencies installed:
source .venv/bin/activate
````

Then, add the following code into **main.py**:
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
dotenv_path = ".env"  # Update this if your .env file is elsewhere

# Load the .env file
load_dotenv(dotenv_path=dotenv_path)

# Config

config = {
    "authority": f"https://login.microsoftonline.com/{os.getenv('TENANT_ID')}",
    "client_id": os.getenv("CLIENT_ID"),
    "client_secret": os.getenv("CLIENT_SECRET"),
    "redirect_uri": os.getenv("REDIRECT_URI"),
    "scope": ["User.Read"],
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

This step, called "App Registration" in Entra ID, is required so that Microsoft can recognize your application, provide it with a unique client ID, and allow it to securely participate in the authentication and authorization process. Without registering, your app cannot request tokens or access protected resources on behalf of users.

Go to Entra ID and create a new App Registration: 
- **Name:** Choose a name that helps you identify your app.
- **Redirect URI:** The authentication response will be redirected to this URI.

![register application](/images/register-confidential-serversideapp.png "Preview")


After creating the application, complete the following steps:
- Create a **Client Secret** (you can use one for server-side apps)
- Add Microsoft Graph API **User.Read** permissions (if not already present).

Copy the secret immediately and store it securely.

![create client secret](/images/CreateClientSecret.png "Preview")

Verify that your app has the User.Read permission for the Microsoft Graph API:

![graph api user read](/images/app-registration-api-permission-confidentialserver.png "Preview")

Create a **.env** file and add your Azure tenant ID, client ID, the client secret you created, and the redirect URI:

`````
TENANT_ID=******************************************
CLIENT_ID=be4099ac-8a21-****************************
CLIENT_SECRET=LFp8Q~********************************
REDIRECT_URI=http://localhost:5001/callback
`````

## Login + request authorization code
Now we are ready to start and analyze the first part of the auth flow:
![auth-flow-requestauth](/images/auth-flow-requestauth.png "Preview")

Restart your Flask app, open http://localhost:5001 in your browser, and click the login button.
![login](/images/Login-Button.png "Preview")

When you click the **Login with Microsoft** button, the following part of the code will run:

````python
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
`````

This creates the Authorization URL which will look like the following, where you will be redirected to request an authorization code: 

````bash
(1) https://login.microsoftonline.com/b7e2c1a3-9f4d-4e2a-8c1b-3a7d2e5f6b8c/oauth2/v2.0/authorize?
(2)   client_id=c4e7a2b1-5d8f-4c3a-9e2b-6f7d8e9b0c1a&
(3)   response_type=code&
(4)   redirect_uri=http%3A%2F%2Flocalhost%3A5001%2Fcallback&
(5)   scope=User.Read+offline_access+openid+profile&
(6)   state=d9228690-f434-4458-9c55-b41a6271ef4c&
(7)   code_challenge=O-6_i86En7fRDUMwAuxohwhSccQNAZXhMUkB0g59V1A&
(8)   code_challenge_method=S256&
(9)   nonce=1414c2a88bbc97a5294a242052512c95f25096ccc08d8f0cb825d1d4c4c1d1b2&
(10)  client_info=1
````

1. The **authorization server URL** is from our config
    - The MSAL library finds the exact path (/oauth2/v2.0/authorize?)
2. **Client_ID** of our Entra ID App Registration from config
3. The **response_type** "code" is required for the authorization code flow
4. **redirect_uri** of your app (from config), where authentication responses can be sent and received by your app
5. A space-separated list of **scopes** that you want the user to consent to
6. **State** is a randomly generated unique value, typically used for preventing cross-site request forgery attacks
7. **Code_challenge** is used to secure authorization code grants by using Proof Key for Code Exchange (PKCE). Generated by MSAL via a code verifier (random string 43-128 characters) with BASE64-URL-encoding
8. The **code_challenge_method** used to encode the code_verifier for the code_challenge parameter
9. **Nonce** is generated by MSAL and sent in its request for an ID token
10. **client_info** to include extra user and tenant identifiers


As you can see, the login link includes a code challenge, indicating that the PKCE extension is also being used. The main purpose of the PKCE extension is for mobile or native apps where you cannot store a secret but still need to prove the identity of your application.


> **Note:** With OAuth 2.1, it is recommended to use PKCE (Proof Key for Code Exchange) for all Authorization Code flows, including confidential clients. PKCE enhances security by mitigating authorization code interception attacks, even when a client secret is used.


You can find more details here: [Request an authorization code](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow#request-an-authorization-code)


You will need to authenticate and consent to the requested permissions:

![permission-requested](/images/confidential-client-permission-requested.png "Preview")

### Return Authorization code
After user authentication and consent, the authorization endpoint (Entra ID) sends back the query parameters to the /callback endpoint. They are then available within request.args and include the authorization code and state, like:

`````
'code' = '1.AS8AwYrb*******'
'client_info' = 'eyJ1aWQiOiIwMDAwM*****'
'state' = '183d4c********'
'session_state' = '007a8f19-**************'
`````
## Acquire token
Now we come to the part of the flow where the token is acquired:
![token-request](/images/auth-flow-token-request.png "Preview")

The callback endpoint will use the information mentioned above (code, etc.) to acquire a token with the following code:
````python
@app.route("/callback")
def callback():
    msal_app = build_msal_app()
    try:
        result = msal_app.acquire_token_by_auth_code_flow(
            session.get("auth_flow", {}), request.args
        )
````
It basically sends a POST request to the token endpoint with the following information:
````bash
https://login.microsoftonline.com:443 "POST /b7e2c1a3-9f4d-4e2a-8c1b-3a7d2e5f6b8c/oauth2/v2.0/token HTTP/1.1" 200 6780

    "client_id": "c4e7a2b1-5d8f-4c3a-9e2b-6f7d8e9b0c1a",
    "data": {
        "claims": null,
        "client_id": "c4e7a2b1-5d8f-4c3a-9e2b-6f7d8e9b0c1a",
        "code": "1.AS8AwYrb*********",
        "code_verifier": "269il***********",
        "redirect_uri": "http://localhost:5001/callback",
        "scope": [
            "User.Read",
            "openid",
            "profile",
            "offline_access"
        ]
    },
    "environment": "login.microsoftonline.com",
    "grant_type": "authorization_code",
    "params": null,
    "scope": [
        "User.Read",
        "profile",
        "openid",
        "email"
    ],
    "token_endpoint": "https://login.microsoftonline.com/b7e2c1a3-9f4d-4e2a-8c1b-3a7d2e5f6b8c/oauth2/v2.0/token"

````

After sending the token request, we will get an access token, and in our case, also an id_token (OpenID Connect) and a refresh token:
````bash
    "response": {
        "access_token": "********",
        "client_info": "eyJ**************",
        "expires_in": 4968,
        "ext_expires_in": 4968,
        "id_token": "********",
        "refresh_token": "********",
        "scope": "User.Read profile openid email",
        "token_type": "Bearer"
    }
````
## Access API with token
Finally, we come to the last part of the auth flow, where we access the Microsoft Graph API using the access token we acquired earlier:
![api-access](/images/auth-flow-access-api.png "Preview")

Within the second part of the callback function in our Python code, we access the Microsoft Graph API with the access token we acquired earlier:
````python
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
````

In the browser, you will see a welcome page with your user's name and email. This information is retrieved using the Graph API:

![welcome-callback](/images/welcome-callback-page.png "Preview")


That's it for now. 

In the following tutorial, I demonstrate how this app can be deployed to an Azure Web App to become a "real" server-side web app and not just a locally running app: 
- [OAuth 2.0 - Tutorial 2 - Azure Server Side Python WebApp Auth Code Flow - Entra ID]({{< relref "posts/2025-09-27-OAuth20-2-AzureServerSidePythonWebApp-AuthCodeFlow.md" >}})