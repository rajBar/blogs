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

^ was the above really neccessary? Probably not. Right, into the tutorial...

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
        "sex": "<m/f>",
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

There are two intersting things here, the **refresh_token** and the**access_token**.
- Access Token
    - This is the end goal. This is what we need to be able to access our data. However, these are very short lived (about 30 mins of use). So we need
 to be able to refresh this from time to time. If only there were a refresh token
- Refresh Token
    - this is actually what we want right now. It is long lived and allows us to utilise our other long lived credentials to acquire an access token
 whenever we want one
 
### Getting the data
Right we technically have all the data that we need to get our activities. But there is a (very) short workflow required to access said data:
1. Use Refresh Token to generate Access Token.
1. Use Access Token to fetch activity data.
 
#### Getting the Access Token
The first request we're going to make takes the following details:
1. POST request
1. Client ID
1. Client Secret
1. Refresh Token
 
Put the above information into the URL below and put it in Postman (don't forget to make it a POST request)
- https://www.strava.com/oauth/token?client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>&refresh_token=<REFRESH_TOKEN>&grant_type=refresh_token

You'll get returned some JSON that looks like this:
```js
{
    "token_type": "Bearer",
    "access_token": "<ACCESS_TOKEN>",
    "expires_at": 1604157714,
    "expires_in": 14350,
    "refresh_token": "<REFRESH_TOKEN>"
}
```

This request returns some JSON, and inlcuded is the **access_token** - exactly what we'll need to finally get our data.

#### Getting OUR Strava data - finally
The final request that we need to make takes the following details:
1. GET request
1. AccessToken

Run using the info populate the URL below and run in Postman
- https://www.strava.com/api/v3/athlete/activities?access_token=<ACCESS_TOKEN>

You should now see all your activity data, in JSON, ready for you to use in whatever application you want

### Codified for JavaScript
The following pattern was use in my original project. The new project doesn't use the same pattern for some glaring security reasons... but we'll get into that

```js
const fetchData = async (clientID, secret, refreshToken) => {
    let stravaData;
    const authLink = "https://www.strava.com/oauth/token";
    let token = "";

    await fetch(authLink, {
        method: 'post',
        headers: {
            'Accept': 'application/json, text/plain, */*',
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            client_id: clientID,
            client_secret: secret,
            refresh_token: refreshToken,
            grant_type: "refresh_token",
        })
    }).then(res => res.json())
        .then(res => token = res.access_token);

    const activitiesLink = "https://www.strava.com/api/v3/athlete/activities?per_page=200&access_token=" + token;
    await fetch(activitiesLink)
        .then(res => res.json())
        .then(res => {
            stravaData = res;
        });
    return stravaData;
};
```

The code above effectively uses pre-stored data about a user - client_id, client_secret & refresh_token - to get the short lived access token and return the users
activity data from Strava.

You can embed the above code into a simple JS (I used React) application to fetch your data. You can create an array of objects as follows:
```js
const users = {
    raj: {
        athleteID: "<ALTHETE_ID>",
        info: {
            id: "<CLIENT_ID>",
            secret: "<CLIENT_SECRET>",
            refresh: "<REFRESH_TOK>",
        }
    },
    ross: {
        athleteID: "53092595",
        info: {
            id: "<CLIENT_ID>",
            secret: "<CLIENT_SECRET>",
            refresh: "<REFRESH_TOK>",
        }
    },
    cally: {
        athleteID: "59236853",
        info: {
            id: "<CLIENT_ID>",
            secret: "<CLIENT_SECRET>",
            refresh: "<REFRESH_TOK>",
        }
    }
};
```
Then iterate through this list calling the `fetchData()` method. Once you have all the data you can play around with visualisation libs, and display the data however you want.

Feel free to look at my [example](https://raj.bar/strava), and feel free to look at my [code](https://github.com/rajBar/strava)

### Don't use the code above though...
