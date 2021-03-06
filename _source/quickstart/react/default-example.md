---
layout: quickstart_partial
exampleDescription: React Implicit
---

This guide will walk you through integrating authentication into a React app with Okta by performing these steps:

1. Add an OpenID Connect Client in Okta
2. Create a React App
3. Create an Authentication Utility
4. Create a SecureRoute
5. Create Routes
6. Connect the Routes
7. Start Your App

At the end of the React instructions you can choose your server type to learn more about post-authentication workflows, such as verifying tokens that your React application can send to your server.
## Prerequisites
If you do not already have a **Developer Edition Account**, you can create one at [https://developer.okta.com/signup/](https://developer.okta.com/signup/).
## Add an OpenID Connect Client in Okta
In Okta, applications are OpenID Connect clients that can use Okta Authorization servers to authenticate users.  Your Okta Org already has a default authorization server, so you just need to create an OIDC client that will use it.
* Log into the Okta Developer Dashboard, click **Applications** then **Add Application**.
* Choose **Single Page App (SPA)** as the platform, then submit the form the default values, which should look like this:

| Setting             | Value                                       |
| ------------------- | ------------------------------------------- |
| App Name            | My SPA App                                  |
| Base URIs           | http://localhost:3000/                      |
| Login redirect URIs | http://localhost:3000/implicit/callback     |
| Grant Types Allowed | Implicit                                    |

After you have created the application there are two more values you will need to gather:

| Setting       | Where to Find                                                                  |
| ------------- | ------------------------------------------------------------------------------ |
| Client ID     | In the applications list, or on the "General" tab of a specific application.    |
| Org URL       | On the home screen of the developer dashboard, in the upper right.             |


These values will be used in your React application to setup the OIDC flow with Okta.

## Create a React App

To quickly create a React app, install the create-react-app CLI:

```bash
npm install -g create-react-app
```

Now, create a new app:

```bash
create-react-app okta-react-app
```

This creates a new project named `okta-react-app` and installs all required dependencies.  Now change directories into this folder:

```bash
cd okta-react-app
```

Your React app will use the [Okta Auth JS](/code/javascript/okta_auth_sdk) library to redirect the user to the authorization endpoint on your Okta Org. You can install it via npm:

```bash
npm install @okta/okta-auth-js --save
```

You'll also need `react-router-dom` to manage our routes:

```bash
npm install react-router-dom --save
```

## Create an Authentication Utility

You will create a class that encapsulates the interaction with the [Okta Auth JS](/code/javascript/okta_auth_sdk) library. This file will expose a `withAuth` method that makes it easy to create [Higher-Order Components](https://facebook.github.io/react/docs/higher-order-components.html) that include an `auth` property.

To create this file, you will need the values from the OIDC client that you created in the previous step.  You will also need to know your Okta Org URL, which you can see on the home page of the Okta Developer console.

Create a new file `src/auth.js` and add the following code to it, replacing the `{yourOrgUrl}` with your Org URL, and `{clientId}` with the Client ID of the application that you created:

```typescript
// src/auth.js

import React from 'react';
import OktaAuth from '@okta/okta-auth-js';

class Auth {
  constructor() {
    this.login = this.login.bind(this);
    this.logout = this.logout.bind(this);
    this.redirect = this.redirect.bind(this);
    this.getIdToken = this.getIdToken.bind(this);
    this.isAuthenticated = this.isAuthenticated.bind(this);
    this.handleAuthentication = this.handleAuthentication.bind(this);

    this.oktaAuth = new OktaAuth({
      url: 'https://{yourOrgUrl}',
      clientId: '{clientId}',
      issuer: 'https://{yourOrgUrl}/oauth2/default',
      redirectUri: 'http://localhost:3000/implicit/callback',
    });
  }

  getIdToken() {
    return this.oktaAuth.tokenManager.get('idToken');
  }

  login(history) {
    // Redirect to the login page
    history.push('/login');
  }

  async logout(history) {
    this.oktaAuth.tokenManager.clear();
    await this.oktaAuth.signOut();
    history.push('/');
  }

  redirect() {
    // Launches the login redirect.
    this.oktaAuth.token.getWithRedirect({
      responseType: ['id_token', 'token'],
      scopes: ['openid', 'email', 'profile']
    });
  }

  isAuthenticated() {
    // Checks if there is a current accessToken in the TokenManger.
    return !!this.oktaAuth.tokenManager.get('accessToken');
  }

  async handleAuthentication() {
    const tokens = await this.oktaAuth.token.parseFromUrl();
    for (let token of tokens) {
      if (token.idToken) {
        this.oktaAuth.tokenManager.add('idToken', token);
      } else if (token.accessToken) {
        this.oktaAuth.tokenManager.add('accessToken', token);
      }
    }
  }
}

// create a singleton
const auth = new Auth();
export const withAuth = WrappedComponent => props =>
  <WrappedComponent auth={auth} {...props} />;
```

## Create a `SecureRoute` Component
You will want to restrict access to routes in your application and require the user to be authenticated. You will leverage our `auth.js` wrapper to ask Okta if the user is authenticated, and you'll wrap this again in a proper component, called `SecureRoute`, so that you can use it elsewhere.  Place this code into a new `src/SecureRoute.js` file:
{% raw %}
```typescript
// src/SecureRoute.js

import React from 'react';
import { Route, Redirect } from 'react-router';
import { withAuth } from './auth';

export default withAuth(({ auth, component: Component, ...rest }) => (
  <Route {...rest} render={props => (
    auth.isAuthenticated() ? (
      <Component {...props}/>
    ) : (
      <Redirect to={{
        pathname: '/login',
        state: { from: props.location }
      }}/>
    )
  )}/>
));
```
{% endraw %}

## Create Routes

You'll want to create these routes in your sample application:

- `/`: A default home page to handle basic control of the app.
- `/protected`: A route protected by `SecureRoute`.
- `/implicit/callback`: Handle the response from Okta and store the returned tokens.
- `/login`: Redirect to the Okta Org login page.

Follow the next sections to create these routes in your React application.

### `/`
Place the following code in a new file, `src/Home.js`.  This will render a home page and show links to navigate within app:

```typescript
// src/Home.js

import React from 'react';
import { Link } from 'react-router-dom';
import { withRouter } from 'react-router';
import { withAuth } from './auth';

export default withAuth(withRouter(props => {
  // Change the button that's displayed, based on our authentication status
  const button = props.auth.isAuthenticated() ?
    <button onClick={props.auth.logout.bind(null, props.history)}>Logout</button> :
    <button onClick={props.auth.login.bind(null, props.history)}>Login</button>;

  return (
    <div>
      <Link to='/'>Home</Link><br/>
      <Link to='/protected'>Protected</Link><br/>
      {button}
    </div>
  );
}));
```

### `/protected`
Create a new component `src/Protected.js`, this route will only be visible to users with a valid `accessToken`:

```typescript
// src/Protected.js

import React from 'react';
import { withAuth } from './auth';

export default withAuth(props => {
  const token = props.auth.getIdToken();
  return (
    <div>
      <h3>This is a protected view</h3>
      <p>Hello, {token.claims.name}, you are able to see this page because you have authenticated.</p>
      <p>Below, you will see your ID Token from Okta.</p>
      <a href="/">Back to Home</a>
      <hr/>
      <pre>
        {JSON.stringify(token, null, 2) }
      </pre>
    </div>
  )
});
```

### `/implicit/callback`
In order to handle the redirect back from Okta, you need to capture the token values from the URL. You'll use `/implicit/callback` as the callback URL, and again you'll use our `auth.js` to delegate the callback details to the [Okta Auth JS](/code/javascript/okta_auth_sdk) library.

Create a new component `src/Callback.js`:

```typescript
// src/Callback.js

import React, { Component } from 'react';
import { Redirect } from 'react-router-dom';
import { withAuth } from './auth';

export default withAuth(class Callback extends Component {
  state = {
    parsingTokens: false
  }

  componentWillMount() {
    if (window.location.hash) {
      this.setState({
        parsingTokens: true
      });

      this.props.auth.handleAuthentication()
      .then(() => {
        this.setState({
          parsingTokens: false
        });
      })
      .catch(err => {
        console.log('error logging in', err);
      });
    }
  }

  render() {
    if (!this.state.parsingTokens) {
      const pathname = localStorage.getItem('referrerPath') || '/';
      return (
        <Redirect to={pathname}/>
      )
    }

    return null;
  }
});
```

### `/login`
This route redirects to the Okta Authorization URL if the user is not authenticated. If the user is coming from a protected page, they'll be redirected back to the page after authentication.

Create a new component `src/Login.js`:

```typescript
// src/Login.js

import React, { Component } from 'react';
import { Redirect } from 'react-router-dom';
import { withAuth } from './auth';

export default withAuth(class Login extends Component {
  render() {
    let from;
    if (this.props.location && this.props.location.state) {
      from = this.props.location.state.from;
    } else {
      from = { pathname: '/' };
    }

    if (this.props.auth.isAuthenticated()) {
      return <Redirect to={from}/>;
    }

    localStorage.setItem('referrerPath', from.pathname);
    this.props.auth.redirect();
    return null;
  }
});
```

### Connect the Routes
Open `src/App.js` (this was created by the generator) and replace its contents with this code, this will create your final application with the routes that you've created:

```typescript
// src/App.js

import React, { Component } from 'react';
import { BrowserRouter as Router, Route } from 'react-router-dom';
import SecureRoute from './SecureRoute';
import Home from './Home';
import Login from './Login';
import Callback from './Callback';
import Protected from './Protected';

class App extends Component {
  render() {
    return (
      <Router>
        <div>
          <Route path='/' exact={true} component={Home}/>
          <SecureRoute path='/protected' component={Protected}/>
          <Route path='/login' component={Login}/>
          <Route path='/implicit/callback' component={Callback}/>
        </div>
      </Router>
    );
  }
}

export default App;
```

## Start your app
Finally, start your app:

```bash
npm start
```

If all is well, the development server should start and your application will be visible at [http://localhost:3000](http://localhost:3000)! At this point you should be able to sign in and view the protected route.

## Using the Access Token

Your React application now has an access token in local storage that was issued by your Okta Authorization server.  You can read this token and present it to your own server to authenticate requests for resources on your server.  As a hypothetical example, let's say that you have an API that gives us messages for our user.  You could create a `MessageList` component that requires authentication, and uses a new `auth.getAccessToken()` method to get the access token from local storage, and attach it to our resource request.

Please continue down to the next section, Server Setup, to learn about access token validation on the server.  Here is what the React component could look like for this hypothetical example:

```typescript
import fetch from 'isomorphic-fetch';
import React, { Component } from 'react';
import { withRouter } from 'react-router';
import { withAuth } from './auth';

export default withAuth(withRouter(class MessageList extends Component {

  constructor(props) {
    super(props)
    this.state = {
      messages: null
    }
  }

  componentDidMount() {
    if (this.props.auth.isAuthenticated()) {
      fetch('http://localhost:3000/api/messages', {
        headers: {
          Authorization: 'Bearer ' + this.props.auth.getAccessToken().accessToken
        }
      }).then(response => {
        response.json().then(data => {
          this.setState({messages: data.messages});
        });
      }).catch(response => {
        // handle error as needed
      });
    };
  }

  render() {
    if (this.state.messages) {
      const items = this.state.messages.map((message) =>
        <li key={message}>{message}</li>
      );
      return (<ul>{items}</ul>);
    } else {
      return (
        <div>Loading..</div>
      )
    }
  }
}));
```
