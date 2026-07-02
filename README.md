# Webhooks Handler for Emailing providers

[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE)
[![codecov](https://codecov.io/gh/Erwane/whep-client/graph/badge.svg?token=L98IZZFBY2)](https://codecov.io/gh/Erwane/whep-client)
[![CI](https://github.com/Erwane/whep-client/actions/workflows/ci.yml/badge.svg)](https://github.com/Erwane/whep-client/actions)
[![Packagist Downloads](https://img.shields.io/packagist/dt/Erwane/whep-client)](https://packagist.org/packages/Erwane/whep-client)
[![Packagist Version](https://img.shields.io/packagist/v/Erwane/whep-client)](https://packagist.org/packages/Erwane/whep-client)

This is the base project to easily handle webhooks sent by different emailing providers and uniformizing in
a comprehensive object.

This project is not made to be used alone, you need to pick your providers handlers corresponding to your project.

## Deprecated

Use `erwane/whep`. https://github.com/Erwane/whep

## Available providers handlers

| Provider                                | Package                                                       |
|-----------------------------------------|---------------------------------------------------------------|
| [Brevo](https://www.brevo.com/)         | [erwane/whep-brevo](https://github.com/Erwane/whep-brevo)     |
| [Mailgun](https://www.mailgun.com/)     | [erwane/whep-mailgun](https://github.com/Erwane/whep-mailgun) |
| [Mailjet](https://www.mailjet.com/)     | [erwane/whep-mailjet](https://github.com/Erwane/whep-mailjet) |
| [Postal](https://docs.postalserver.io/) | [erwane/whep-postal](https://github.com/Erwane/whep-postal)   |

## Usage

```shell
composer require erwane/whep-<provider>
```

```php
use WHEP\Factory;  
use WHEP\Exception\SecurityException;  
use WHEP\Exception\WHEPException;  
use WHEP\ProviderInterface;  

try {
    $provider = Factory::provider('<provider>', [
        'client_ip' => $_SERVER['REMOTE_ADDR'] ?? null, // Use your framework correct method to get the client ip.
        'callbacks' => [
            ProviderInterface::EVENT_BLOCKED => [$this, 'callbackInvalidate'],
            ProviderInterface::EVENT_BOUNCE_HARD => [$this, 'callbackInvalidate'],
            ProviderInterface::EVENT_BOUNCE_QUOTA => [$this, 'callbackUnsub'],
        ],
    ]);

    // process the data.
    $provider->process($webhookData);
    
    // Data available from provider getters.
    $recipient = $provider->getRecipient();
    
    // Launch callback
    $provider->callback();
} catch (SecurityException $e) {
    // log ?
} catch (WHEPException $e) {
    // log ?
}
```

## Options

You can pass options to `Factory::provider('<provider>', $options)` method.  
All available options are:

* `client_ip`: The client IP who request your url. Default `null`
* `allowed_ip`: Array of IPv4/IPv6 network (range) and allowed IP. Default depends on provider.
* `check_ip`: Set to false to bypass security IP check. Default is `false`.
* `signing_key`: Your provider private key to validate request came from trusted provider. Default `null`
* `callbacks`: You `callable` you want to be called, depends on event type.

### Security

Except if your webhook url has a security token, you can't ensure the webhook really came from trusted provider.  
Some providers use a signing key to validate data or provide an IP addresses list.

#### IP validation

When provider publish his IP addresses, you should pass the webhook client IP to the provider.

```php
Factory::provider('mailjet', ['client_ip' => $_SERVER['REMOTE_ADDR'] ?? null]);
```

When provider is self-hosted, like [Postal](https://docs.postalserver.io/), you can pass your postal server IP.

```php
Factory::provider('postal', [
    'client_ip' => $_SERVER['REMOTE_ADDR'] ?? null,
    'allowed_ip' => [
        '10.0.0.1',
        'fe80::0023:1',
        '192.168.0.1/24',
    ],
]);
```

You can bypass IP check with `check_ip` sets to `false`.

```php
Factory::provider('mailjet', ['check_ip' => false]);
```

#### Signing key

When provider support signing key, you can pass your private key with `signing_key` option.

```php
Factory::provider('mailgun', ['signing_key' => 'my-private-signing-key']);
```

The validation is done during `ProviderInterface::process()`

### Callbacks

Your callback method are cast when `$provider->callback()` is called (you decide when).
See [Event type & Callbacks](#event-type--callbacks) section for details.

```php
Factory::provider('<provider>', ['callbacks' => [ProviderInterface::EVENT_UNSUB => [$this, 'callbackUnsub']]]);
```

#### Event type & Callbacks

You can configure one callback by event type. Available callbacks are:

| Event                                   | Why event was emitted                                      |
|-----------------------------------------|------------------------------------------------------------|
| `ProviderInterface::EVENT_REQUEST`      | You send an e-mail to your provider.                       |
| `ProviderInterface::EVENT_DEFERRED`     | The send was deferred by provider.                         |
| `ProviderInterface::EVENT_BLOCKED`      | The recipient e-mail is in provider blocklist.             |
| `ProviderInterface::EVENT_SENT`         | E-mail was sent.                                           |
| `ProviderInterface::EVENT_BOUNCE_SOFT`  | E-mail receive a soft-bounce (4xx) with reason.            |
| `ProviderInterface::EVENT_BOUNCE_QUOTA` | Like BOUNCE_SOFT but quota problem detected.               |
| `ProviderInterface::EVENT_BOUNCE_HARD`  | E-mail receive a hard-bounce (5xx) with reason.            |
| `ProviderInterface::EVENT_OPENED`       | E-mail was opened.                                         |
| `ProviderInterface::EVENT_CLICK`        | A link was clicked.                                        |
| `ProviderInterface::EVENT_ABUSE`        | Recipient report your e-mail as abuse.                     |
| `ProviderInterface::EVENT_UNSUB`        | Recipient want to unsubscribed from you list.              |
| `ProviderInterface::EVENT_BLOCKLIST`    | You provider IP is in MX recipient blocklist (spam/dnsbl). |
| `ProviderInterface::EVENT_ERROR`        | Provider error.                                            |

## Methods

ProviderInterface has the following methods:

- [getName()](#getname)
- [getTime()](#gettime)
- [getType()](#gettype)
- [getRecipient()](#getrecipient)
- [getDetails()](#getdetails)
- [getSmtpResponse()](#getsmtpresponse)
- [getUrl()](#geturl)
- [getRaw()](#getraw)
- [process()](#process)
- [callback()](#callback)
- [securityChecked()](#securitychecked)

### getName()

Return provider name.

```php
echo $provider->getName();
```

### getTime()

Get event time as `\DateTimeInterface`. This represents when hook was received, not event time.

```php
$time = $provider->getTime();
```

### getType()

Return event type. See [Event type & Callbacks](#event-type--callbacks) for all types.

```php
if ($provider->getType() === \WHEP\ProviderInterface::EVENT_UNSUB) {
    // Do something
}
```

### getRecipient()

Return event related e-mail recipient.

```php
echo $provider->getRecipient();
```

### getDetails()

Return provider event details (or reason).

```php
echo $provider->getDetails();
```

### getSmtpResponse()

Return recipient MX SMTP response.

```php
echo $provider->getSmtpResponse();
```

### getUrl()

Return url of clicked link. Available for `\WHEP\ProviderInterface::EVENT_CLICK` only.  
Some providers (mailgun) do not return this information.

```php
echo $provider->getUrl();
```

### getRaw()

Return event raw data as array by default. Return as json if `$asJson` is `true`.

```php
$raw = $provider->getRaw();

// raw data in json format.
echo $provider->getRaw(true);
```

### process()

Process the webhook data. This method is chainable.

```php
$provider = \WHEP\Factory::provider('mailgun')
    ->process($webhookData);
```

### callback()

Run you related event type callable if configured.

```php
// This will process data and call self::callbackUnsub($provider) if event is unsub.
$provider = \WHEP\Factory::provider('mailgun', [
    'callbacks' => [
        \WHEP\ProviderInterface::EVENT_UNSUB => [$this, 'callbackUnsub'],
    ],
])
    ->process($webhookData)
    ->callback();
```

### securityChecked()

Return true if security was checked. Default to `false`.

```php
if (!$provider->securityChecked()) {
    // Your webhook url deserve security.
}
```
