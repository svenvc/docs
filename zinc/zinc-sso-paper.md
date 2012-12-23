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
      url: 'http://mc.stfx.eu/Neo';
      package: 'Neo-JSON-Core';
      load.

    Gofer it
      url: 'http://mc.stfx.eu/Zodiac';
      package: 'Zodiac-Core';
      load.

    Gofer it
      url: 'http://mc.stfx.eu/ZincHTTPComponents';
      package: 'Zinc-FileSystem';
      package: 'Zinc-Character-Encoding';
      package: 'Zinc-Resource-Meta';
      package: 'Zinc-HTTP';
      package: 'Zinc-SSO-OAuth2-Core';
      load.

Since Zodiac is needed, you will need the SSL Plugin for your VM.

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
