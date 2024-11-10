---
title: Adding Basic OAuth Flow to a MPA
date: 2024-11-10T15:09:10-05:00
draft: false
description: I recently added "Sign in with Google" functionality to the https://newdepths.xyz app. Here's an rough overview and some notes about how to do it.
tags:
  - python
  - auth
  - development
  - software-engineering
  - software
categories:
  - software
keywords:
  - fastapi
  - python
  - oauth
  - software development
  - full stack
---

**TL;DR** - For [newdepths.xyz](https://newdepths.xyz) I wanted to add a simpler way for users to log in, in that way they can avoid creating a new password just for this little site. I also wanted to implement all the parts of the flow to get a good feel for them and how they fit together. Of course there are a lot of libraries that handle this well, but I think that doing it manually with low stakes is a great way to learn how everything fits together. 

Here are my notes on how I got this set up, hopefully it may be helpful to someone else in the future.

**What are our goal here?**
- Add a "sign in with Google" button
- When clicked, user gets redirected to authorize with Google (nice! we don't have to handle any password hashing etc)
- Google's server will redirect their browser to our auth callback endpoint (redirect_uri)
- If they approved us and authenticated properly with google, this request has an authorization code that can be exchanged for credentials in the query params
- Our server takes the authorization code and sends it to Google's server to exchange it for user information
- We now have the email for the user, and associated Google id - we can create a new user or log in an existing user as needed in our app, and do whatever else we need to

Here's what the flow looks like on a high level:

![google auth overview](/google_auth/overview.png)

### Google Auth URL
#### Get Client Creds
The first step is to make credentials for your app on the Google Cloud Console - make a project if you don't have one already, and then go to ["Google Auth Platform"](https://console.cloud.google.com/auth). First you have to set up a consent screen - under "Branding", enter your App Name, email, add your apps domains under "Application home page" and "Authorized domains". Here's what mine looks like: 

![consent screen in google cloud console](/google_auth/consent_screen.png)

Once that is all set, you can generate credentials. Specifically we'll need: 
- `client_id`
- `client_secret` 

To make them, go to to "Clients" - pick your app type and fill in the original and authorized redirect URIs for your app. These are important, if they don't match, Google will not redirect the user to your app after the sign in and you'll get a "redirect uri mismatch error". 

Here's what it might look like in local development:

![client set up in google cloud console](/google_auth/client.png)

Of course for the deployed app, the protocol will be `https` and the domain will be the actual domain of the app. 

Once this is created, copy or download your credentials - we'll need to load them into the project in some way (don't put them in version control). We have what we need to build the authorization URL now. 

#### Make the Auth URL
Using the information from our credentials, and the endpoint we want to redirect to (we didn't implement it but it's going to be `<protocol>://<app domain>/auth/callback`). We make the authorization URL like:

```python 
import base64
from urllib.parse import urlencode

def generate_google_auth_url():
  """ Generate a google auth url with our client id and redirect uri.

  We also generate a state token to prevent CSRF attacks and to store
  any non sensitive data we need to send to google and get back (like maybe
  where to redirect the user back to after they authenticate - currently not
  being used in that capacity).

  Returns:
	  state: str - unique state that google auth server will
	         echo back when it sends the client to the callback
	  url: the authorization url to redirect the client to 
  """

  # When Google redirects to our callback - it will echo this
  # random id back to us. If it doesn't match, we know it didn't
  # come from Google
  state = base64.b64encode(str(uuid.uuid4()).encode())
  # This is stored on the client - in NewDepths it's in an http only
  # cookie, but it can be session. There's more information about
  # this at:
  #   https://auth0.com/docs/secure/attack-protection/state-parameters

  baseurl = "https://accounts.google.com"
  path = "/o/oauth2/v2/auth"
  params = {
    # From the creds we just made
    "client_id": GOOGLE_CLIENT_ID,
    # The endpoint we want the browser to be redirected
    # to after auth (were we receive the code) - this
    # needs to match what is in the Cloud Console exactly
    "redirect_uri": GOOGLE_REDIRECT_URI,
	# We want to get back an autorization code
    "response_type": "code",
    # We only need to read their email address
    "scope": "email",
	# This is a unique state to protect against CSRF
    "state": state,
  }
  return state, f"{baseurl}{path}?{urlencode(params)}"
```

#### Authorization Endpoint
Then this will depend on lot on the framework you're using - but we can create an endpoint to trigger this redirect and store the state in a cookie - for example in `FastAPI` it might look something like this: 

```python 

@app.get("/auth/authorize")
def google_auth_authorize():
  """ Redirect the user to the Google OAuth consent screen. """
  state, url = generate_google_auth_url()
  response = RedirectResponse(
    url=url,
    status_code=status.HTTP_307_TEMPORARY_REDIRECT,
  )
  response.set_cookie(
    key="state",
    value=state,
    httponly=True,
    secure=True,
    max_age=60*5
  )
  return response
```

Then anywhere we want to create a button or link to initiate the oauth flow - just link to `/auth/authorize` and navigating there will kick off the OAuth flow.

The final part we need for this puzzle is the endpoint to receive the redirect once the user is done authenticating with the Google server. Again that will look a little different depending on your framework, etc, but in `FastAPI` it might look something like this: 

```python
@app.get("/auth/callback")
def google_auth_callback(
  request: Request,
  code: str = None,
  state: str = None,
  error: str = None
):
  """ Exchange the authorization code for an access token and id token.

    The id_token is a JWT that contains the users email, avatar link, etc. We
    only care about the email, and sub (a unique google id for the user). We don't
    need to make anymore requests to the google api so we're not even going to
    store the access token.
  """
  if error:
    raise HTTPException(...)

  if not state or state != request.cookies.get("state", None):
    raise HTTPException(...)

  # If we're here, the state checks out and we're ready to exchange the code
  # for an access token and id token
  data = {
    'code': code,
    'client_id': GOOGLE_CLIENT_ID,
    'client_secret': GOOGLE_CLIENT_SECRET,
    'redirect_uri': GOOGLE_REDIRECT_URI,
    'grant_type': 'authorization_code'
  }

  try:
    r = requests.post(
      'https://oauth2.googleapis.com/token',
      data=data,
      timeout=5
    )
    if r.status_code != 200:
      logging.error(f"Failed to get token from google: {r.text}")
      raise HTTPException(...)
  except Exception as e:
    logging.critical(f"Failed to get token from google: {e}")
    raise HTTPException(...)

  # Parse the JWT to get info (also verify but we know it's from google)
  if not (info := verify_google_id_token(r.json()['id_token'])):
    raise HTTPException(...)

  # Now `info` contains the user data (email, link to avatar, etc) - store
  # in your database or do whatever you need with it.
  # The response `r` also contains an access token you can use to access
  # Google APIs - if you're trying to do something on their behalf or access
  # more information. In the case of Newdepths we don't need it, Google
  # has auth'd the user for us and we're done.

  # USE `info` for something

  # Don't render a template using this endpoint, instead, redirect
  # your newly registered/logged in user to a new endpoint
```

So that's it - you get back the users info and Google has handled all the password storage / checking for you. It's then up to for your specific case to figure out what you want to do with that information.
#### Persisting Auth State
In the case of the [newdepths](https://newdepths.xyz) app - I make a JWT signed with a secret that's good for some duration and store it in an HTTP only cookie. The rest of the routes on the site check for that cookie and validate it to make sure it's not expired, that it was generated by the server, and  figure out who the user is. I did it this way because it makes both methods look the same to the rest of the app (if they signed up with email or with google, it's the same) - and because I don't need to
ever talk to Google for anything in this case until it's time for them to sign in again (because they signed out or the JWT expired).
#### Useful References
- https://developers.google.com/identity/protocols/oauth2/web-server
- https://developers.google.com/identity/gsi/web/guides/verify-google-id-token
- https://auth0.com/docs/secure/attack-protection/state-parameters
