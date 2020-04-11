# AWS Cognito Authentication
[<img src="http://img.shields.io/badge/swift-5.1-brightgreen.svg" alt="Swift 5.1" />](https://swift.org)
[<img src="https://github.com/adam-fowler/aws-cognito-authentication/workflows/Swift/badge.svg" />](https://github.com/adam-fowler/aws-cognito-authentication/actions?query=workflow%3ASwift)

This is the Vapor wrapper for [AWS Cognito Authentication Kit](https://github.com/adam-fowler/aws-cognito-authentication-kit). It provides application storage for configurations and authentication calls on request. Documentation on AWS Cognito Authentication Kit can be found [here](https://github.com/adam-fowler/aws-cognito-authentication-kit/blob/master/README.md)

# Using with Vapor
## Configuration
Store your AWSCognitoConfiguration on the Application object. In configure.swift add the following with your configuration details
```swift
let awsCognitoConfiguration = AWSCognitoConfiguration(
    userPoolId: String = "eu-west-1_userpoolid",
    clientId: String = "23432clientId234234",
    clientSecret: String = "1q9ln4m892j2cnsdapa0dalh9a3aakmpeugiaag8k3cacijlbkrp",
    cognitoIDP: CognitoIdentityProvider = CognitoIdentityProvider(region: .euwest1),
    region: Region = .euwest1
)
app.awsCognito.authenticatable = AWSCognitoAuthenticatable(configuration: awsCognitoConfiguration)
```
The CognitoIdentity configuration can be setup in a similar way.
```swift
let awsCognitoIdentityConfiguration = AWSCognitoIdentityConfiguration(
    identityPoolId: String = "eu-west-1_identitypoolid"
    identityProvider: String = "provider"
    cognitoIdentity: CognitoIdentity = CognitoIdentity(region: .euwest1)
)
let app.awsCognito.identifiable = AWSCognitoIdentifiable(configuration: awsCognitoIdentityConfiguration)
```
## Accessing functionality
Functions like `createUser`, `signUp`, `authenticate` with username and password and `responseToChallenge` are all accessed through `request.application.awsCognito.authenticatable`. Extend `AWSCognitoAuthenticateResponse` to conform to `Content` and the following login route will return the full response from `AWSCognitoAuthenticable.authenticate`.
```swift
    func login(_ req: Request) throws -> EventLoopFuture<AWSCognitoAuthenticateResponse> {
        let user = try req.content.decode(User.self)
        return req.application.awsCognito.authenticatable.authenticate(
            username: user.username, 
            password: user.password, 
            context: req, 
            on:req.eventLoop)
    }
```
If id, access or refresh tokens are provided in the 'Authorization' header as Bearer tokens the following functions in Request can be used to verify them `authenticate(idToken:)`, `authenticate(accessToken:)`, `refresh`. as in the following
```swift
func authenticateAccess(_ req: Request) throws -> Future<> {
    req.awsCognito.authenticateAccess().flatMap { _ in
        ...
    }
}
```

## Authenticators

Three authenticators are available. See the [Vapor docs](https://docs.vapor.codes/4.0/authentication) for more details on authentication in Vapor.`AWSCognitoBasicAuthenticator` will do username, password authentication and returns a `AWSCognitoAuthenticateResponse`. `AWSCognitoAccessAuthenticator` will do access token authentication and returns an `AWSCognitoAccessToken` which holds all the information that could be extracted from the access token. `AWSCognitoIdAuthenticator<Payload>` does id token authentication and extracts information from the id token into your own `Payload` type. The standard list of claims that can be found in an id token are detailed in the [OpenID spec] (https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims). Your `Payload` type needs to decode using these tags, the username tag "cognito:username" and any custom tags you may have setup for the user pool. Below is an example of using the id token authenticator.

First create a User type to store your id token payload in.
```swift
struct User: Content & Authenticatable {
    let username: String
    let email: String
    
    private enum CodingKeys: String, CodingKey {
        case username = "cognito:username"
        case email = "email"
    }
}
```
Add a route using the authenticator. The `AWSCognitoIdAuthenticator` authenticates the request, the `guardMiddleware` ensures the user if authenticated. The actual function accesses the `User` type via `req.auth.require`.
```swift
app.grouped(AWSCognitoIdAuthenticator<User>())
    .grouped(User.guardMiddleware())
    .get("user") { (req) throws -> EventLoopFuture<User> in
    let user = try req.auth.require(User.self)
    return req.eventLoop.next().makeSucceededFuture(user)
}
```

