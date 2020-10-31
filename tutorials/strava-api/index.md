---
layout: blog
title: (One way to) Get Strava's API working
photo: apple-devices.jpg
summary: |
    Strava's docs are quite painful to navigate and I wanted to be able to access my data, so here is a quick overview
    how to access your data and then display it securely!
url: /tutorials/strava-api
date_created: 2020-10-30
---

### Very pointless prelogue

Why bother? Well, at the height of lockdown my friends and I decided we needed some motivation to stay fit. We had gotten used to not really being allowed
to leave the house and suddenly we're allowed out for 30-odd minutes alone to do some exercise - we should jump at the chance right? Meh, we'd gotten used
to sitting around doing nothing...

However, some of us (genuinely not me though) started to really develop a bit of a barrel. So we sucked it up and decided to run a month long triathlon, we'd
record running, cycling and... something else? Right, we quickly realised that something else was too difficult, so we adopted an apparently already created
word - duathlon. The idea was to run 30km and cycle 100km over a month. We all already had Strava, but without paying, you can't make group leaderboards. So
we decided to use use Nike Run Club for the running board and then add the cycle distance - as a developer I felt sick. I decided to work on accessing the 
strava API and automating the whole thing

^ was the above really neccessary? Probably not... right into the tutorial

## The Tutorial

### Pre reqs
- Some knowledge of what a REST API is
- Postman (or equivalent)
- IDE (I use intellij)

### Enabling your Strava Account
The first thing you'll need to do is enable API access to your Strava account.
1. Log into your account [here](https://www.strava.com/).
1. Change the url from https://www.strava.com/ to [https://www.strava.com/settings/api/](https://www.strava.com/settings/api/)
1. Create an 'Application' - honestly here, just put random placeholder information
Right, you've enabled API access on your account and you should now be able to view your **Client ID**, **Client Secret**, 
**Access Token** and **Refresh Token**

### Get somme codes
#### Authorisation Code
Next we'll need to get an 'authorisation code'.
You need to still be logged into strava and the following command (replacing <INSERT_ID_HERE with your **Client ID)>:
- https://www.strava.com/oauth/authorize?client_id=<INSERT_ID_HERE>&redirect_uri=http://localhost&response_type=code&scope=activity:read

The browser will then take you to a new page where you will be able to authorise access from an application to your Strava Data.
Once you select "Authorise", it'll redirect the browser to your localhost domain. Copy and paste the URL - the code we need is in there

The URL will look something like this: http://localhost/?state=&code=BLAHLBLAHBLAHBLAH&scope=read,activity:read

Extract the code where BLAHBLAHBLAHBLAH - this is all we need to keep.

#### Long Lived Token
The next step is to use the authorisation code above, which expires every 30 minutes or so, to generate a Long Lived code. Take the following URL:
- https://www.strava.com/oauth/token?client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>&code=<AUTHORISATION_CODE>&grant_type=authorization_code

As you can imagine, fill in <CLIENT_ID>, <CLIENT_SECRET> & <AUTHORISATION_CODE> accordingly. Then copy the whole URL into Postman
and change the request type to "POST". Then hit send.
- if you took too long between getting the authorisation code and running this command, then you can have a timeout. Quickly go back and generate a new
authorisation code (can quickly press back in the browser a few times till you see the authorise application button) and then extract that code into
the POST request we're trying to do

You'll get returned some JSON that looks like this:
```js
{
    "token_type": "Bearer",
    "expires_at": <sometime>,
    "expires_in": <sometime>,
    "refresh_token": "<REFRESH_TOKEN>",
    "access_token": "<ACCESS_TOKEN>",
    "athlete": {
        "id": someID,
        "username": null,
        "resource_state": 2,
        "firstname": "firstname",
        "lastname": "surname",
        "city": null,
        "state": null,
        "country": null,
        "sex": "M",
        "premium": false,
        "summit": false,
        "created_at": "sometime",
        "updated_at": "sometime",
        "badge_type_id": 0,
        "profile_medium": "avatar/athlete/medium.png",
        "profile": "avatar/athlete/large.png",
        "friend": null,
        "follower": null
    }
}
```

There are two intersting things here, the **access_token 