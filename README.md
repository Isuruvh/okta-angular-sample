# Okta Angular SPA with PKCE Redirect Flow

This project is a minimal Angular Single-Page Application that verifies the Okta redirect-based sign-in flow using OAuth 2.0 Authorization Code with PKCE. The app redirects the user to Okta's hosted sign-in page, authenticates them, and then returns to the Angular app at the callback route.

This sample demonstrates the recommended SPA pattern for Okta authentication:

- Authorization Code + PKCE for browser-based apps
- Okta redirect flow to the hosted sign-in page
- Callback handling with `OktaCallbackComponent`
- Access-token attachment using an HTTP interceptor
- Route protection with `canActivateAuthGuard`
- User profile retrieval after login

## What this app does

When the user clicks Login, the app calls `signInWithRedirect()`. Okta then redirects the browser to the Okta-hosted sign-in page where the user enters credentials. After successful authentication, Okta redirects the browser back to:

- `http://localhost:4200/login/callback`

The app processes the callback and completes the token exchange. If authentication succeeds, the Angular routes behind `canActivateAuthGuard` become available.

## PKCE and redirect flow

This sample uses the Authorization Code flow with PKCE, which is the standard secure method for single-page applications running in a browser.

The sequence is:

1. The Angular app creates an `OktaAuth` instance.
2. The user clicks Login.
3. The browser is redirected to Okta.
4. Okta shows the hosted sign-in page.
5. The user authenticates.
6. Okta redirects back to the configured callback URL.
7. The app exchanges the code for tokens on the client side using PKCE.
8. The app stores the tokens and uses them for API access.

In this repository, the flow is configured in `src/app/app.config.ts` and the callback route is registered in `src/app/app.routes.ts`.

## Prerequisites

Before running the app, you need:

- An Okta Developer org
- A Single-Page Application app created in Okta
- A valid Okta issuer URL
- A valid Okta client ID
- An Angular app running locally on `http://localhost:4200`

## Okta app configuration

Create an app in Okta as follows:

1. Sign in to your Okta Admin Console.
2. Go to Applications > Applications.
3. Click Create App Integration.
4. Choose:
   - Sign-in method: OIDC - OpenID Connect
   - Application type: Single-Page Application
5. Click Next.
6. Give the app a name such as `Angular PKCE SPA`.
7. Under Grant type, enable:
   - Authorization Code
   - Refresh Token
8. Add redirect URIs:
   - Sign-in redirect URI: `http://localhost:4200/login/callback`
   - Sign-out redirect URI: `http://localhost:4200`
9. Save the app.

If you use a custom authorization server, make sure it has a policy and rule that includes Authorization Code grant. The default authorization server is usually sufficient for development unless your org has custom restrictions.

## Configure the app

Update the Okta configuration in `src/app/app.config.ts`:

```ts
const oktaAuth = new OktaAuth({
  clientId: '{yourClientId}',
  issuer: 'https://{yourOktaDomain}/oauth2/default',
  redirectUri: 'http://localhost:4200/login/callback',
  scopes: ['openid', 'profile', 'email', 'offline_access'],
});
```

### Where to find the values

After creating the app, open the app’s General tab to collect:

- Client ID: under Client Credentials
- Issuer: from the Authorization Server / Issuer URI

You may also need to add `http://localhost:4200` as a Trusted Origin in Okta if your org requires it.

## Local setup

Clone the project and install dependencies:

```bash
git clone <your-repo-url>
cd okta-angular-sample
npm install
```

Start the Angular app:

```bash
npm start
```

Then open:

```text
http://localhost:4200
```

## How the sign-in flow works in this app

### Home page

The home page is served by `src/app/home/home.component.ts` and includes a `Login` action. Clicking it triggers:

```ts
await this.oktaAuth.signInWithRedirect();
```

That command redirects the browser to the Okta login page.

### Callback route

The route `login/callback` is configured in `src/app/app.routes.ts` with `OktaCallbackComponent`:

```ts
{ path: 'login/callback', component: OktaCallbackComponent }
```

This callback component finishes the login response and processes the tokens returned from Okta.

### Protected routes

Protected pages use `canActivateAuthGuard`:

```ts
{ path: 'profile', component: ProfileComponent, canActivate: [canActivateAuthGuard] }
```

If the user is not authenticated, the guard redirects them to Okta.

### Access token injection

The app uses an HTTP interceptor in `src/app/auth.interceptor.ts` to attach the bearer token to safe local API requests:

```ts
const accessToken = oktaAuth.getAccessToken();
```

The interceptor adds the `Authorization: Bearer <token>` header when the request is to a local origin.

## Project structure

```text
src/
  app/
    app.config.ts          # OktaAuth configuration
    app.routes.ts          # Angular routes and callback registration
    auth.interceptor.ts    # Adds access token to outgoing requests
    home/
      home.component.ts    # Login trigger and auth state
    profile/
    messages/
    protected/
```

## Verifying the flow manually

To verify this setup works end-to-end:

1. Start the app with `npm start`.
2. Open `http://localhost:4200`.
3. Click Login.
4. You should be redirected to Okta’s hosted sign-in page.
5. Sign in with a valid Okta user.
6. After successful authentication, Okta redirects back to `http://localhost:4200/login/callback`.
7. The app should complete the login and show authenticated state.
8. Navigate to a protected route such as `/profile` or `/messages`.
9. Confirm the route loads only after authentication succeeds.

### Expected behavior

- A logged-out user sees the home page and login option.
- After successful login, the app shows the authenticated user state.
- Protected routes are accessible only after the auth flow completes.
- The access token is sent in the Authorization header for allowed requests.

## Troubleshooting

### Redirect URI mismatch

If login fails with an OAuth redirect error, verify the redirect URI exactly matches:

- `http://localhost:4200/login/callback`

It must match the value configured in Okta and the app configuration.

### Invalid issuer or client ID

If Okta rejects the request, confirm that:

- `clientId` is correct
- `issuer` points to the correct Okta authorization server
- the app is created as a Single-Page Application

### User is not redirected back properly

Ensure that:

- the Angular app is running on the correct port
- no other route is intercepting `/login/callback`
- the browser is not blocking the redirect

### Access token missing on API calls

If outgoing requests do not include a bearer token:

- confirm the user is authenticated
- check `oktaAuth.getAccessToken()` is returning a value
- ensure the target URL matches an allowed origin in the interceptor

## Running the tests

This project includes Angular specs for basic auth-related behavior. To run them:

```bash
npm run test
```

## Related references

- Okta Developer Docs: OAuth 2.0 and OIDC
- Okta Angular SDK documentation
- Authorization Code with PKCE for SPAs

## Notes

This sample uses the Okta-hosted redirect model rather than an embedded Sign-In Widget. In other words, the user is redirected to Okta for authentication instead of embedding the widget directly inside the Angular page. The redirect approach is the most common and recommended setup for SPAs using PKCE.

If you want, this project can also be extended with:

- logout handling
- refresh token management
- custom login UX
- backend protected API integration
