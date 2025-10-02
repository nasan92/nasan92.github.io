---
title: "OAuth 2.0 - Tutorial 2 - Azure Server Side Python WebApp Auth Code Flow - Entra ID"
author: Nathanael Santschi
date: 2025-09-29T05:10:21+01:00
draft: true
tags:
  - OAuth
categories:
  - OAuth

  
---

## Introduction
In the previous tutorial [Localhost Python WebApp Auth Code Flow with Entra ID](/posts/2025-09-27-OAuth20-1-LocalhostPythonWebApp-AuthCodeFlow)  , we demonstrated using a localhost web app as a "server-side app" (confidential client) with the Authorization Code Flow. In this tutorial, we will deploy the app to Azure, making it a true "server-side app" where users cannot access secrets.

The following steps are required to deploy the app to Azure:
- Create an Azure Container Registry
- Create a Dockerfile and publish the image to the registry
- Create an Azure Web App using this container
- Add environment variables (tenant, client ID, client secret)
- Add a new redirect URI to the app registration
- Update the redirect URI in the app code

## Prerequisites
- Microsoft Azure Tenant with active Subscription
- Docker installed
- Azure CLI installed
- UV installed: [UV Installation](https://docs.astral.sh/uv/getting-started/installation/)

## Create Azure Container Registry
[Create an Azure container registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-portal?tabs=azure-cli) where you can store your docker images:

![Create-azure-container-registry](/images/Azure-create-container-registry.png "Preview")

## Create a Docker Image and publish it to the registry

### requirements.txt 

First, in your app folder (e.g., `flask_app`), create a `requirements.txt` file listing all the dependencies your app will use in the Docker image:

````requirements.txt
flask==3.1.2
msal==1.34.0
python-dotenv==1.1.1
````

### Dockerfile

Next, create a new `Dockerfile` with the following content:

````Dockerfile
FROM python:3.11
WORKDIR /code

COPY requirements.txt .
RUN pip3 install -r requirements.txt 

# copy your actual python code
COPY main.py .

# Add some debugging
RUN echo "Files in /code:" && ls -la /code

# Expose the port that the app will run on
EXPOSE 5001

CMD ["flask", "--app", "main.py", "run", "--host", "0.0.0.0", "--port", "5001"]
````

### Python Source Code

The actual code for the Python Flask sample web app is largely the same as in the previous tutorial:

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

# Load the .env file -> is not used in "prod" web app only on local development
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

### Build Docker Image

**Note:** On macOS, it is necessary to build the Docker image explicitly for `linux/amd64`:

````bash
IMG_NAME=flask-app
cd flask_app && \
docker buildx build --platform linux/amd64 -t ${IMG_NAME} .
````


Log in to your Azure Container Registry:
````bash
ACR_NAME=nasatestreg
az login # First Login with the Azure CLI
az acr login --name $(ACR_NAME) # Login to you specific Azure Container Registry
````


Now, tag and push your image to your Azure Container Registry:
````bash
docker tag ${IMG_NAME} $(ACR_NAME).azurecr.io/${IMG_NAME}:v5
docker push $(ACR_NAME).azurecr.io/${IMG_NAME}:v5
````

## Create an Azure Web App

Next, create an Azure Web App:

![Create-azure-web-app](/images/Azure-create-web-app.png "Preview")


Use the container image you created earlier:

![Create-azure-web-app-container](/images/Azure-Create-Web-App-container.png "Preview")


## Configure Environment Variables 

We use environment variables to provide client information such as client ID, secret, tenant id and redirect URI. Add them with the correct names to your app:

![Create-azure-web-app-env](/images/Azure-web-app-environment-variables.png "Preview")


## New Redirect URI for your App Registration
Your app registration (created in part 1: [Localhost Python WebApp Auth Code Flow with Entra ID](/posts/2025-09-27-OAuth20-1-LocalhostPythonWebApp-AuthCodeFlow) ) only had a redirect URI pointing to your localhost address. Since the app now runs on an Azure Web App Service, you need to update this:

Find your new URL in the Azure Web App's Overview section under Default Domain (for simplicity, we use the default domain). Copy this domain and go to the app registration you created in the previous tutorial. Replace `localhost` with your new URI and add `/callback` (as used in the Python code).

![Create-azure-redirect-uri](/images/Azure-App-Redirect-URI.png "Preview")


## Test the OAuth flow
![login](/images/azure-web-app-loginscreen.png "Preview")

Link it redirects you:
````
https://login.microsoftonline.com/YOUR-TENANT-ID/oauth2/v2.0/authorize?
    client_id=be4099ac-8a21-4cb1-941e-dbd62563b379&
    response_type=code&
    scope=User.Read+offline_access+openid+profile&
    state=92ec2536-8bb8-4c68-9c52-81fc7718c26b&
    code_challenge=QW-jO9BfBjCilaVtntzgptICnjrXnY4fnThJEfWEcMo&
    code_challenge_method=S256&
    nonce=e2671ebbd5e72c8c36ee98b842db7ee39dd1bbb2321b2797b80a2504872e9a09&
    client_info=1&
    sso_reload=true
````

![ms login](/images/oauth-example-login-screen.png "Preview")


This time, it does not ask for consent again because you already granted consent for the same app in the previous tutorial.

![welcome-callback](/images/welcome-callback-page.png "Preview")