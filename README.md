# Explainer: Login Status API

A [Work Item](https://privacycg.github.io/charter.html#work-items)
of the [Privacy Community Group](https://privacycg.github.io/).

## Editors:

- (suggestion) [Sam Goto](https://github.com/samuelgoto), Google Inc.
- (suggestion) [John Wilander](https://github.com/johnwilander), Apple Inc.
- (suggestion) [Ben Vandersloot](https://github.com/bvandersloot), Mozilla

## Participate
- https://github.com/privacycg/login-status-api/issues

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

## Introduction

This explainer proposes a Web Platform API called the **Login Status API** which websites can use to inform the browser of their user's login status, so that other Web APIs can operate with this additional signal.

On one hand, browsers don't have any built-in notion of whether the user is logged in or not.  Neither the existence of cookies nor frequent/recent user interaction can serve that purpose since  most users have cookies for and interact with plenty of websites they are not logged-in to. High level Web Platform APIs, such as form submission with username/password fields,  WebAuthn, WebOTP and FedCM, can (implicitly) record login, but don't record logout - so the browser doesn't know if the user is logged in or not.

On the other hand, there is an increasing number of Web Platform APIs (e.g. [FedCM](https://github.com/privacycg/is-logged-in/issues/53), [Storage Access API](https://github.com/privacycg/storage-access/issues/8)) and Browser features (e.g. visual cues in the url bar, extending long term disk and backup space) that could perform better under the assumption of whether the user is logged in or not.

This proposal aims at creating an API that allows websites to opt-into declaring their user's login status.

At its [baseline](#the-baseline-api), the API offers a way for websites to self-declare their user's client-side login status, which can be used in a series of planned Web Platform features.

A series of planned [extensions](#extensions) are enumerated that informed the design of the [baseline API](#the-baseline-api) and are expected to be actively introduced in the future.

## Proposal

### The Baseline API

At its baseline, the website's **self-declared** login status is represented as a three-state value **per origin** with the following possible values:

- `unknown`: the browser has never observed a login nor a logout
- `logged-in`: a user has logged-in to **an** account on the website
- `logged-out`: the user has logged-out of **all** accounts on the website

 By **default**, every origin has their login status set to `unknown`.

The login status represents the **client-side** state of the origin in the browser and can be out-of-date with the **server-side** state (e.g. a user can delete their account in another browser instance). Even then, due to cookie expiration, it is an imperfect representation of the client-side state, in that it may be outdated.

It should not be the user agent's responsibility to ensure that client-side login status is reflective of server-side or cookie state. However, this proposal wants to make it as easy as possible for developers following recommended patterns to keep these states synchronized.

#### Setting the Login Status

There are two mechanisms that a website can use to set the login status: a JS API and an HTTP header API.

##### JS API

To set the login status, a website can call the following JS APIs:

```javascript
navigator.login.setStatus("logged-in");
navigator.login.setStatus("logged-out");
```

That's enabled by the following web platform changes:

```javascript
partial interface Navigator {
  Login login;
};

enum LoginStatus {
  "logged-in",
  "logged-out",
};

partial interface Login {
  Promise<void> setStatus(LoginStatus status);
};
```

> A few alternatives considered:
> - navigator.login.status = "logged-in"; // needs to be async and doesn't throw 
> - navigator.loginStatus.setLoggedIn(); // a bit verbosed

##### HTTP API

In addition to the JS APIs, HTTP response headers are also introduced, to offer the equivalent functionality:

```
Set-Login: logged-in
Set-Login: logged-out
```

#### Using the Login Status

The login status is available to Web Platform APIs and Browser features outside of this proposal.

Every user of the login status (e.g. Web Standards or browser features integrating with it) **MUST** incorporate into their threat model that it is:

1. Self-declared: any website can and will lie to gain any advantage
2. Client-side: the state represents the website's client-side knowledge of the user's login status, which is just an approximation of the server-side's state (which is the ultimate source of truth) 

Therefore, the login status of a cross-origin domain must not be observable by a page itself. 

One potential for abuse is if websites don’t call the logout API when they should. This could allow them to maintain the privileges tied to login status even after the user logged out.

Features using the baseline Login Status need to assume that (1) and (2) are the case and design their security and privacy models under these conditions.

###### FedCM

As a concrete example of use of the Login Status bit, [FedCM](https://github.com/privacycg/is-logged-in/issues/53#issue-1664953653) needs a mechanism that allows Identity Providers to signal to the browser when the user is logged in.

```javascript
// Records that the user is logging-in to a FedCM-compliant Identity Provider.
navigator.login.setStatus("logged-in");
```

###  Extensions

There are a few planned extensions that we want the baseline to be forwards compatible with:

- The [Storage Access API](#storage-access-api)
- [Extending Site Data Storage](#extending-site-data-storage)
- Introducing [Status Indicators](#status-indicators)
- [Remember-me](#remember-me)
- Expiry due to user inactivity

#### Storage Access API

The [Storage Access API](https://github.com/privacycg/storage-access/issues/8#issue-560633211) could benefit from a login status signal by allowing developers to call the API with the option to only show user-facing prompts if the user is logged into their site, and reject otherwise. The Storage Access API must avoid directly exposing the login status to attackers that are measuring the time to rejection, by delaying the rejection by random response times that reflect average real user response times (which are usually in the range of a few seconds).

```javascript
// Records that the user is logging-in, which allows the Storage Access API to conditionally dismiss its
// UX.
navigator.login.setStatus("logged-in");

// In another cross-site top-level document, auto-reject rSA (with some delay to avoid timing attacks) unless the user is logged in
document.requestStorageAccess({
  requireLoggedIn: true
}).then(...);
```

#### Extending Site Data Storage

Browsers automatically clear site storage from time to time to manage the limited resources on a user's device. If the browser knew, with high confidence, that the user was logged in, then it could[extend](https://github.com/privacycg/is-logged-in/issues/15#issue-678611489) the site data storage.

Because this feature requires the browser to have a higher level of assurance, we'd want to introduce a mechanism that allows the login status indicator to represent beyond something that is self-declared.

In this variation, the website has access to a parameter that indicates to the browser that the website has gathered a `mediated` authentication signal from the browser, so that the browser can be more confident about the assertion.

```javascript
navigator.login.setStatus("logged-in", {
  browserMediated: true
});
```

What and how exactly the browser checks whether a mediated login occurred in the recent past is left to be done.

#### Status Indicators

Much like favicons, websites may be benefited from having a [login status indicator](https://github.com/privacycg/is-logged-in/issues/15#issue-678611489) in browser UI (e.g. the URL bar). The indicator could extend the login status API to gather the explicit signals that it needs from the website to display the indicator (e.g. a name and an avatar).

```javascript
// Records that the user is logging-in to a FedCM-compliant Identity Provider.
navigator.login.setStatus("logged-in", {
  indicator: {
    name: "John Doe",
    picture: "https://website.com/john-doe/profile.png",
  }
});
```

#### Remember me

When the browser automatically clears site data, it also purges signals that are useful for 2FA. It is plausible that the Login Status could be extended in the future to store independently a [Remember Me](https://github.com/privacycg/is-logged-in/issues/9) token which doesn't get cleared during the automatic site data purge, but can be retrieved later.

```javascript
navigator.login.setStatus("logged-in", {
  token: "0xCAFE"
});
```

## Challenges and Open Questions

## Considered alternatives

### An API-specific Signal

One obvious alternative that occurred to us was to build a signal that is specific to each API that is being designed. Specifically, [should FedCM use its own signal or reuse the Login Status API](https://github.com/privacycg/is-logged-in/issues/53)?

While not something that is entirely ruled out, it seemed to most of us that it would be worth trying to build a reusable signal across Web Platform features.

### Implicit Signals

Another trivial alternative is for Web Platform APIs to implicitly assume the user's login status based on other Web Platform APIs, namely username/password form submissions, WebAuthn, WebOTP and FedCM.

While that's an interesting and attractive venue of exploration, it seemed like it lacked a few key properties:

- first, logout isn't recorded by those APIs, so not very reliable
- second, the login isn't explicitly done and the consequences to other Web Platform APIs isn't opted-into. 
- third, username/passwords form submissions aren't a very reliable signal, because there are many ways in which a browser may be confused by the form submission (e.g. the password field isn't marked explicitly so but rather implemented in userland)

So, while this is also not an option that is entirely ruled out, a more explicit signal from developers seemed more appropriate.

### User Signal

Another alternative that we considered is an explicit user signal, likely in the form of a permission prompt. While that would address most of the abuse vectors, we believed that it would be too cumbersome and hard to explain to users (specially because the benefits are in the future).

## Stakeholder Feedback / Opposition

There is an overall directional intuition that something like this is a useful/reasonable addition to the Web Platform by the original proposers of this API and the APIs interested in consuming this signal (most immediately [FedCM and SAA](https://github.com/privacycg/is-logged-in/issues/53)).

This is currently in an extremely early draft that is intended to gather convergence within browser vendors.

## References & acknowledgements

[Your design will change and be informed by many people; acknowledge
them in an ongoing way! It helps build community and, as we only get by
through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in
alphabetical order.]

Former editor: [Melanie Richards](https://github.com/melanierichards), Microsoft


