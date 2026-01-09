# Geocaching Adventure Labs

This is an unofficial documentation of how the Geocaching Adventure Labs API and mobile app work. It is based on the Android version 1.56.0 of the official [Groundspeak Adventure Labs app](https://play.google.com/store/apps/details?id=com.groundspeak.react.adventures). This documentation is incomplete.

## Basics

The app uses SSL/TLS with certificate pinning so watching the network traffic is a bit challenging. I suggest using [HTTP Toolkit](https://httptoolkit.com/)

For version 1.56.0 of the Android app,\
The user-agent is: `Adventures/1.56.0 (4936) (android/32)`\
The consumerKey is `A01A9CA1-29E0-46BD-A270-9D894A527B91`

By default, responses will be in JSON format. With the "Accept" header set, you can also get XML format.

### X-Consumer-Key header

Update as of July 2021: all requests must now contain the `X-Consumer-Key` header, set to the consumerKey.

    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91

## Authentication

Authentication is done with the “Authorization” header however most data is available without authentication. Authentication is required to check answers, claim finds and post logs and ratings.

### Login Request

    POST https://api.groundspeak.com/adventuresmobile/v1/public/accounts/login HTTP/1.1
    accept: application/json
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Content-Type: application/json
    Content-Length: ***
    Host: api.groundspeak.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91
      
    {"Username":"*username*","Password":"*password*"}

#### Response (failed login)

    HTTP/1.1 400 Bad Request
    Cache-Control: no-cache
    Content-Type: application/json; charset=utf-8
    Content-Length: 31
    
    "Invalid username or password."

#### Response (login successful)

    HTTP/1.1 200 OK
    Cache-Control: no-cache
    Content-Type: application/json; charset=utf-8
    Content-Length: ***
    
    {"accessToken":"*access_token*","refreshToken":"*refresh_token*","expiresIn":3599}

The access_token is then used in the Authorization header with the "bearer" prefix:
`Authorization: bearer *access_token*`
After the *expires_in* time (in seconds) has elapsed, the access_token needs to be refreshed. This is done as shown below using the refresh_token from the previous login response.

### Refresh Access Token

    POST https://labs-api.geocaching.com/Api/Accounts/RefreshAccessToken?consumerKey=*consumerKey* HTTP/1.1
    accept: application/json
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Content-Type: application/json
    Content-Length: ***
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91
    
    {"RefreshToken":"*refresh_token*"}

The response will be similar to the login response

### Get user account details

Returns username, user ID number, user public GUID, user find count and URL to user avatar.

    GET https://labs-api.geocaching.com/Api/Accounts/GetAccount HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91

## Searching for Adventure Labs

Each Adventure Lab is identified by it's unique GUID, for example: `5556bf6f-55fb-4e1b-a952-3633044f5be0`

### Searching by coordinates

    POST https://api.groundspeak.com/adventuresmobile/v1/public/adventures/search HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Content-Type: application/json
    Content-Length: ***
    Host: api.groundspeak.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91

    {
      "Origin": {
        "Latitude": 51.5,
        "Longitude": 0.0
      },
      "RadiusInMeters": 10000,
      "Take": 300,
      "CompletionStatuses": [],
      "OnlyHighlyRecommended": false,
      "AdventureTypes": [],
      "MedianCompletionTimes": [],
      "Themes": [],
      "ExcludeOwned": false
    }

This will return a list of Adventure Lab caches within the search radius. Basic information is also provided such as title, location, description, owner and rating.

### Smart Links

These are special links to lab caches that are designed to be shared on Facebook, geocache pages and in QR codes. They look like this: `https://labs.geocaching.com/goto/*`
The Adventure Labs app can be set as the default app for links in this format. They are also called "Deep Links" sometimes.

The part of the link after the `/goto/` is called the "Smart Name" and is unique to that lab cache. Originally the lab cache owner was able to choose the Smart Name (if their desired name wasn't already taken) however it now appears to be that, for new caches, the Smart Name is a "GUID-format-like" string (note: not the same as the GUID of the Adventure Lab cache).

To convert the Smart Name to the GUID

    POST https://labs-api.geocaching.com/Api/Adventures/GetAdventureIdBySmartLink HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Content-Type: application/json
    Content-Length: ***
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91
    
    "*SmartName*"

Note that the Smart Name in the post data is enclosed in double quotes.
The response with be the GUID of the Adventure Lab cache.

## Get the cache details

Once we have the GUID of the Adventure Lab we are interested in, we can load the full details.

    GET https://api.groundspeak.com/adventuresmobile/v1/public/adventures/*GUID_GOES_HERE* HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Host: api.groundspeak.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91

The response to this is mostly self-explanatory but it does vary slightly if you are logged in. If logged in, you get information on which stage you have already completed and a bit of data named `findCodeHashBase16v2` which is required to check answers to the question.

There is also `answerCodeHashesBase16v2`. I don't know what the difference is, it appears to be the same as `findCodeHashBase16v2` (which is also the same as `findCodeHashBase16` from the previous version)

### Past logs

This request is used to get a list of logs that past finders have written for this Adventure Lab cache:

    POST https://api.groundspeak.com/adventuresmobile/v1/public/adventures/*GUID_GOES_HERE*/reviews/search?skip=0&take=20 HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Content-Length: 0
    Host: api.groundspeak.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91

Note that this is a POST request with no body. Not sure why.

## Checking answers to stages

When the user enters the answer to a stage it is first checked locally (by comparing hashes) and if it is correct, it is then sent to the server. The server also checks that submitted answers are correct.

### Local (hash) check (v1.2.15)

The `FindCodeHashBase16` value returned in the cache details is a MD5 hash of the user's public GUID (from the Get user account details request) and the answer concatenated together. Spaces are removed and all uppercase letters are converted to lowercase. The algorithm is shown below.

    answer_nospace = answer.replace(" ", "");  // Remove spaces
    hash_input = publicGuid + answer_nospace;  // Concatenate strings
    hash_input = hash_input.toLowerCase();     // Convert to all lowercase
    hash_result = md5(hash_input);             // md5 hash is used
    if (hash_result == FindCodeHashBase16) {
        // Answer is correct
    }

If the hash does not match, the app then replaces any occurrences of `"1920"` in the answer with a single apostrophe `"'"` and checks the hash again. If this also fails, the reverse is tried, any apostrophes are replaced with the string `"1920"` and the hash calculated and checked again. If this fails as well, then the answer is deemed to be incorrect.

The space removal is done with a Regex match to `\s` (the JavaScript variant) so tabs, non-breaking spaces, ideographic spaces, etc. are also removed.

#### Versions

The above is from version 1.2.15 of the app. In version 1.56.0, it only gives `findCodeHashBase16v2` and `answerCodeHashesBase16v2`

These three values all appear to be the same on all the ALs I've looked at so I don't know what the difference between them is. But the hash check method from version 1.2.15 still seems to work.

### Server check

After the answer has been confirmed correct by the hash check, it is then sent to server. This marks the stage as found for the user and increments their find count.

    POST https://api.groundspeak.com/adventuresmobile/v1/submitAnswer HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Content-Type: application/json
    Content-Length: ***
    Host: api.groundspeak.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91

    {
      "AdventureGuid":"*lab_guid*",
      "StageGuid":"*stage_guid*",
      "Answer":"*answer*",
      "ChallengeType": "SingleChoice"
    }

The answer has the spaces removed and is converted to lowercase.

The GUIDs are found in the cache details. ChallengeType can be `"SingleChoice"` or `"MultiChoice"`

If the answer is correct, the response will look like this:

    {
      "result": "Success",
      "journalMessage": "Well done",
      "adventureComplete": false
    }

## Writing Logs and Ratings

### Submitting logs, ratings and recommended

To submit your own log to the adventure lab, use the following HTTP request. This will overwrite any existing log the user has for the given adventure lab (ie. you can't log the same adventure lab twice, if you try, it will only show the most recent one).

    PUT https://api.groundspeak.com/adventuresmobile/v2/adventures/*GUID_GOES_HERE*/reviews HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.56.0 (4936) (android/32)
    Content-Type: application/json
    Content-Length: ***
    Host: api.groundspeak.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91

    {
      "Rating": 5,
      "ReviewText": "Your log text goes here",
      "Recommended": false,
      "PlayerTags": []
    }

## More info

This document is still a work in progress. If you have any further info on how the Adventure Labs app works, please do get in contact.
