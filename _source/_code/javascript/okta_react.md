---
layout: docs_page
title: Okta with React
weight: 20
excerpt: Integrate Okta with a React application.
---

# Overview

You can easily add authentication to your React single-page app (SPA). Follow along to learn how.

## Here's what you'll do in this quickstart:

- Create an OpenID Connect app in Okta
- Connect your React app to this OpenId Connect 

## Prerequisites

- An Okta org - Register for [Okta Developer Edition](https://www.okta.com/developer/signup/) if you don't have an existing org
- [Node.js](https://nodejs.org/en/download/)

## Create an Authorization Server

This is optional. We'll be using Okta's organization authorization server to make setup easy, but it's less flexible than a custom authorization server. Most SPAs use access tokens instead of cookies to access APIs. If you're building an API that will need to accept access tokens, [create an authorization server](https://developer.okta.com/docs/how-to/set-up-auth-server.html#create-an-authorization-server).

## Create an OpenID Connect App

In order to connect to your authorization server, you need to create an app using the [App Integration Wizard](https://help.okta.com/en/prev/Content/Topics/Apps/Apps_App_Integration_Wizard.htm#OIDCWizard) with these settings:

- **platform** - Single-Page App (SPA)
- **redirect URI** - `http://localhost:3000/callback`

Copy the **client id**, because we'll need it later.

## Enable CORS for your Domain

For security reasons, browsers make it difficult to make requests to other domains. In this example, we'll make requests from `http://localhost:3000` to `https://your-okta-domain.com`. We can configure `https://your-okta-domain.com` to accept our requests by [enabling CORS for `http://localhost:4000`](/docs/api/getting_started/enabling_cors.html#granting-cross-origin-access-to-websites).

## Create a React App

To quickly create a React app, we'll use `create-react-app`:

```bash
npm install -g create-react-app

create-react-app okta-app
cd okta-app/

# @okta/okta-auth-js will help retrieve and manage our tokens
# history and react-router-dom are to help manage our SPA's url location
npm install @okta/okta-auth-js history react-router-dom --save

npm start
```

Your app is now hosted on `http://localhost:3000`.

## Create an Authentication Utility

In order to authenticate from anywhere in our app, we should create an Auth utility. First, we need to create a history singleton, so our React app can change the url.

```javascript
// src/history.js

import createBrowserHistory from 'history/createBrowserHistory';

export default createBrowserHistory();
```

Now, we can create an Auth utility that manages our authentication flows.

```javascript
// src/Auth.js

import OktaAuth from 'okta-auth-js';
import history from './history';

export default class Auth {
  constructor() {
    this.login = this.login.bind(this);
    this.logout = this.logout.bind(this);
    this.handleAuthentication = this.handleAuthentication.bind(this);
    this.isAuthenticated = this.isAuthenticated.bind(this);
  }

  okta = new OktaAuth({
    url: 'https://your-org.okta.com',
    clientId: 'your-client-id',
    redirectUri: 'http://localhost:3000/callback'
  });

  login() {
    this.okta.getWithRedirect({
      responseType: ['id_token', 'token']
    });
  }

  async logout() {
    this.okta.tokenManager.clear();
    await this.okta.signOut();
  }

  async handleAuthentication() {
    const tokens = await this.okta.parseFromUrl();
    for (let token of tokens) {
      if (token.idToken) {
        this.okta.tokenManager.set('idToken', token);
      } else if (token.accessToken) {
        this.okta.tokenManager.set('accessToken', token);
      }
    }
  }

  isAuthenticated() {
    return !!this.okta.tokenManager.get('idToken');
  }
}
```

## Add Routes
