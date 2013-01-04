# Zinc SSO

*Jan van de Sandt & Sven Van Caekenberghe*

*December 2012*

*(This is a draft)*

Zinc SSO is a library/framework to enable web applications to delegate
authentication to often well-known external service providers.
Your users will be happy because they can reuse their existing account,
you will be happy because you have less to implement and worry about.
Zinc SSO is an addon to Zinc HTTP Components.

*(Please note that Zinc-SSO is currently in development. This is alpha code.)*

# OAuth

See also [http://en.wikipedia.org/wiki/Oauth](http://en.wikipedia.org/wiki/Oauth).

## Installation

Zinc-SSO depends on Zinc-HTTP-Components and Zodiac and uses NeoJSON as well. 
This is a Gofer load script for Pharo 2.0:

    Gofer it
      repository: 'http://mc.stfx.eu/Neo';
      package: 'NeoJSON';
      load.

    Gofer it
      repository: 'http://mc.stfx.eu/Zodiac';
      package: 'Zodiac-Core';
      load.

    Gofer it
      repository: 'http://mc.stfx.eu/ZincHTTPComponents';
      package: 'Zinc-FileSystem';
      package: 'Zinc-Character-Encoding';
      package: 'Zinc-Resource-Meta';
      package: 'Zinc-HTTP';
      package: 'Zinc-SSO-OAuth2-Core';
      load.

Since Zodiac is needed, you will need the SSL Plugin for your VM.

## Zinc-SSO OAuth Support

With OAuth an application can ask a user permission to access the user's data on a third party system. For example a Seaside application can ask a user permission to read its data on Twitter. If an application only asks for permission to view the user profile data than you use OAuth just for single sign-on and user identification functionality. It is also possible to ask for additional authorizations, for example for permission to create new Tweets for a user. In this case yoy can do a lot more than just single sign-on.

Currently there are two versions of OAuth in active use. Version 1.0a and version 2.0, see wikipedia for a list of OAuth service providers and the versions they support. The two versions offer the same kind of functionality buy are technically very different. Version 2.0 is really simple to use and just relies on https for security. Version 1.0a is more complex and uses a digital signature to verify the integrity of a http request. 

# OAuth 1.0a Support

A well known service provider that uses version 1.0a is Twitter. We will use Twitter as an example here but things will work the same for other service providers that support this version.

Twitter documentation: https://dev.twitter.com/docs/auth/implementing-sign-twitter

The first thing that you need to do is to register your application with Twitter. You need to provide a name and a description of your application and you must specify what kind of access you need. In the case of Twitter you can choose between readonly access or write access. For single sign-on readonly access is sufficient. Twitter will generate a consumer key and a consumer secret string. We need these two values in the requests we send to twitter. The key will be one of the parameters, the secret is used to create the digital signature for the requests and must never be used as a request parameter.

In Zinc-SSO the consumer key and secret are stored in a ZnOAuth1ConsumerData instance. This object also holds the urls for the required API calls. The url's for Twitter are hardcoded in a class method:

consumerData := ZnOAuth1ConsumerData newForTwitterAuthentication
    consumer: 'YgnhZtjas8ccdVdyZ1QGBA';
	consumerSecret: '--the-secret--';
	yourself.

Now you can put a “Signin with Twitter” button or link in your web application. In the code that gets called you should perform the following steps:

Step 1: Get a request token – Before we can redirect the user to the Twitter signin page we need to get a request token. An important parameter for the API call to get this token is the callbackUrl. This is the url that the user will be redirected back to after the signin. 

	service := ZnOAuth1Service new
		providerAccount: consumerData ;

		yourself.
	requestToken := service getRequestTokenFor: 'http://my-domain/sso-callback'.

The #getRequestTokenFor: will create a signed OAuth request which includes your consumer key and will send the request to Twitter. If all goes well Twitter will respond with a request token. The token consists of a value and a secret. We need to store this token in our session. Later when the user is redirected back to our app we need this token.

Step 2: Redirect the user to the signin page of the service provider

	redirectUrl := service loginUrlFor: requestToken

ZnOAuth1Service>>#loginUrl: will answer a ZnUrl object to which we should redirect the user.

Step 3: Convert the request token into an access token

In our code that gets called when the user is redirected back to us we need to make a second API call to convert the request token into an access token. If the user signed in successfully and allowed our application access the request should contain two parameters: oauth_token and oauth_verifier. The oauth_token should be equal to the value of the requestToken and the oauth_verifier is needed to get the access token: 

handleOAuth1Callback: request

	requestToken := self session at: 'oauth-req-token'
 
		ifAbsent:  [ self error: 'Invalid callback – no request token' ].

	
	oauthToken := request uri queryAt: 'oauth_token'.

	oauthVerifier := request uri queryAt: 'oauth_verifier'.


	(oauthToken isNil or: [ oauthVerifier isNil])
		ifTrue: [ self error: 'Invalid request' ].


	oauthToken = requestToken value

		ifFalse: [ self error: 'Invalid request' ].


	accessToken := self twitterOAuth1Service getAccessToken: requestToken verifier: oauthVerifier.

Now we have an accessToken the requestToken is no longer needed. We can use the accessToken to retrieve information about the user. We can do that immediately and forget about the accessToken or we can store this accessToken somewhere and do this at a later time. The time that an accessToken stays valid is service provider specific.

The way to get information about the user is also service provider specific. Twitter recommends that you use the verify_credentials API call to get basic user information. The class ZnOAuth1TwitterUserAccess contains a method that wraps this Twitter API call.

	userData := ZnOAuth1TwitterUserAccess new

		oauth1Service: self twitterOAuth1Service ;
		accessToken: accessToken ;
		accountVerifyCredentials.

The #accountVerifyCredentials method answers a Dictionary with information about the user. See the Twitter documentation for the exact contents.


## OAuth Providers

Currently, OAuth version 2 is implemented. 
There are demos available for Google, Microsoft and Facebook accounts.

## Demos

Setting up OAuth takes some work, which why two online demos are available:

- [http://sso.stfx.eu] (http://sso.stfx.eu)
- [http://sso.doit.st] (http://sso.doit.st)

The first demo is running on top of Zinc only, 
the second demo uses Seaside for its user interface.
Both use the same Zinc-SSO-OAuth-Core code.

## Getting Started

In order to get started with your own setup, 
you will first have to register your
application with the providers that you want to use.
During that registration you have to specify the domain of your web application.

After registration you will get two crucial elements of information:

- a client id or key
- a secret

Combined with your callback URL/URI these are used as parameters:

    ZnOAuth2ConsumerData 
      key: '531369976180189'
      secret: '16ag98834ec8925787a75a8141a98810'
      redirectUrl: 'http://my-app.my-domain.com/sso-google-callback'

The ZnOAuth2ConsumerData is then used to parameterize a ZnOAuth2Session.
The ZnOAuth2Session does two things:

- generate the URL to request authentication from the service provider
- handle the authentication callback from the service provider to your web app

If all goes well, the callback will contain enough information, called user data,
for you to accept that the user holds that account. 
You can trust that the service provider did its normal work to authenticate the login.

***

Lots more to understand and explain ;-)
