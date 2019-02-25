LTO Network client for PHP
===

[![Build Status](https://travis-ci.org/legalthings/lto-api.php.svg?branch=master)](https://travis-ci.org/legalthings/lto-api.php)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/legalthings/lto-api.php/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/legalthings/lto-api.php/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/legalthings/lto-api.php/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/legalthings/lto-api.php/?branch=master)
[![Packagist Stable Version](https://img.shields.io/packagist/v/legalthings/lto-api.svg)](https://packagist.org/packages/legalthings/lto-api)
[![Packagist License](https://img.shields.io/packagist/l/legalthings/lto-api.svg)](https://packagist.org/packages/legalthings/lto-api)

_Signing and addresses work both for the (private) event chain as for the public chain._ 

Installation
---

    composer require legalthings/lto-api

Usage
---

### Creation

#### Create an account from seed

```php
$seedText = "manage manual recall harvest series desert melt police rose hollow moral pledge kitten position add";

$factory = new LTO\AccountFactory('T'); // 'T' for testnet, 'L' for mainnet
$account = $factory->seed($seedText);
```

#### Create an account from sign key

```php
$secretKey = 'wJ4WH8dD88fSkNdFQRjaAhjFUZzZhV5yiDLDwNUnp6bYwRXrvWV8MJhQ9HL9uqMDG1n7XpTGZx7PafqaayQV8Rp';

$factory = new LTO\AccountFactory('T'); // 'T' for testnet, 'L' for mainnet
$account = $factory->create($secretKey);
```

#### Create an account from full info

```php
$accountInfo = [
  'address' => '3PLSsSDUn3kZdGe8qWEDak9y8oAjLVecXV1',
  'sign' => [
    'secretkey' => 'wJ4WH8dD88fSkNdFQRjaAhjFUZzZhV5yiDLDwNUnp6bYwRXrvWV8MJhQ9HL9uqMDG1n7XpTGZx7PafqaayQV8Rp',
    'publickey' => 'FkU1XyfrCftc4pQKXCrrDyRLSnifX1SMvmx1CYiiyB3Y'
  ],
  'encrypt' => [
    'secretkey' => 'BnjFJJarge15FiqcxrB7Mzt68nseBXXR4LQ54qFBsWJN',
    'publickey' => 'BVv1ZuE3gKFa6krwWJQwEmrLYUESuUabNCXgYTmCoBt6'
  ]
];

$factory = new LTO\AccountFactory('T'); // 'T' for testnet, 'L' for mainnet
$account = $factory->create($accountInfo);
```

Properties that are specified will be verified. Properties that are omitted will be generated where possible.  

### Signing (ED25519)

#### Sign a message

```php
$signature = $account->sign('hello world'); // Base58 encoded signature
```

#### Verify a signature

```php
if (!$account->verify($signature, 'hello world')) {
    throw new RuntimeException('invalid signature');
}
```

### Encryption (X25519)

#### Encrypt a message for another account

```php
$message = 'hello world';

$recipientPublicKey = "HBqhfdFASRQ5eBBpu2y6c6KKi1az6bMx8v1JxX4iW1Q8"; // base58 encoded X25519 public key
$recipient = $factory->createPublic(null, $recipientPublicKey);

$cyphertext = $account->encryptFor($recipient, $message); // Raw binary, not encoded
```

You can use `$account->encryptFor($account, $message);` to encrypt a message for yourself.

#### Decrypt a message received from another account

```php
$senderPublicKey = "HBqhfdFASRQ5eBBpu2y6c6KKi1az6bMx8v1JxX4iW1Q8"; // base58 encoded X25519 public key
$sender = $factory->createPublic(null, $senderPublicKey);

$message = $account->decryptFrom($sender, $cyphertext);
```

You can use `$account->decryptFrom($account, $message);` to decrypt a message from yourself.

### Event chain

#### Create a new event chain

```php
$chain = $account->createEventChain(); // Creates an empty event chain with a valid id and last hash
```

_Note: You need to add an identity as first event on the chain. This is **not** done automatically._

#### Create and sign an event and add it to an existing event chain

```php
$body = [
  "$schema": "http://specs.livecontracts.io/01-draft/12-comment/schema.json#",
  "identity": {
    "$schema": "http://specs.livecontracts.io/01-draft/02-identity/schema.json#",
    "id": "1bb5a451-d496-42b9-97c3-e57404d2984f"
  },
  "content_media_type": "text/plain",
  "content": "Hello world!"
];

$chainId = "JEKNVnkbo3jqSHT8tfiAKK4tQTFK7jbx8t18wEEnygya";
$chainLastHash" = "3yMApqCuCjXDWPrbjfR5mjCPTHqFG8Pux1TxQrEM35jj";

$chain = new LTO\EventChain($chainId, $chainLastHash);

$chain->add(new Event($body))->signWith($account);
```

You need the chain id and the hash of the last event to use an existing chain.

### HTTP Authentication

Signing HTTP Messages is described IETF draft [draft-cavage-http-signatures-10](https://tools.ietf.org/id/draft-cavage-http-signatures-10.html).

The [HTTP Authentication library](https://github.com/legalthings/http-signature-php) can be used to sign and verify
[PSR-7 requests](https://www.php-fig.org/psr/psr-7/#33-psrhttpmessageresponseinterface).

This library can be used in conjunction with the HTTP authentication library. The `keyId` should be the base58 encoded
public key.

#### Creating the HTTP Authentication service

```php
use LTO\HttpSignature\HttpSignature;

$secretKey = 'wJ4WH8dD88fSkNdFQRjaAhjFUZzZhV5yiDLDwNUnp6bYwRXrvWV8MJhQ9HL9uqMDG1n7XpTGZx7PafqaayQV8Rp';

$factory = new LTO\AccountFactory('T'); // 'T' for testnet, 'L' for mainnet
$ourAccount = $factory->create($secretKey);

$service = new HttpSignature(
    ['ed25519', 'ed25519-sha256'],
    new SignCallback($ourAccount),
    new VerifyCallback($accountFactory)
);
```

#### Server middleware

Create server middleware to verify incoming requests.

The `LTO\Account\ServerMiddleware` can be used to set the `account` attribute for a server request that contains a
`signature_key_id` attribute.

```php
use LTO\HttpSignature\ServerMiddleware as SignatureMiddleware;
use LTO\Account\ServerMiddleware as AccountMiddleware;
use Relay\RelayBuilder;

$factory = new LTO\AccountFactory('T'); // 'T' for testnet, 'L' for mainnet
$ourAccount = $factory->create($secretKey);

$service = new HttpSignature(
    ['ed25519', 'ed25519-sha256'],
    function() { throw new \LogicException('sign not supported'); },
    new VerifyCallback($accountFactory)
);

$signatureMiddleware = new SignatureMiddleware($service);
$accountMiddleware = new AccountMiddleware($factory);

$relayBuilder = new RelayBuilder($resolver);
$relay = $relayBuilder->newInstance([
    $signatureMiddleware->asDoublePass(),
    $accountMiddleware->asDoublePass(),
]);
```

The server middleware implements the PSR-15 `MiddlewareInterface` for single pass support and returns a callback for
double pass with the `asDoublePass()` method.

#### Client middleware

Create client middleware to sign outgoing requests.

```php
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Client;
use LTO\HttpSignature\ClientMiddleware;

$secretKey = 'wJ4WH8dD88fSkNdFQRjaAhjFUZzZhV5yiDLDwNUnp6bYwRXrvWV8MJhQ9HL9uqMDG1n7XpTGZx7PafqaayQV8Rp';

$factory = new LTO\AccountFactory('T'); // 'T' for testnet, 'L' for mainnet
$ourAccount = $factory->create($secretKey);

$service = new HttpSignature(
    ['ed25519', 'ed25519-sha256'],
    new SignCallback($ourAccount),
    function() { throw new \LogicException('verify not supported'); }
);

$middleware = new ClientMiddleware(
    $service->withAlgorithm('ed25519-sha256'),
    $ourAccount->getPublicKey()
);

$stack = new HandlerStack();
$stack->push($middleware->forGuzzle());

$client = new Client(['handler' => $stack]);
```
