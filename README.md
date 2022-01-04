# Geocaching Adventure Labs

This is an unofficial documentation of how the Geocaching Adventure Labs API and mobile app work. It is based on the Android version 1.2.15 of the official [Groundspeak Adventure Labs app](https://play.google.com/store/apps/details?id=com.groundspeak.react.adventures). This documentation is incomplete.

## Basics

The app uses SSL/TLS with certificate pinning so watching the network traffic is a bit challenging. The server is at [https://labs-api.geocaching.com](https://labs-api.geocaching.com)

For version 1.2.15 of the Android app,\
The user-agent is: `Adventures/1.2.15 (1822) (android/30)`\
The consumerKey is `A01A9CA1-29E0-46BD-A270-9D894A527B91`

By default, responses will be in JSON format. With the "Accept" header set, you can also get XML format.

### X-Consumer-Key header

Update as of July 2021: all requests must now contain the `X-Consumer-Key` header, set to the consumerKey.

    X-Consumer-Key: A01A9CA1-29E0-46BD-A270-9D894A527B91

## Authentication

Authentication is done with the “Authorization” header however most data is available without authentication. Authentication is required to check answers, claim finds and post logs and ratings.

### Login Request

    POST https://labs-api.geocaching.com/Api/Accounts/Login?consumerKey=*consumerKey* HTTP/1.1
    accept: application/json
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Content-Type: application/json
    Content-Length: ***
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
      
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
    
    {"access_token":"*access_token*","refresh_token":"*refresh_token*","expires_in":3599}

The access_token is then used in the Authorization header with the "bearer" prefix:
`Authorization: bearer *access_token*`
After the *expires_in* time (in seconds) has elapsed, the access_token needs to be refreshed. This is done as shown below using the refresh_token from the previous login response.

### Refresh Access Token

    POST https://labs-api.geocaching.com/Api/Accounts/RefreshAccessToken?consumerKey=*consumerKey* HTTP/1.1
    accept: application/json
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Content-Type: application/json
    Content-Length: ***
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    
    {"RefreshToken":"*refresh_token*"}

The response will be similar to the login response

### Get user account details

Returns username, user ID number, user public GUID, user find count and URL to user avatar.

    GET https://labs-api.geocaching.com/Api/Accounts/GetAccount HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

## Searching for Adventure Labs

Each Adventure Lab is identified by it's unique GUID, for example: `5556bf6f-55fb-4e1b-a952-3633044f5be0`

### Searching by coordinates

    GET https://labs-api.geocaching.com/Api/Adventures/SearchV3?origin.latitude=51.5&origin.longitude=0&radiusMeters=20000 HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

This will return a list of Adventure Lab caches within the search radius. Basic information is also provided such as title, location, description, owner and rating.

### Smart Links

These are special links to lab caches that are designed to be shared on Facebook, geocache pages and in QR codes. They look like this: `https://labs.geocaching.com/goto/*`
The Adventure Labs app can be set as the default app for links in this format.
The part of the link after the `/goto/` is called the "Smart Name" and is unique to that lab cache. Originally the lab cache owner was able to choose the Smart Name (if their desired name wasn't already taken) however it now appears to be that, for new caches, the Smart Name is a "GUID-format-like" string (note: not the same as the GUID of the Adventure Lab cache).

To convert the Smart Name to the GUID

    POST https://labs-api.geocaching.com/Api/Adventures/GetAdventureIdBySmartLink HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Content-Type: application/json
    Content-Length: ***
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip
    
    "*SmartName*"

Note that the Smart Name in the post data is enclosed in double quotes.
The response with be the GUID of the Adventure Lab cache.

## Get the cache details

Once we have the GUID of the Adventure Lab we are interested in, we can load the full details.

    GET https://labs-api.geocaching.com/Api/Adventures/*GUID_GOES_HERE* HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

The response to this is mostly self-explanatory but it does vary slightly if you are logged in. If logged in, you get information on which stage you have already completed and a bit of data named `FindCodeHashBase16` which is required to check answers to the question.

### Past logs

This request is used to get a list of logs that past finders have written for this Adventure Lab cache:

    GET https://labs-api.geocaching.com/Api/ActivityLogs/Search?adventureId=*GUID_GOES_HERE*&skip=0&take=20 HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

## Checking answers to stages

When the user enters the answer to a stage it is first checked locally (by comparing hashes) and if it is correct, it is then sent to the server. The server also checks that submitted answers are correct.

### Local (hash) check

The `FindCodeHashBase16` value returned in the cache details is a MD5 hash of the user's public GUID (from the Get user account details request) and the answer concatenated together. Spaces are removed and all uppercase letters are converted to lowercase. The algorithm is shown below.

    answer_nospace = answer.replace(" ", "");  // Remove spaces
    hash_input = publicGuid + answer_nospace;  // Concatenate strings
    hash_input = hash_input.toLowerCase();     // Convert to all lowercase
    hash_result = md5(hash_input);             // md5 hash is used
    if (hash_result == FindCodeHashBase16) {
        // Answer is correct
    }

If the hash does not match, the app then replaces any occurrences of `"1920"` in the answer with a single apostrophe `"'"` and checks the hash again. If this also fails, the reverse is tried, any apostrophes are replaced with the string `"1920"` and the hash calculated and checked again. If this fails as well, then the answer is deemed to be incorrect.

### Server check

After the answer has been confirmed correct by the hash check, it is then sent to server. This marks the stage as found for the user and increments their find count.

    POST https://labs-api.geocaching.com/Api/Adventures/EnterFindCode HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Content-Type: application/json
    Content-Length: ***
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

    {
    "AdventureId":"*lab_guid*",
    "GeocacheId":"*stage_guid*",
    "FindCode":"*answer*",
    "Location":{
        "Speed":0,
        "Heading":0,
        "Accuracy":5,
        "Altitude":0,
        "Longitude":**.*,
        "Latitude":**.*
        }
    }

The GUIDs are found in the cache details. The Location data comes from the user's GPS.

The response will be a 0 for correct answer and 1 for incorrect answer.

## Writing Logs and Ratings

### Status Check

Before any logs or ratings are submitted, a status check request is done to check that the user has found all the stages of the adventure lab.

    GET https://labs-api.geocaching.com/Api/AdventurePlayerStates/*GUID_GOES_HERE* HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

The response will have a field `IsComplete`. Logs and ratings can only be submitted if this field if true.

### Writing logs

To submit your own log to the adventure lab, use the following HTTP request. This will overwrite any exisiting log the user has for the given adventure lab (ie. you can't log the same adventure lab twice, if you try, it will only show the most recent one).

    PUT https://labs-api.geocaching.com/Api/ActivityLogs/*GUID_GOES_HERE* HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Content-Type: application/json
    Content-Length: ***
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

    {"LogText":"Your log text goes here"}

### Ratings

To rate an adventure lab (number of stars out of 5) use the following HTTP request. Like with logs, if you try rate the same adventure lab twice, it will overwrite your previous rating.

    PUT https://labs-api.geocaching.com/Api/Ratings/*GUID_GOES_HERE* HTTP/1.1
    accept: application/json
    authorization: bearer *access_token*
    user-agent: Adventures/1.2.15 (1822) (android/30)
    Content-Type: application/json
    Content-Length: ***
    Host: labs-api.geocaching.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

    {"Value":5}

## More info

This is the full list of the API functions\
[https://labs-api.geocaching.com/swagger/docs/v1](https://labs-api.geocaching.com/swagger/docs/v1)

This document is still a work in progress. If you have any further info on how the Adventure Labs app works, please do get in contact.
