# \<hwcrypto-button\>

`hwcrypto-button` is Polymer 2 element which wraps [hwcrypto.js](https://github.com/hwcrypto/hwcrypto.js) library into easy to reuse button.
Basically it is [`paper-button`](https://www.webcomponents.org/element/@polymer/paper-button) with predefined `on-tap` handler which fires events at key points. This helps to break one huge chain of Promises into smaller, easier to understand pieces.

<!---
```html
<custom-element-demo>
  <template>
    <script src="../webcomponentsjs/webcomponents-lite.js"></script>
    <link rel="import" href="hwcrypto-button.html">
    <next-code-block></next-code-block>
  </template>
</custom-element-demo>
```
-->
```html
<hwcrypto-button></hwcrypto-button>
```

## Installation

    $ bower install hwcrypto-button --save

**NB!** This should also install the [hwcrypto.js](https://github.com/hwcrypto/hwcrypto.js) library this element wraps, however, at the time of this writing the published release of the hwcrypto.js in the bower registry is 0.0.10 while in the GitHub repo version 0.0.13 is available. It is recommended that you manually download the latest version of the [hwcrypto.js](https://raw.githubusercontent.com/hwcrypto/hwcrypto.js/master/hwcrypto.js) file and overwrite the one in your project's `.../bower_components/hwcrypto/` directory.

## Usage
For example to implement login with EstEID:
### html
In the html of your page add the element onto the login page:
```html
<link rel="import" href="../bower_components/hwcrypto-button/hwcrypto-button.html">
...
<hwcrypto-button
  action="AUTH"
  on-hwcrypto-sign="onIdKaartSign"
  on-hwcrypto-signed="onIdKaartSigned"
  on-hwcrypto-validated="onValidated">
</hwcrypto-button>
```

Then in the script section of your page define functions `onIdKaartSign`, `onIdKaartSigned` and `onValidated` which have an event object as their parameter. The Events' `detail` is an object like:
```javascript
{
   info: '',
   action: 'AUTH',
   lang: 'et',
   hash: 'SHA-256',
   cert: null,
   value: null,
   signature: null,
   valid: false
}
```

### `hwcrypto-sign` event
The `hwcrypto-sign` event is fired before any data is sent to the hwcrypto lib and that's where you can set up the data to send to it.
What you must do here is to assign value to the `event.detail.value` property. The value must be a hash generated with algorithm given in `event.detail.hash` and that's the data you're going to sign with hwcrypto.js library. The value must be of type either `String` or `Uint8Array` or `Promise` which resolves to value of either of these types.

```javascript
function onIdKaartSign(event) {
  // "action", "lang", "hash" and "info" can be changed here
  event.detail.info = 'Youre about to login into "MySite"';
  // if someCondition is met then cancel signing
  if (someCondition) {
    event.preventDefault();
    return;
  }
  // Send the required hash type to server and get value to sign as a response.
  // NB! the request to the server must be either synchronous (so that you
  // wouldn't return from this eventhandler before you have the hash value assigned
  // to the `detail.value`) or you must assign an Promise to the `value` which
  // resolves to the hash to be signed.
  event.detail.value = new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.onerror = reject;
    xhr.onload = (e) => {
      var tkn = JSON.parse(xhr.response);
      // store the "token" for later use
      event.detail.token = tkn.token;
      // resolve the promise with hash value
      resolve(tkn.hash);
    };
    xhr.open("GET", "https://myserver.com/auth/hash/"+event.detail.hash);
    xhr.send();
  });
}
```
The assumption here is that server has a endpoint like "https://myserver.com/auth/hash/SHA-256" (one for each supported hash type) which returns JSON like
```javascript
{
  token: 'a23d45fcb09361ef',
  hash: '413140d54372f9baf481d4c54e2d5c7bcf28fd6087000280e07976121dd54af2'
}
```
The `token` is random string generated by server and it is used by the server to keep track of login attempt. When request comes in for new hash value server stores the generated hash and time it was generated using token as key. The generation time is used to periodically remove "expired hashes" from the list, typically they shouldn't be valid for more than few minutes.

### `hwcrypto-signed` event

The `hwcrypto-signed` event is fired when client successfully signed the hash given to it by the `hwcrypto-sign` event. Use this event to let the backend validate the signature and if everything checked out return an access token for the client. In this event one should assign `Boolean` (or `Promise` which resolves to `Boolean`) to the `event.detail.valid` property, `true` meaning that signature is valid and `false` that it is invalid.

```javascript
function onIdKaartSigned(event) {
   event.detail.valid = new Promise((resolve, reject) => {
     var data = new FormData();
     data.append("token", event.detail.token);
     data.append("cert", new Blob([event.detail.cert.encoded], {type: 'application/pkix-cert'}));
     data.append("sign", new Blob([event.detail.signature.value], {type: 'application/octet-stream'}));

     const r = new XMLHttpRequest();
     r.onerror = reject;
     r.onload = (e) => {
       // save access token for later use
       cur_user.token = r.response;
       // signal that login was successful
       resolve(true);
     };
     r.open("POST", "https://myserver.com/auth/token");
     r.send(data);
   });
}
```
The https://myserver.com/auth/token endpoint will look up the hash value using `token` sent by the client and if it is not expired checks wether the signature is valid for the hash and given certificate. If everything is OK it returns an access token for the client, otherwise appropriate error code is returned. As the certificate sent contains user's name and personal code that information can be used to authorize logged in user.

### `hwcrypto-validated` event

The `hwcrypto-validated` event is fired after validating the signature and here one should redirect user to the next page after login.

```javascript
function onValidated(event) {
  if (event.detail.valid) {
    // logged in! redirect to right page
  } else {
    // login failed
  }
}
```

## Credits
 - [`hwcrypto.js`](https://github.com/hwcrypto/hwcrypto.js) browser JavaScript library for working with hardware tokens.
 - The text "ID-kaart" used as the default caption of the button is a svg image extracted from the official image available at [Insignia and reference](https://www.id.ee/?lang=en&id=35768) page.

## License
MIT
