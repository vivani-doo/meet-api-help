# Meet API samples

This repository contains samples of how to integrate your application and/or service with Rest API.

So far we have sample integation code for:

- [CSharp](https://github.com/meet-rs-api/api-docs-samples/tree/master/csharp/MeetApiSample)
- [NodeJS](https://github.com/meet-rs-api/api-docs-samples/tree/master/nodejs)

We plan to add more languages and use cases as the time comes - contributions are also very welcome :)

## General overview of the Meet API integration

Meet API is hosted on the *https://api.meet.rs* address.

In general, completing a scenario on Meet API requires two steps:

- Obtaining a valid API bearer token
- Acting while using that token

### Table of Contents

Here are the scenarios which we will cover here:

- [Authentication](#authentication)
- [Creating a Meet](#creating-a-meet)
- [Meet participants](#meet-participants)
- [Meet configuration](#meet-configuration)
- Meet scheduling
- Working with Meet projects (aka "interview job positions")
- Manage tenants
- Webhooks
- [Feedbacks](#feedbacks)
- [Usage](#usage)
- Billing

## Authentication

Meet API authentication is a standard client credentials OAUth scenario where you exchange your api key and secret for a JWT bearer token.

You can obtain your API key and secret in the Meet Pro app (<https://meet.rs/pro)> under Settings >> Company profile.

![alt text](https://cdn.meet.rs/assets/docs/settings_devtools.png "DevTools")

Using the language/technology of your choice, you need to simply make this request.

pseudo
POST https://api.meet.rs/v1/token
Content-Type: application/json

{
    grant_type: "client_credentials",
    client_key: "YOUR API KEY HERE",
    client_secret: "YOUR API SECRET HERE",
}


Note: If your api key and secret are not valid, you will get a *401 - Unauthorized* response status code.

In case api key and secret are valid, you will get a response containing the JWT bearer access token and a timestamp when the issued token expires.

pseudo

    // JWT bearer token
    access_token: string

    // Unix epoch based date/time when become invalid
    expires_at: number


Note: You can examine the content of the JWT access token if you just paste it in https://jwt.io 

Ok, so you got yourself a brand new Meet API token - groovy! 

Let us see it in action!

## Creating a Meet

### Create a Quick Meet

The simplest way to create a quick Meet (60 min, default config, anyone with a link can join at any time) is to make a simple POST request with an empty body payload.

pseudo
POST https://api.meet.rs/v1/meetings
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE

{} <-- empty body 



In response to that request, you will get a new Meet definition which, among other things, will contain the URL for the newly minted Meet.

pseudo

    ...

    // A full URL to a new Meet
    joinUrl: string

    ...



You can now give this URL to your own users so they have their frictionless Meet.

### Customizing essential Meet attributes (title, description, etc.)

Let say you would like to customize the Meet title or description, and the way to do that is to pass it in the Meet creation request body simply.

pseudo
POST https://api.meet.rs/v1/meetings
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE

{ 
    "title": "Meeting with John Snow",
    "description": "Annual gathering of the Black Crow society."
}


In response to that request, you will get a new Meet definition which will have a part.

pseudo

    ...

  "code": "5LM7fdZ0",
  "joinUrl": "https://meet.rs/5LM7fdZ0",
  "description": "Annual gathering of the Black Crow society",
  "title": "Meeting with John Snow",

    ...



## Meet participants

If you don't provide the list of participants in the Meet creation, anyone with a link can join the Meet, which in many cases is not a desirable thing.
In addition to the security aspect, many Meet scenarios require differentiation on what participants can do in the Meet.

For example, in an interview scenario, typically, there are roles of Interviewer and Candidate where Interviewer is driving the Meet and has access to the task library, etc., while the Candidate is seeing Candidate specific experience.

Meet API supports defining the participants who can access the Meet and their roles in a straightforward but powerful way for this type of scenario.

Every participant can authenticate himself/herself in one of the few supported ways:
  
- passcode  
- OAuth
- SMS (soon)
- ? (StackOverflow, GitHub, Amazon, Twitter) <- if requested

### Passcode authorized participants

Let start with the example mentioned above where we would like a Meet for two participants where Jack Sparrow is an interviewer, and John Smith is the CandidateCandidate.

pseudo
POST https://api.meet.rs/v1/meetings
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE

{
  participants: [
    {
      user : {
        firstName: "Jack",
        lastName: "Sparrow",
        email: "jack@sparrow.com"
      },
      role: "PowerUser",
      accessRules: [
        {
          provider: "Passcode",
          condition: "12345"
        }
      ]
    },
    {
      user : {
        firstName: "John",
        lastName: "Smith",
        email: "john@smith.com"
      },
      role: "User",
      accessRules: [
        {
          provider: "Passcode",
          condition: "112233"
        }
      ]
    }
  ]
}



As you can tell from this sample in the interview, use case roles are mapped like:

- role: Admin       -> recruiter/hr/organizer
- role: PowerUser   -> interviewer
- role: User        -> candidate

Also, we will use the email address provided here to send meeting participants emails containing information on how and when to Meet and a reminder email one hour before Meet starts.

### OAuth authorized participants

In this example, to make things a bit more interesting, let's have a Meet of more than two people:

- John Smith candidate who is still going to authenticate using passcode

- Jack Sparrow interviewer, who we expect to authenticate with his G Suite email jack@sparrow.com using Google authentication
- Zumi Zami interviewer, who we expect to login using her Office 365 login with her email office365@zami.com

Here is how would Meet creation request look like in this scenario:

pseudo

{
  participants: [
    {
      user : {
        firstName: "John",
        lastName: "Smith",
        email: "john@smith.com"
      },
      role: "User",
      accessRules: [
        {
          provider: "Passcode",
          condition: "112233"
        }
      ]
    },
    {
      user : {
        firstName: "Jack",
        lastName: "Sparrow",
        email: "jack@sparrow.com"
      },
      role: "PowerUser",
      accessRules: [
        {
          provider: "Google"
        }
      ]
    },
    {
      user : {
        firstName: "Zumi",
        lastName: "Zumi",
        email: "zumi@zumi.com"
      },
      role: "PowerUser",
      accessRules: [
        {
          provider: "Microsoft",
          condition: "office365@zumi.com"
        }
      ]
    }
  ]
}

  

NB: In the case of the Zumi Zumi participant, our access rule contains both provider and condition to be used, while in the case of Jack Sparrow, there is the only provider. The reason for this is that, in the case of condition email is the same as user email, the condition can be omitted.

How would this Meet work when defined like this?

Anyone opening the meeting URL will see this on their login page.

![alt text](https://cdn.meet.rs/assets/docs/multi_auth_provider.png "Authentication with multiple providers")

John will enter the passcode he will receive in his email and join the Meet as CandidateCandidate

Jack will click on "Sign in with Google" and use his work G Suite account to join the Meet.
Zumi will click on "Sign in with Microsoft" and log in using her Office 365 work account.

Two advantages for this model of authentication:

- Jack and Zumi don't need to remember any passcodes for any of the meetings - they just log in, and once they log in next Meet will, thanks to Single Sign-On, allow them to join the Meet without any login.

Companies where Jack and Zumi are working control their identities, and once Jack and Zumi leave the company; their login access is revoked, etc.

There are four oAuth providers supported by Meet in the initial release:

- Google,
- Microsoft,
- LinkedIn and
- Facebook

(We will extend the list based on the requests of our partners - any OAuth 2.0 provider can be used)

Finally, the participant auth model is not constrained to a single provider but instead can be a combination of multiple providers.

Take a look at how Jack Sparrow admin will be able to access his Meet.

pseudo

    {
      user : {
        firstName: "Jack",
        lastName: "Sparrow",
        email: "jack@sparrow.com"
      },
      role: "Admin",
      accessRules: [
        {
          provider: "Google",
          condition: "jack.sparrow@gmail.com"
        },
        {
          provider: "facebook",
          condition: "jacks.sarrow@gmail.com"
        }
      ]
    },



He can log in with any identity he has with Google, Microsoft, Linkedin, Facebook (as long he used their frank@moody.com email) or simply enter a password.

![alt text](https://cdn.meet.rs/assets/docs/all_providers.png "Authentication with all providers")

### Guest participants

#### Anonymous guest

There is some use case where we want to have an ability for anyone with a link to join the Meet where he will have some lower-level rights compared to other participants.

For example, let's take an educational use case where a Brave Startup Founder (BSF) is doing a webcast every Monday at 3 pm. He would log in using his passcode or his oauth, and everyone else will just join without any authentication.
He will be the only one who can share the screen, which can mute/kick attendees from the Meet etc.

Meet API makes this type of scenario straightforward (note definition of the second participant)

pseudo

{
  participants: [
    {
      user : {
        firstName: "Brave",
        lastName: "Founder",
        email: "brave@founder.com"
      },
      role: "PowerUser",
      accessRules: [
        {
          provider: "Microsoft",
          condition: "brave@founder.com"
        }
      ]
    },
    {
      role: "Guest",
    }
  ]
}



NB: The second participant has only a role without any access rule definition, which is fine as API will create for such entry implicitly.

pseudo
    accessRole: {
      provider: "Guest"
    }


This is how the logging screen for this Meet will look like

![alt text](https://cdn.meet.rs/assets/docs/guest_anon_login.png "Anonymous guest authentication")

#### Password protected anonymous guest

Sometimes Meet needs to be just a bit more constrained so anyone who knows a passcode (e.g., shared on Twitter) can access the Meet as a guest regardless of their identity  as a "Guest."

pseudo

{
  participants: [
    {
      user : {
        firstName: "Brave",
        lastName: "Founder",
        email: "brave@founder.com"
      },
      role: "PowerUser",
      accessRules: [
        {
          provider: "Microsoft",
          condition: "brave@founder.com"
        }
      ]
    },
    {
      role: "Guest",
      accessRules: [
        {
          provider: "Passcode",
          condition: "1234567"
        },
      ]
    }
  ]
}



NB: Only users who know the given passcode can enter the Meet as Guest.


#### Guest with identity

Sometimes you need a Meet where only a specific participant will be allowed to join as a Guest, and luckily that too is simple.

pseudo
{
  participants: [
    {
      user : {
        firstName: "Brave",
        lastName: "Founder",
        email: "brave@founder.com"
      },
      role: "PowerUser",
      accessRules: [
        {
          provider: "Microsoft",
          condition: "brave@founder.com"
        }
      ]
    },
    {
      user : {
        firstName: "John",
        lastName: "Smith",
        email: "john@smith.com"
      },
      role: "Guest",
      accessRules: [
        {
          provider: "Microsoft",
          condition: "john@smith.com"
        }
      ]
    }
  ]
}



NB: John Smith will have to authenticate with Microsoft in order to be allowed to join the Meet but only in a role of a Guest.

## Meet configuration

Every created Meet can be made using some of the predefined configuration templates to get an optimal Meet experience by selecting which addons you will need and precisely configuring them.

For example, if a Meet needs only a shared code editor and whiteboard, but the call will be done using Zoom, the ideal Meet experience will NOT include the whiteboard and calling parts.

![alt text](https://cdn.meet.rs/assets/docs/configurations-pro-list.png "Configurations PRO")

You can then select any of those configurations during the Meet creation in the PRO application.

![alt text](https://cdn.meet.rs/assets/docs/configuration-pro-create.png "Creating Meet with custom config")

As usual, everything you can do in the PRO app you can create using Meet API.

Here is how you can fetch the available configurations.

json
GET https://api.meet.rs/v1/addonConfigurations
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE



The addon configuration response contains a few properties:

- name and description - localized values defining the PRO app UI behaviors
partition key - ID of your tenant 
- isDefault - boolean defining if config should be used by default
- type - enum defining if the config is or Meet app (1) or for Teams addon (3)
- items - an array of configurations of every addon included in the addon configuration with key properties:
  - identifier - addon ID - 'monaco' (editor), 'rnto' - whiteboard, 'orpheus' - chat, roster - participant roster, 'twilio' - Twilio calling addon
  - configuration - JSON serialized list of specific addon configuration properties defining how addon should work.

Obviously, besides GET-ing a list of addon configurations, you can POST/PUT/DELETE them too. 
(As addon configurations are created very rarely (if ever), we recommend using PRO app UI for managing configurations and just use an ID value of the desired configuration in Meet creation requests.)

To create a Meet with a non-default addon configuration, you just need to pass it as a request param in the Meet creation call.

json
POST https://api.meet.rs/v1/meetings
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE

Body
---------------------------------------------
{
    addonConfigurationId: 'GUID_ID_CONFIG_TO_BE_USED'
}



That's it - your Meet will be created using the config you've specified. In case you omit the parameter, the config marked as 'default' will be used.

## Feedbacks

Every participant of the Meet can click a feedback button at any time, and on the NPS form, rate his/her Meet experience and provide feedback. We are also asking each of the participants at the end of the Meet if they are willing to provide feedback.

This feedback is about the quality of the Meet app, so it is an important QoS parameter, and sometimes it can be about the interviewer and interview itself, which can be interesting feedback to the Meet organizer.

To be transparent with our users, we have decided to share this NPS feedback data, so our users can see the NPS feedback for their tenant in the [PRO application](https://meet.rs/pro).

As with everything else available in the PRO app, you can also just use the rest API to fetch the same data as the PRO app does.

The simplest way to get the data for the tenant whose API key and secret are using and for the current month is a simple get with no params.

json
GET https://api.meet.rs/v1/feedbacks
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE


If you would like to get the feedback for different date ranges, you can add to query params from and to parameters with range dates given in ISO format.

pseudo
GET https://api.meet.rs/v1/feedbacks?from=2020-07-01T00:00:00.000Z&to=2020-07-31T23:59:59.000Z
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE


Both calls will return all the data of your tenant and every subtenant your tenant might have.

If you would like to get only the data of a single-tenant, you need to pass tenant code/id parameter, which you can see in Company settings in PRO app, or if you GET fetch from API the list of tenants. 

pseudo
GET https://api.meet.rs/v1/feedbacks/A1B2C3D4E5F6
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE


This will return the feedback data only of a tenant with a given code.

Of course, you can also define optional from and even when you have tenant code defined.

pseudo
GET https://api.meet.rs/v1/feedbacks/A1B2C3D4E5F6?from=2020-07-01T00:00:00.000Z&to=2020-07-31T23:59:59.000Z
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE


Regardless of how you make the web service call, you would always get the same feedback items looking like this.

json
[{
    "rating": 5,
    "text": "test feedback 1",
    "userId": "c475f065-9aa7-457e-e605-08d82d562450",
    "resourceId": "le3YHkUb",
    "resourceType": 1,
    "tenantId": "84587e95-77a0-453d-dd3b-08d7611907eb",
    "timestamp": "2020-07-22T11:57:40.3817518Z",
    "appVersion": "2.7.21.1050",
    "browser": "Anheim",
    "operatingSystem": "Windows",
    "screenHeight": "1200",
    "screenWidth": "1920",
    "isTouchEnabled": false,
    "id": "cf08a200-3b79-445d-a849-08d82d5c6c28"
}]



## Usage
In order to have precise insights on the Meets happening in your tenant, which will have an impact on your monthly payments, you can control your usage data providing easy to understand overview of your Meet app utilization.

You can see the usage data in the [PRO application](https://meet.rs/pro) visualized in a table or as everything else available in the PRO app. Of course, you can also just use the rest API to fetch the same data as the PRO app does.

The simplest way to get the data for the current tenant and for the current month is a simple make a GET request with no params.

json
GET https://api.meet.rs/v1/usage
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE


If you would like to get the usage information for a different date range than this month, you can add to query params from and to parameters with range dates given in ISO format.

pseudo
GET https://api.meet.rs/v1/usage?from=2020-07-01T00:00:00.000Z&to=2020-07-31T23:59:59.000Z
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE


Both calls will return all the data of your tenant and every subtenant your tenant might have.

If you would like to get only the data of a single-tenant, you need to pass tenant code/id parameter, which you can see in Company settings in PRO app, or if you GET fetch from API the list of tenants. 

pseudo
GET https://api.meet.rs/v1/usage/A1B2C3D4E5F6
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE


This will return the feedback data only of a tenant with a given code.

Of course, you can also define optional from and even when you have tenant code defined.

pseudo
GET https://api.meet.rs/v1/usage/A1B2C3D4E5F6?from=2020-07-01T00:00:00.000Z&to=2020-07-31T23:59:59.000Z
Content-Type: application/json

Headers
---------------------------------------------
Authorization: bearer ACCESS_TOKEN_VALUE_HERE


Regardless of how do you make the web service call, you would always get the array of the same feedback items looking like this.

json
[{
    "code": "le3YHkUb",
    "title": "Quick meet 07/22 - 265",
    "description": "Guest ",
    "participants": 2,
    "createdAt": "2020-07-22T11:57:00.408594Z",
    "startedAt": "2020-07-22T11:57:11.2909288Z",
    "endedAt": "2020-07-22T11:57:28.0734762Z",
    "totalDurationInSeconds": 17,
    "billableDurationInSeconds": 10,
    "isFree": true,
    "isSubscriptionCovered": true,
    "subscriptionId": "d66b178d-c801-481f-d5a9-08d82d522f18",
    "id": "a5c9d5b6-87f6-4743-a06c-08d82d5e2c69"
}]
```

A few clarifications on the response data:

- *Total duration* is a duration in seconds when there is at least one participant on the ongoing Meet
- *Billable duration* is a duration in seconds of the Meet when there was at least one participant on the ongoing Meet
- *Is Free* if a billable duration is less than 20 minutes, Meet is free.
- *Is Subscription Covered* most of the subscriptions have a certain number of Meets per month included in the price of the subscription, so if this field has a true value, the Meet cost is covered with a subscription. If the value is false, the Meet will be charged to you at the end of the month in the invoice based on the price defined for your subscription for additional Meet units.
