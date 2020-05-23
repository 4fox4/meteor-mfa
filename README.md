Multi-Factor Authentication and Passwordless for Meteor, supporting U2F hardware keys as well as OTPs.

#### What is U2F?

This package was created to add support for U2F aka hardware keys. Hardware keys are widely regarded as the strongest form of multi factor authentication. They require the user plugs a physical key into their device in order to login to a website. This also includes built in anti-phishing security, if a key is used to authenticate on the wrong domain (go0gle.com instead of google.com), the signature it produces will be invalid to the intended domain. This means, barring a compromised device/browser, its very difficult for phishers to attack users who use U2F keys.

This package also supports OTPs (codes delivered via SMS, email, etc).

#### Known Issues
- Password Resets that fail MFA authentication will still expire token (e.g. if used has a typo in their code they will require a new token)

#### Roadmap/To-Do
If you are interested in contributing on any of these tasks please reach out!

- Passwordless (There is now a [release candidate](https://github.com/TheRealNate/meteor-mfa/releases/tag/v0.0.12-rc) for passwordless, if you'd like to check it out)
- Add rate limiting
- Create `meteor-react-native` companion package
- Implement Automated Testing
- Support Recovery Keys (Keys generated at time of setting up MFA which can be used when no other method is available)
- Add "Login other device" support: generate an OTP from a device capable of using U2F to be entered on a device not capable of U2F
- Support requiring more then 2 factors to authenticate

<h1>Table of Contents</h2>

- [Getting Started](#installation)
  - [Universal 2nd Factor (U2F)](#u2f)
  - [One Time Passwords (OTPs)](#otp)
  - [Time-Based One Time Passwords (TOTPs)](#totp)
  - [Logging in a user](#login)
  - [Retrieving user's MFA status](#retrieving-mfa-status)
  - [Resetting Passwords with MFA](#reset-password)
  - [Authenticating in a Meteor method](#method-authentication)  
  - [Authorizing Another Device (U2F MFA on devices without U2F)](#authorizing)
  - [Passwordless](#passwordless)  
- [API Docs](#api-docs)
  - [Client](#client-api)
  - [Server](#server-api)
  

<h2 id="installation">Getting Started</h2>

Add the package with `meteor add ndev:mfa`

You can then follow the instructions below for setting up U2F and/or OTP.

*Note that U2F is enabled by default, and OTP is disabled by default*

<h3 id="u2f">Universal 2nd Factor (U2F)</h3>

First, Set required configuration fields on server:
````
import MFA from 'meteor/ndev:mfa';

MFA.setConfig({
  rp:{
    id:"thedomain.of.your.app",
    name:"My App"
  }
});
````

And add the ability for users to enable MFA from the client
````
import MFA from 'meteor/ndev:mfa';

MFA.registerU2F().then(r => {
  alert("MFA is now turned on!");
}).catch(err => {
  alert("Something went wrong");
});
````

<h3 id="otp">One Time Passwords (OTPs)</h3>

One-Time-Passwords are codes typically sent via SMS or Email. This package takes a very simplistic approach to enrollment in order to let you control how the codes get delivered.

First, enable OTP in the config, and define a function for sending codes:
````
MFA.setConfig({
  enableOTP:true,
  onSendOTP:(userId, code) => {...}
})
````

And simply call this function from the server to enable OTP MFA for a user (note that it will fail if MFA is already enabled):
````
MFA.enableOTP(userId);
````

Since you have full control over how a code is sent, it is up to you to maintain the user's prefered method of delivery.

<h3 id="totp">Time-Based One Time Passwords (TOTPs)</h3>

Time-Based One Time Passwords are generated from an app. The client can register their TOTP device like so:

````
MFA.registerTOTP().then(r => {
  // The secret can be retrieved from r.secret, and should be shown to the user in a QR code
  // You should save r.registrationId to be used when calling MFA.finishRegisterTOTP later
  
  // This should be triggered by your UI after the user adds the secret to their app and enters a token generated by the app
  let token = prompt("What is the token");
  MFA.finishRegisterTOTP(token, r.registrationId).then(() => {
    this.setState({secret:null});
  }).catch(e => {
    alert(e.reason);
  });
  
}).catch(e => {
  console.error(e);
  alert(e.reason);
});
````

<h3 id="login">Logging in a user</h3>

The package provides a method called `MFA.login`. This method will attempt to login the user normally, then if it fails due to mfa being required, runs `MFA.loginWithMFA`. If you prefer, you can customize your implementation by doing your own check to see if MFA exists then, if it does, directly calling `MFA.loginWithMFA`.

There are two steps to logging in with MFA:
1. Perform the first-factor login (password), and retrieve the MFA method
2. Obtain any necessary information from the user and finish the login

The flow has been designed this way so that the implementation for U2F and OTP (one-time-password/code) is as similar as possible.

First, you call `MFA.login` with your username/email and password. This will resolve with an object containing the method and a field called finishLoginParams, which can be stored to be passed to finishLoginParams later.

For U2F: you can then either immediately call `MFA.finishLogin(finishLoginParams)` or make some changes to your UI, then call it.

For OTP and TOTP: you can store finishLoginParams, update your UI to collect the OTP, then once the OTP has been entered call `MFA.finishLogin`.

Here is a simple example:

````
import MFA from 'meteor/ndev:mfa';

MFA.login(username, password).then(({method, finishLoginParams}) => {
  if(method === "u2f") {
    // For U2F, you don't really need to make any changes to your UI since the pop-up will appear immediately, but you can do-so here if you wish
    MFA.finishLogin(finishLoginParams);
  }
  else {
    //
    // You can save the finishLoginParams, collect the code in your UI, then call MFA.finishLogin
    let code = prompt("What is the OTP?"); // Note that prompt should NEVER be used as it blocks the JS thread. This is just a simplified example
    MFA.finishLogin(finishLoginParams, code);
  }
}).catch(err => {
  alert(err.message);
});
````

You can see a complete login page example [here](/examples/react/Login.jsx)

<h3 id="retrieving-mfa-status">Retrieving user's MFA status</h3>
The field specified in `config.mfaDetailsField` (default `mfa`) will contain an object with the properties enabled(Boolean) and type(String). You can publish this property to a user to allow them to check if MFA is enabled.

<h3 id="reset-password">Resetting Passwords with MFA</h3>
The `config.requireResetPasswordMFA` property defines whether a user must authenticate with MFA to reset their password. This is set to `false` by default.

To reset a password with MFA authentication, use the `MFA.resetPassword` method. The usage of this method is very similar to the usage of `MFA.login`:

````
MFA.resetPassword(token, newPassword).then(r => {
  if(r.method === null) {
    // The user doesn't have MFA enabled
  }
  else {
    if(method === "u2f") {
      // For U2F, you don't really need to make any changes to your UI since the pop-up will appear immediately, but you can do-so here if you wish
      MFA.finishResetPassword(finishLoginParams);
    }
    else {
      // You can save the finishLoginParams, collect the code in your UI, then call MFA.finishLogin
      let code = prompt("What is the OTP?"); // Note that prompt should NEVER be used as it blocks the JS thread. This is just a simplified example
      MFA.finishResetPassword(finishLoginParams, code);
    }    
  }
})
````

You should also set `config.requireResetPasswordMFA` to true. This will require that users use MFA when resetting their password.

<h3 id="method-authentication">Authenticating in a Meteor method</h3>

This package exposes the `MFA.generateChallenge` and `MFA.verifyChallenge` methods which allow you to integrate U2F authentication into your method. See an example below which requires the user authenticates before they can disable MFA:

````
Meteor.methods({
  "start:disableMFA":function () {
    let challenge = MFA.generateChallenge(this.userId, "disableMFA");
    return challenge;
  },
  
  "complete:disableMFA":function (solvedChallenge) {
    let isValid = MFA.verifyChallenge("disableMFA", solvedChallenge);
    
    if(!isValid) {
      throw new Meteor.Error(404, "MFA Challenge Failed");
    }
    
    MFA.disableMFA(this.userId);
  }  
});
````

<h3 id="authorizing">Authorizing Another Device (U2F MFA on devices without U2F)</h3>

U2F authentication is the most secure option. However, there are a lot of devices and platforms that are incompatible with it (React Native, for example).

This package exposes the MFA.authorizeAction method on the client to solve this. The MFA.authorizeAction function triggers U2F authentication, then creates a one-time-code which can be used in place of a solved challenge for any other method.

This one-time-code can then be passed to a different device to allow it to do something that requires U2F authentication.

**The `MFA.authorizeAction` method can only be used for accounts with U2F mfa.**

Here is how you generate the one-time-code:
````
MFA.authorizeAction(type).then(code => { // Type should refer to the action. If the one-time-code is for logging in, it should be "login"
  // Display the code in the UI
}).reject(e => {
  alert(e.reason);
});
````

On the other device, you collect the code and wrap it with the `MFA.useU2FAuthorizationCode(code)` method.

````
MFA.loginWithMFA(username, password).then((r) => {
  if(r.method === "u2f") {
    let code = prompt("Please enter your authorization code generated from another device"); // As always, you should never use prompt, this is just an example
    MFA.finishLogin(r.finishLoginParams, MFA.useU2FAuthorizationCode(code)).then(/*...*/)
  }
}).catch(/*...*/);
````

In order to design your login flows better, the package exposes the method `MFA.supportsU2FLogin()`. This attempts to detect whether the browser will support a u2f login. As a convenience, this value is also passed in the resolution of `MFA.login` or `MFA.loginWithMFA`. If the value is false, show the user instructions to generate a code on another device along with an input for them to enter the code.

Here is a complete login example:
````
import MFA from 'meteor/ndev:mfa';

MFA.login(username, password).then(({method, finishLoginParams, supportsU2FLogin}) => {
  if(method === "u2f") {
    if(supportsU2FLogin) {
      // For U2F, you don't really need to make any changes to your UI since the pop-up will appear immediately, but you can do-so here if you wish
      MFA.finishLogin(finishLoginParams);
    }
    else {
      let code = prompt("Enter the code generated from your other device");
      MFA.finishLogin(finishLoginParams, MFA.useU2FAuthorizationCode(code));
    }
  }
  else {
    let code = prompt("What is the OTP?"); 
    MFA.finishLogin(finishLoginParams, code);
  }
}).catch(err => {
  // There was an error in the first stage of login, likely a "user not found" or "incorrect password"
  alert(err.reason);
});
````
As always, prompt is used as an example. After `MFA.login` resolved, unless the method is u2f and u2f login is supported, you should save finishLoginParams, collect the code in your UI, then continue with MFA.finishLogin.

For situations where you are not logging in (like in the "Authenticating in a Meteor method" section above), you can use `MFA.useU2FAuthorizationCode(code)` in place of `MFA.solveChallenge()`.

<h3 id="passwordless">Passwordless</h3>

Passwords are widely regarded as the "weakest link" when it comes to security. Passwordless is really straightforward. No passwords. Instead, physical security keys.  This concept is being promoted by Microsoft, the FIDO Alliance (a consortium consisting of PayPal, Google, etc), and more giant tech companies.

This package only supports passwordless login using U2F security keys. It does *not* support passwordless using OTPs or TOTPs. 

This package enables passwordless login in a straightforward way. It is also flexible. You can have some users on passwordless, some users on MFA, and some users without either.

**1. Enable passwordless in config**

Passwordless is disabled by default. On the server, set `config.passwordless` to `true`:

````
MFA.setConfig({passwordless:true});
````

**2. Register the user's U2F key:**

To enable passwordless for a user, call `MFA.registerU2F` on the client with the following options:

````
MFA.registerU2F({passwordless:true, password:"user's current password here"}).then(() => {
  // All done!
}).catch(e => {
  // User cancelled U2F verification, incorrect password, etc
})
````

**3. Login with passwordless:**

On the client, call `MFA.loginWithPasswordless`. This method will catch only due to an error with U2F. If the user does not have passwordless enabled, it will resolve with `passwordRequired` as `true`:

````
MFA.loginWithPasswordless("email or username here").then(passwordRequired => {
  if(passwordRequired) {
    // User doesn't have passwordless enabled. Show regular login form.
  }
  else {
    // Passwordless login successfull
  }
}).catch(e => {
  // user cancelled U2F verification, etc
});
````

There are many ways to design your login flow. Here are some examples:
- Add a "Login with Security Key" button to your login page, and when clicked, collect their username/email and trigger `MFA.loginWithPasswordless`
- On your login form, only ask for email/username initially, then immediately attempt MFA.loginWithPasswordless. If it fails (due to passwordless not being enabled), then collect the password


<h1 id="api-docs">Full API Documentation</h1>

<h2 id="client-api">Client</h2>
`import MFA from 'meteor/ndev:mfa';`

#### MFA.login(email/username, password)<promise>
Resolves when logged in, catches on error. This function is a wrapper for the `MFA.loginWithMFA` function. It attempts to login the user. If it receives an `mfa-required` error, it uses `MFA.loginWithMFA`. If you prefer to customize this, you can use the `MFA.loginWithMFA` function

#### MFA.loginWithMFA(email/username, password)<promise>
Requests a login challenge, solves it, then logs in. This function will fail if the user doesn't have MFA enabled.

#### MFA.loginWithPasswordless(email/username)<promise:(passwordNeeded)>
Attempts a passwordless login. Resolves with a single boolean `passwordNeeded`. If true, the user doesn't have passwordless turned on, so you must use the regular login flow. If false, the user is now logged in.

#### MFA.finishLogin(finishLoginParams)<promise>
Completes a login

#### MFA.registerU2F(params)<promise>
Registers the user's U2F device and enables MFA. To just enable MFA, call without any arguments. To enable MFA and passwordless, call with the following params:

````
{passwordless:true, password:"..."}
````

#### MFA.registerTOTP()<promise>
Generates a TOTP secret and a registrationId. Resolves with `{secret, registrationId}`.

#### MFA.finishRegisterTOTP(token, registrationId)<promise>
Completes registration of a TOTP app.

#### MFA.solveChallenge(challenge)<promise>
Solves a challenge and resolves with a solved challenge. Useful for server-side methods that require MFA authentication (like disabling MFA). 

### MFA.resetPassword(token, newPassword)<promise>
Resets a password using the password reset token. See "Resetting Passwords with MFA" above for usage instructions. This method works even if the user doesn't have MFA enabled.

### MFA.resetPasswordWithMFA(token, newPassword)<promise>
Like MFA.resetPassword, but will fail if user doesn't have MFA enabled

### MFA.authorizeAction(type)<promise>
Creates a pre-authenticated code for challenges of a certain type

### MFA.useU2FAuthorizationCode(code)<promise>
Wraps a pre-authenticated code to be used in place of `MFA.solveChallenge()`

### MFA.supportsU2FLogin()<Boolean>
Returns a boolean of whether the device supports u2f login

<h2 id="server-api">Server</h2>
`import MFA from 'meteor/ndev:mfa';`

#### MFA.setConfig(options)
See the config options section below

#### MFA.disableMFA(userId)
Disables MFA for a user. This is an internal method. If you'd like the user to authenticate before you disable, see [Authenticating in a Method](#method-authentication). Note: if the user has passwordless enabled, this will also disable passwordless.

#### MFA.disablePasswordless(userId)
Disables passwordless for a user. When this method is called, MFA will remain enabled. To disable passwordless and MFA, call `MFA.disableMFA`.

#### MFA.generateChallenge(userId, type)
Generates a challenge. This is then sent to the client and passed into `MFA.solveChallenge()`

#### MFA.verifyChallenge(type, solvedCHallenge)
Verifies the solvedChallenge (created by `MFA.solveChallenge` on client).

### Config Options

**mfaDetailsField *String* (default: "mfa")** The field where the mfa status object is stored (this is the field you can publish to tell whether a user has enabled)

**challengeExpiry *Number* (default: 1 minute)** How long before a challenge expires (in milliseconds)

**getUserDetails *Function(userId)* (default: `{id:user._id, name:user.username}`)** A function that returns an object with the `id` and `name` properties. The function recieves a single argument, the id of the user.

**onFailedAssertion *Function(info)* (default: none)** Function that runs when an assertion is failed. The only situation where this can be produced by the user accidentally is if the timeout expires or the wrong key is plugged into the device. Therefore, you should consider notifying the user if this happens. This function receives a single argument, which is the info argument of `Accounts.validateLoginAttempt`.

**enableU2F *Boolean* (default:true)** Enable U2F Authentication

**enableTOTP *Boolean* (default:true)** Enable TOTP Authentication.

**enableOTP *Boolean* (default:false)** Enable OTP Authentication. If this is enabled, you will need to set `config.onSendOTP`.

**onSendOTP *Function* (default: none)** Function that is called when an OTP needs to be sent. Receives the arguments `(userId, code)`.

**requireResetPasswordMFA *Boolean* (default: false)** Require MFA authentication when resetting password. This is set to false by default because you must use the custom resetPassword method should you enable this. See "Resetting Passwords with MFA" above.

**enforceMatchingConnectionId *Boolean* (default: true)** Enforce that the connection id that finishes a login challenge is the same as the one that creates it

**enforceMatchingClientAddress *Boolean* (default: true)** Enforce that the client address (IP) that finishes a login challenge is the same as the one that creates it

**enforceMatchingUserAgent *Boolean* (default: true)** Enforce that the user agent that finishes a login challenge is the same as the one that creates it

**allowU2FAuthorization *Boolean* (default: true)** Allow the `MFA.authorizeAction` method

**authorizationDisabledMethods *Array* (default: `[]`)** Block certain challenge types from being validated using authorization codes generated by `MFA.authorizeAction`

**keepChallenges *Boolean* (default: false)** Defines whether challenges should be maintained in the database. When set to false, challenges are deleted after use. When set to true, challenges are marked as invalid, but remain in database.

