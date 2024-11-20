# Crunchyroll Game Vault API Cookbook

Crunchyroll's Game Vault provides an ever-expanding set of games to our users. Subscribers that have the appropriate membership tier can access the Game Vault and download games to their mobile device as they would with other mobile games.

Full integration with Game Vault provides a standardized login flow for your users. When they open your game they'll see a login page where they can enter their Crunchyroll credentials. If they have a subscription with Game Vault access, the flow will let them access your game immediately. If not, it will guide them to a screen where they can upgrade their subscription to the appropriate tier.

For any questions about processes and APIs not covered in this Cookbook, reach out to your Crunchyroll representative.

## Game Vault APIs

As a Game Vault partner, you'll use several APIs to integrate with Crunchyroll services. The endpoints outlined in this document come from the following Crunchyroll APIs:

* [`cx-account`](https://github.com/crunchyroll/)
* [`cx-account-auth`](https://github.com/crunchyroll/)
* [`cx-partners`](https://github.com/crunchyroll/)
* [`cx-subscription-processor`](https://github.com/crunchyroll/)

External access to these APIs is only available to approved Game Vault partners, teams, and vendors. You can use these endpoints to perform the following tasks:

* Create user accounts.
* Modify existing user accounts.
* Set up password recovery options.
* Create access tokens.
* Retrieve a user's subscription tier and its associated benefits.
* Validate an iOS user with an iTunes receipt.

During production of your Game Vault project you can access these API endpoints using the Crunchroll production server: `https://xxx.crunchyroll.com/`

If you need to use the staging server, you can access it with this address: `https://xxx.crunchyroll.com/`

!!! info
    You must set up and connect to the [Crunchyroll VPN](https://crunchyroll.wiki) before you can use the staging server.

**Endpoints Overview**

| Method and Endpoint | Description |
| -- | -- |
| `POST /accounts/v1` | Creates a new user account through the `cx-account` API. |
| `GET /accounts/v1/{account_id}/profile` | Retrieves the profile information for a user's account. |
| `GET /accounts/v1/tokens/password/{token}` | Uses an account's token ID to retrieve an account-specific password reset token. |
| `POST /accounts/v1/password_forgot` | Triggers the `cx-account` API to send a user a password reset email. |
| `POST /accounts/v1/password_reset` | Changes a user's password using their one-time password reset token. |
| `POST /auth/v1/token` | Retrieves a user's access token after providing a set of parameters. The parameters you use depend on the authorization method your game uses. |
| `POST /itunes/receipt-verification` | Validates a user's iTunes receipt and then associates that receipt with a subscription account. |
| `GET /subscriptions/{account_id}/benefits` | Retrieves a list of benefits associated with the requested account. |

### POST /accounts/v1

Creates a new user account, including any optional profile information.

``` title="URI"
POST https://xxx.crunchyroll.com/accounts/v1
```

#### Header Parameters

| Name | Type | Required | Description |
| -- | -- | -- | -- |
| **X-Tenant** | string | Yes | The requested tenant. A list of supported tenants is available in the [API service config file](https://github.com/crunchyroll/).<br>Do not use if a client accesses the external server through an AWS CloudFront URL. |

#### Request Body

The parameters here are defined in the `CreateAccount` schema.

You can provide additional profile information by adding fields from the [profile validation configuration](https://github.com/crunchyroll/).

| Name | Type | Required | Description |
| -- | -- | -- | -- |
| **email** | string | Yes | The email address for the new account. |
| **password** | string | Yes | The plain-text password for the new account. |
| **username** | string |  | The username for the new account. |
| **profile_name** | string |  | The profile name to use for the main account.<br>This value is alphanumeric and must contain at least one (1) character. 50 character maximum. |

##### Example Request

```js
{
  "email": "example@example.com",
  "password": "myverystrongpassword",
  "username": "foo_bar_92",
  "maturity_rating": "M3",
  "extended_maturity_rating":
    { "BR": "18" },
  "profile_name": "profileName1234"
}
```

#### Responses

| Code | Description |
| -- | -- |
| **204** | Account successfully created. |
| **400** | Request validation failed. |
| **401** | Authorization failed. |
| **403** | Access is forbidden. |

## Game Vault Silent Login

Silent login provides a method for mobile apps to access Game Vault APIs. This service is available as an option in apps developed for Android and iOS.

### Android

Users must install and sign in to the Android app. The app must be distributed using the Crunchyroll Google Play account before they can use silent login.

You can access a sample app on the [`cx-android-app` GitHub repository](https://github.com/crunchyroll/). If you don't have access, contact your Crunchyroll representative.

#### Set Up an Android App

To view refresh tokens and enable silent login for your Android app, add the following permission to your `AndroidManifest.xml` file:

```xml
<uses-permission android:name="com.crunchyroll.crunchyroid" />
```

Add the following tag to the `AndroidManifest.xml` file of the app that will access silent login provider:

```xml
<queries>
    <provider
        android:authorities="com.crunchyroll.crunchyroid" />
</queries>
```

The following example uses a Kotlin function to access a refresh token with `CredentialsProvider`:

```kotlin
fun getRefreshToken(): String {
    val uri = Uri.parse("content://com.crunchyroll.crunchyroid")
    val cursor = contentResolver.query(uri, null, null, null, null)
    if (cursor != null) {
        if (cursor.moveToFirst()) {
            val refreshToken = cursor.getString(0)
            return refreshToken
        }
    }
    cursor?.close()
    return ""
}
```

### iOS

On iOS, apps save the user's refresh token in a shared keychain that any app from the same app group can access. Apps with the same App ID Prefix setup are considered part of the same app group.

Before you can enable silent login for an iOS app, you must be a member the Crunchyroll Apple Development Team. Your app must be distributed through Crunchyroll's [AppStoreConnect](https://appstoreconnect.apple.com/login) account or with [TestFlight](https://developer.apple.com/testflight/).

A user's refresh token contains the following information in UTF8 format:

| | |
| -- | -- |
| **Keychain Service Name** | `crunchyroll.ios` |
| **Keychain Access Group** | `crunchyroll.iphone` |
| **Keychain Key** | `crunchyroll.token` |

#### Set Up an iOS App

**To set up your app for iOS**

1. Make sure you have **Keychain Sharing** enabled in the Sharing & Capabilities menu of Xcode.
1. Add *crunchyroll.iphone* to Keychain Groups.
1. Add the `$(AppIdentifierPrefix)crunchyroll.iphone` keychain group to your app's *\*.entitlements* file.

The following code snippet performs a silent login if there is a valid refresh token in the user's shared Keychain.

```js
func performSilentLogin() {
    let keychainService = "crunchyroll.ios"
    let keychainRefreshTokenKey = "crunchyroll.token"
    let keychainAccessGroup = "crunchyroll.iphone"
    let keychain = Keychain(service: keychainService, accessGroup: keychainAccessGroup)
    if let token = try? keychain.get(keychainRefreshTokenKey) {
        // Get JWT using refresh token by calling "POST /auth/v1/token" with
        // grant_type = refresh_token
        // refresh_token = {token}
    } else {
        // CR iOS app is not installed or user is not logged in
    }
}
```

## JSON Web Key Sets

Crunchyroll provides a [method to obtain JSON Web Key](https://xxx.crunchyroll.com/) (JWK) sets that let you validate a user's JSON Web Token (JWT). If the user's JWT contains a `cr_bento` claim, they are on a subscription tier that includes Game Vault. JWKs are cached for up to one (1) hour and are rotated at least once every 30 days.

!!! warning
    Even though JWTs are limited in scope by the client ID issued to your team, they are still considered sensitive data and you should never store or log them on a server.
