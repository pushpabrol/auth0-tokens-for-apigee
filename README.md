# Using Auth0 tokens to access Apigee protected resources


Apigee with its rich ecosystem of policies can enable flows that allow verification of external access tokens for APIs protected under Apigee. The flow itself is pretty simple and straight forward and is explained below:

## Flow
1.	An Application using Auth0 as the Authentication system allows a user to Authenticate with Auth0 and issue an access_token
2.	Application can then use this access_token to access an API proxied via Edge
3.	Application makes a call to the API and attaches the access_token as an Authorization header to the proxy endpoint
4.	Apigee policies,
    * extract the access_token
    * Perform a callout to verify the access_token
    * Extract claims from the access_token once verified
    * Attach claims as headers if the target endpoint requires that data
5.	Allow the call to go thru to the target API

This is the source code for the Proxy, policies and callouts required for this setup

The Java Callout policy is borrowed from a great set of code libraries made available by Dino Chiesa at https://github.com/apigee/iloveapis2015-jwt-jwe-jws/tree/master/jwt_signed

## Setup to test this:
1.	Signup at auth0.com for an account/tenant
2.	Create a single page client under Auth0’s management console -> https://manage.auth0.com/#/clients
3.	Set the allowed callback url as https://jwt.io
4.	Define an API/Resource Server under Auth0 -> APIs. This step allows Auth0 to recognize the external resource as an audience and will make sure the access_tokens are issued for this audience. When a user authenticates with Auth0 via the application and specifies this audience as part of the process of authentication Auth0 makes sure that the access_token is scoped to this audience so that the application can access the resource on behalf of the user with the scopes that the user consented to while authenticating

5.	Make note of the following information:
      * Auth0 domain   <tenant>.auth0.com
      * Client ID of the application created above  <client_id>
      * Allowed callback url    https://jwt.io
      * Audience of the API defined above
  I will not go into explaining how to create a connection and users within Auth0 as it is subject to the readers setup and they can use any type of connection and user source such as Database, LDAP, federation
  With the above setup you have enough information to login as a user and get an access_token and use that to call the Apigee proxy.

6.	To login the user you can start an authentication transaction with Auth0 constructing a URL as shown below:

```
https://<tenant>.auth0.com/authorize?client_id=<client_id>&response_type=token&redirect_uri=https://jwt.io&scope=openid&audience=<audience>&nonce=<random number>&state=<random number>
```
On login you will be redirected to `https:jwt.io` with an access_token. You can use this access_token to call the Apigee proxy
```
curl -X GET -H "Authorization: Bearer <access_token>" -H "Cache-Control: no-cache" "http://<your>-test.apigee.net/auth0-hello"
```
Before you do that though you have to have the correct setup within the Proxy to be able to validate the token presented in the Authorization header. Lets talk a little bit about that below

## API Proxy Configuration
The API proxy needs to know how to validate/verify the access_token that is attached to the Authorization header. The Java Callout Policy in the attached proxy has configuration to allow for this. ( Thanks Dino Chiesa).
The information you will need for the Java Callout are in bold below
```xml
<JavaCallout name="JWT-Parse-OpenIDConnect">
    <Properties>
        <Property name="algorithm">RS256</Property>
        <Property name="jwt">{clientrequest.oauthtoken}</Property>
        <!-- public-key used only for algorithm = RS256>. This is Auth0 tenant’s public key that you can download from https://<tenant>.auth0.com/pem or https://<tenant>.auth0.com/cer and convert format to get the public key if requred --> 
        <Property name="public-key">
    -----BEGIN PUBLIC KEY-----
    …
    -----END PUBLIC KEY-----    
        </Property>
        <Property name="iss">https://<tenant>.auth0.com</Property>
        <Property name="aud">audience_of_api_defined_in_auth0</Property>
    </Properties>

```

Once you have this configuration done based on your setup you can deploy your proxy and configure your target endpoint and test out the protection of your APIs. 

Note: For testing you can just setup a target API that extracts the headers and shows that the required user claims are available within the header 





 





