# WebPush
> Web Push library for PHP

[![Build Status](https://travis-ci.org/Minishlink/web-push.svg?branch=master)](https://travis-ci.org/Minishlink/web-push)
[![SensioLabsInsight](https://insight.sensiolabs.com/projects/d60e8eea-aea1-4739-8ce0-a3c3c12c6ccf/mini.png)](https://insight.sensiolabs.com/projects/d60e8eea-aea1-4739-8ce0-a3c3c12c6ccf)

## Installation
`composer require minishlink/web-push`

## Usage
WebPush can be used to send notifications to endpoints which server delivers web push notifications as described in 
the [Web Push protocol](https://tools.ietf.org/html/draft-thomson-webpush-protocol-00).
As it is standardized, you don't have to worry about what server type it relies on.

Notifications with payloads are supported with this library on Firefox 46+ and Chrome 50+.

```php
<?php

use Minishlink\WebPush\WebPush;

// array of notifications
$notifications = array(
    array(
        'endpoint' => 'https://updates.push.services.mozilla.com/push/abc...', // Firefox 43+
        'payload' => 'hello !',
        'userPublicKey' => 'BPcMbnWQL5GOYX/5LKZXT6sLmHiMsJSiEvIFvfcDvX7IZ9qqtq68onpTPEYmyxSQNiH7UD/98AUcQ12kBoxz/0s=', // base 64 encoded, should be 88 chars
        'userAuthToken' => 'CxVX6QsVToEGEcjfYPqXQw==', // base 64 encoded, should be 24 chars
    ), array(
        'endpoint' => 'https://android.googleapis.com/gcm/send/abcdef...', // Chrome
        'payload' => null,
        'userPublicKey' => null,
        'userAuthToken' => null,
    ), array(
        'endpoint' => 'https://example.com/other/endpoint/of/another/vendor/abcdef...',
        'payload' => '{msg:"test"}',
        'userPublicKey' => '(stringOf88Chars)', 
        'userAuthToken' => '(stringOf24Chars)',
    ),
);

$webPush = new WebPush();

// send multiple notifications with payload
foreach ($notifications as $notification) {
    $webPush->sendNotification(
        $notification['endpoint'],
        $notification['payload'], // optional (defaults null)
        $notification['userPublicKey'], // optional (defaults null)
        $notification['userAuthToken'] // optional (defaults null)
    );
}
$webPush->flush();

// send one notification and flush directly
$webPush->sendNotification(
    $notifications[0]['endpoint'],
    $notifications[0]['payload'], // optional (defaults null)
    $notifications[0]['userPublicKey'], // optional (defaults null)
    $notifications[0]['userAuthToken'], // optional (defaults null)
    true // optional (defaults false)
);
```

### Client side implementation of Web Push
There are several good examples and tutorials on the web:
* Mozilla's [ServiceWorker Cookbooks](https://serviceworke.rs/push-payload.html) (outdated as of 03-20-2016, because it does not take into account the user auth secret)
* Google's [introduction to push notifications](https://developers.google.com/web/fundamentals/getting-started/push-notifications/) (as of 03-20-2016, it doesn't mention notifications with payload)
* you may take a look at my own implementation: [sw.js](https://github.com/Minishlink/physbook/blob/07433bdb5fe4e3c7a6e4465c74e3b07c5a12886c/web/service-worker.js) and [app.js](https://github.com/Minishlink/physbook/blob/2a468273665a241ddc9aa2e12c57d18cd842d965/app/Resources/public/js/app.js) (payload sent indirectly)

### GCM servers notes (Chrome)
For compatibility reasons, this library detects if the server is a GCM server and appropriately sends the notification.

You will need to specify your GCM api key when instantiating WebPush:
```php
<?php

use Minishlink\WebPush\WebPush;

$endpoint = 'https://android.googleapis.com/gcm/send/abcdef...'; // Chrome
$apiKeys = array(
    'GCM' => 'MY_GCM_API_KEY',
);

$webPush = new WebPush($apiKeys);
$webPush->sendNotification($endpoint, null, null, null, true);
```

### Payload length and security
Payload will be encrypted by the library. The maximum payload length is 4078 bytes (or ASCII characters).

However, when you encrypt a string of a certain length, the resulting string will always have the same length,
no matter how many times you encrypt the initial string. This can make attackers guess the content of the payload.
In order to circumvent this, this library can add some null padding to the initial payload, so that all the input of the encryption process
will have the same length. This way, all the output of the encryption process will also have the same length and attackers won't be able to 
guess the content of your payload. The downside of this approach is that you will use more bandwidth than if you didn't pad the string.
That's why the library provides the option to disable this security measure:

```php
<?php

use Minishlink\WebPush\WebPush;

$webPush = new WebPush();
$webPush->setAutomaticPadding(false); // disable automatic padding
```

### Time To Live
Time To Live (TTL, in seconds) is how long a push message is retained by the push service (eg. Mozilla) in case the user browser 
is not yet accessible (eg. is not connected). You may want to use a very long time for important notifications. The default TTL is 4 weeks. 
However, if you send multiple nonessential notifications, set a TTL of 0: the push notification will be delivered only 
if the user is currently connected. For other cases, you should use a minimum of one day if your users have multiple time 
zones, and if they don't several hours will suffice.

```php
<?php

use Minishlink\WebPush\WebPush;

$webPush = new WebPush(); // default TTL is 4 weeks
// send some important notifications...

$webPush->setTTL(3600);
// send some not so important notifications

$webPush->setTTL(0);
// send some trivial notifications
```

### Changing the browser client
By default, WebPush will use `MultiCurl`, allowing to send multiple notifications in parallel.
You can change the client to any client extending `\Buzz\Client\AbstractClient`.
Timeout is configurable in the constructor.

```php
<?php

use Minishlink\WebPush\WebPush;

$client = new \Buzz\Client\Curl();
$timeout = 20; // seconds
$webPush = new WebPush(array(), null, $timeout, $client);
```

You have access to the inner browser if you want to configure it further.
```php
<?php

use Minishlink\WebPush\WebPush;

$webPush = new WebPush();

/** @var $browser \Buzz\Browser */
$browser = $webPush->getBrowser();
```

## Common questions

### Is there any plugin/bundle/extension for my favorite PHP framework?
The following are available:

- Symfony: [MinishlinkWebPushBundle](https://github.com/Minishlink/web-push-bundle)

Feel free to add your own!

### Is the API stable?
Not until the [Push API spec](http://www.w3.org/TR/push-api/) is finished.

### What about security?
Payload is encrypted according to the [Message Encryption for Web Push](https://tools.ietf.org/html/draft-ietf-webpush-encryption-01) standard,
using the user public key and authentication secret that you can get by following the [Web Push API](http://www.w3.org/TR/push-api/) specification.

Internally, WebPush uses the [phpecc](https://github.com/phpecc/phpecc) Elliptic Curve Cryptography library to create 
local public and private keys and compute the shared secret.
Then, if you have a PHP >= 7.1, WebPush uses `openssl` in order to encrypt the payload with the encryption key.
Otherwise, if you have PHP < 7.1, it uses [Spomky-Labs/php-aes-gcm](https://github.com/Spomky-Labs/php-aes-gcm), which is slower.

### How to solve "SSL certificate problem: unable to get local issuer certificate" ?
Your installation lacks some certificates.

1. Download [cacert.pem](http://curl.haxx.se/ca/cacert.pem).
2. Edit your `php.ini`: after `[curl]`, type `curl.cainfo = /path/to/cacert.pem`.

You can also force using a client without peer verification.

### I need to send notifications to native apps. (eg. APNS for iOS)
WebPush is for web apps.
You need something like [RMSPushNotificationsBundle](https://github.com/richsage/RMSPushNotificationsBundle) (Symfony).

### This is PHP... I need Javascript!
This library was inspired by the Node.js [marco-c/web-push](https://github.com/marco-c/web-push) library.

## Contributing
See [CONTRIBUTING.md](https://github.com/Minishlink/web-push/blob/master/CONTRIBUTING.md).

## Tests
Copy `phpunit.xml` from `phpunit.dist.xml` and fill it with your test endpoints and private keys.

## License
[MIT](https://github.com/Minishlink/web-push/blob/master/LICENSE)
