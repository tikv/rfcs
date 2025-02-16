# TiKV Auth implementation

This RFC outlines the design space for TiKV auth and a potential solution: https://github.com/tikv/rfcs/pull/40

Here we will discuss in much greater detail implementing the solution proposed in that RFC.

## Summary

* PD stores a list of applications and their authentication. This database can be filled at deployment time and modified by the root user.
* An application such as TiDB can get a token from PD.
* The token can be sent to TiKV. TiKV checks to see if the token is properly signed by PD by using the public key of PD.
* The token can also be used to access admin APIs of PD and TiKV, this requires admin permission.
* Programs such as tools that need to access TiKV directly can get a delegation token from TiDB to securely access TiKV.

## Requirements

Clusters must have TLS enabled to prevent credentials from being stolen in transit.
Certificate authentication for TiDB is mentioned, this is available in version 3.0.8.

## PD Auth

Applications are registered and identified by their certificate.
Simply setting up TLS properly will suffice to setup access for TiDB and TiKV: we only need to register the certificates for the users during the deployment.
There is also a root application that users may setup with password access.

* PD has a database of applications allowed access
  * Application name (string)
  * Admin (boolean): is the application allowed access to administrative apis
  * Authentication
    * Method (certificate or password)
    * Value (certificate identity or hashed password)
* Those using TiDB should add “tidb” as an application with admin access (see the admin access section)
* There are two pre-seeded applications
  * Tikv (for TiKV): securing the cluster requires this auth to be seeded
    * The TiKV application has implied special privileges needed to operate TiKV
  * Root: securing the cluster requires this auth to be seeded
    * The root application has the Admin flag set to true by default
      * The root application may access via password
* A configuration option and environment variable `AUTH_REQUIRED` will require that all accessors of PD present authentication.
  * Auth is always required for the tikv and root applications once their auth is set

Applications can be registered at deployment time or by the root user via API
* Deployment specification is by configuration parameters that can be specified in the config file or by environment variables, below we use environment variables
  * The auth method is specified by `AUTH_METHOD_$APPLICATION` (defaults to certificate)
  * Admin access is specified by `AUTH_ADMIN` (defaults to false)
  * The secret itself is specified by one of
    * an environment variable that contains the user’s secret `AUTH_$APPLICATION` (this cannot be set in the config file)
    * The contents of a file specified by `AUTH_FILE_$APPLICATION`
* Add an API to PD POST /auth/application_register
  * As per above, has fields: name, admin, auth_method, auth_value
  * Only the root application can access this API
* Add an API to PD PUT /auth/application/{application}
  * This API is used to change the registration of an existing application
  * Application in the url is the application name
  * Only the root application can access this API

With the above we have achieved useful authentication to PD.
However, we also need access for tooling.
At this stage, administrators must setup cert access for their tools just as TiKV and TiDB will use their certs for access.


## PD Token for TiKV

Now we need auth to TiKV.
We do this by using a token that PD generates.

* Add an API to PD: POST /auth/token
  * This API returns a cryptographically signed, tamper-proof token
    * The signing can use PD’s public/private key pair already known from TLS. TLS cert re-loading in TiKV will require TiKV to temporarily (for the expiration window, see the Refresh section), maintain the old cert.
* This token must be included in requests to TiKV (from TiDB)
  * Add a middleware to TiKV to accept these tokens
  * If TiKV can verify the signature (as per above this is public key cryptography), the request is valid, if not it is denied
* A configuration option and environment variable AUTH_REQUIRED will require that all accessors of TiKV present authentication

Now we have auth for both PD and TiKV.
Still, administrators must setup cert access for their tools to PD.


## Delegation and TiDB

A tool such as BR wants to login to TiDB and then directly access TiKV.
We make this more secure by having TiDB give a token to the tool for its usage with TiKV.

* Add an API to TiDB POST /auth/login/delegate
  * The requestor must include their login information
    * Username
    * Password (if not using certificate auth)
  * This will go through the MySQL login process
  * TiDB creates a token by making a request to PD for a delegation token
  * PD has an API /auth/delegate
    * This is the same as /auth/token but also allows TiDB to provide claims that it finds meaningful.
    * TiDB includes in the request any additional restrictions it wants added to the token: these will be included in the cryptographic signing in addition to any restrictions that TiDB itself has (currently none, but when multiple applications are supported it will have restrictions)
  * TiDB returns this delegation token to the requestor
* This token can be used for requests to PD and TiKV just the same as the original TiDB token (but the additional restrictions will be enforced)
* PD must also accept these tokens in addition to its existing enforcement of cert-based authentication
  * Add a middleware to PD to accept tokens for authentication
* TiDB itself can also accept these tokens for its (admin) HTTP APIs

Now a tool can login to TiDB, get a token, and use that token for its access to TiKV.

It can also be useful for TiDB clients to exchange their password for refreshable tokens. For example the tidb-dashboard project would like to ask the user for their password and then make queries over time without asking the user for their password again for some time.

## Supporting Access Restrictions in TiKV

In the above we create an API that allows for the addition of restrictions in delegated tokens, but we need TiKV and PD to verify them.
I suggest we start with just distinguishing the following useful restrictions:
  * Read-only access: this is useful for backup, TiSpark, and TiFlash
  * Admin access: this gives access to PD/TiKV administrative APIs

By default, a token will have no access.
Token data will include the fields:
* admin: an object with fields
  * Allowed (boolean): If true, user is allowed admin API access
* Ranges: a list of allowed key range accesses (currently unsupported)
  * Each range in the list has fields
    * Start
    * End
    * Mode: (string). Currently supported are:
      * Full
      * readonly
  

When TiKV receives an API request, it decrypts the token and checks the permissions.
For an API request it checks the admin field.
For a non-administrative API request, it checks that the keys in the API request are within the start and end of the range.
If the mode is readonly, it will not allow access to write APIs.


## Tooling changes
TiSpark, TiFlash, and user tools must be adopted to perform a delegated token login.


## Encryption
Encryption will use the PASETO standard. There are implementations in many programming languages.
Golang https://github.com/o1egl/paseto
Rust https://github.com/instructure/paseto
Replay attacks are currently only prevented by token expiration (see the Refresh section).


# Refreshing
All tokens will expire after a small time period (suggested: 30 minutes). Applications should start trying to refresh their tokens after 5 minutes.

# Refresh tokens
Login APIs that return a token should also return a refresh token.
The refresh token can be used to refresh the existing tokens without providing credentials again.
Note that the refresh token itself is refreshed.
PD tracks refresh tokens and therefore can detect a stolen refresh token when there are duplicate refresh attempts with the same token.
The refresh token should only be kept in memory by the client.

# Admin access to PD, TIKV, and TiDB APIs
PD and TIKV expose administrative APIs.
Access to these is granted to any application that has the “admin” flag set to true.
By setting it to true for TiDB, this allows us to use TiDB delegation tokens to manage access if we want to.
If a user does not run TiDB, or TiDB cannot connect, the admin application must be used for access.
Admin access must be integrated into pd-ctl and tikv-ctl. tidb-operator should seed its own user with admin access.

# PD Dashboard
The PD dashboard is thought of as a separate entity from PD admin APIs.
I believe a large part of this is due to complications of authorization.
However, it is really just some new PD APIs and a single point of access that will proxy to TiDB admin APIs.
With proper authorization it is easy to think of things as they are and put them in their proper place.

This situation is exactly the same as the above admin APIs.
The only difference is that the dashboard will connect to other components, particularly TiDB.
In the current iteration of the dashboard, we must run a MySQL client to connect to TiDB from the PD Dashboard in order to login.
However, after this proposal is implemented, we can use the delegation login API of TiDB, and that token can be used for HTTP APIs.
We could also move the TiDB part into TiDB where it belongs (PD shouldn't be executing SQL) and PD could just proxy access to TiDB for the TiDB part of the TiDB dashboard.



# Details Missing
* Does cert re-loading always work well?
* API details of sending tokens
* Discuss where things connect to existing code


# Future Additions
* Integrate with Key spaces for multiple applications
* Additional ways to prevent replay attacks (confirm ip address, blacklist the jti field)
