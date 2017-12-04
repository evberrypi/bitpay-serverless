A fork of the defunct [BitPay Node.js API Client](https://github.com/bitpay/node-bitpay-client/), updated for use with Firebase Functions/ Cloud Functions


Created because of this [issue](https://github.com/bitpay/node-bitpay-client/issues/87)

==========================
[![Build Status](https://travis-ci.org/evberrypi/bitpay-serverless.svg)](https://travis-ci.org/evberrypi/bitpay-serverless)


## Getting Started

Install using [Node Package Manager](https://www.npmjs.org/)... Alternatively use Yarn
```
~# npm install bitpay-serverless

```


If you do not use NPM to install (instead cloning this repository), you will
need to run the following from the project root:

```
~# npm run setup
```

### Pairing

Set up your client's private key:

```
./node_modules/bitpay-serverless/bin/bitpay.js keygen
< enter a password, or hit enter for no password >
Generating keys...
Keys saved to: /Users/<your_username>/.bitpay
```
Next you have to pair up your client's private key with your bitpay account. This is done by requesting a pairing code:

```
./node_modules/bitpay-serverless/bin/bitpay.js pair
Do you have a pairing code?
no < hit enter twice >
Okay, we can get a pairing code, please choose a facade:
  1) Point of Sale // Just want to make invoices
  2) Merchant      // Want to have full account access
```
This will spit out a bunch of output. At the end of it will be a URL:
```
Pair this client with your organization:
https://test.bitpay.com/api-access-request?pairingCode=XXX

If you are using production mode:
https://bitpay.com/api-access-request?pairingCode=XXX

```
Visit this URL in your browser and hit the approve button. Afterwards, you can test creating a basic invoice from the command line like this:

```
//If you selected #1 then use this:
./node_modules/bitpay-serverless/bin/bitpay.js request -T pos -X post -R invoices -P '{"price": 1, "currency": "USD"}'
//If you selected #2 then use this:
./node_modules/bitpay-serverless/bin/bitpay.js request -T merchant -X post -R invoices -P '{"price": 1, "currency": "USD"}'
```
If it worked, you'll see some JSON outputted regarding the newly created invoice. If you get an error like this:
```
Error: { error: 'Invalid token' }
```
Then something is wrong, either you used the wrong line, or you haven't approved the token yet in the bitpay dashboard.


For this utility Bitpay's test platform is used by default, so if you want to use the regular production platform (ie. bitpay.com and not test.bitpay.com), do this:
```
./node_modules/bitpay-rest/bin/bitpay.js config --use prod
```
You'll need to pair again as well to get a new token for the production environment.


## Usage

### CLI

Use the `bitpay` command line program to generate your client keys and
associate them with your account.

```
~# cd bitpay-rest && npm link
~# bitpay keygen
~# bitpay pair
```

If you switch your environment a lot, you can avoid editing your config file:

```
~# bitpay config --use prod
~# bitpay config --use test
```

You can even create custom preset configurations:

```
~# bitpay config --set apiHost --value gordon.bp
~# bitpay config --save local
~# bitpay config --use local
```

Last but not least, you can issue API requests directly from the command line:

```
~# bitpay request -T merchant -R invoices -P '{"dateStart":"2014-01-01"}'
```

For more information on how to use the CLI, run:

```
~# bitpay --help
```

If you are running this module as a Firebase Function, save the `api.key` and  `password` variables as environment variables as outlined [here](https://firebase.google.com/docs/functions/config-env)

### Firebase Config

```
firebase functions.config.set bitpay.pass="apiPasswordGoesHere" bitpay.key='apiKeyGoesHere'
```
### Module

Unlike in the original bitpay-rest module, bitpay-serverless is a standalone library, you will not need bitauth.


Here's a basic example for creating an invoice:
```js
var bitpay = require('bitpay-serverless');
// NOTE: necessary to decrypt your key even if you didn't enter a password when you generated it.
// If you did specify a password, pass it as the first param to bitauth.decrypt()
 const pass = functions.config().bitpay.pass; 
  const key = functions.config().bitpay.key;  
  const privkey = bitauth.decrypt(pass, key)
  // console.log(privkey)

// Note the original bitpay-rest was not accepting the local configuration file so it would not work on Firebase. 
// It is fixed not
var client = bitpay.createClient(privKey, {
  config: {
    apiHost: 'bitpay.com', // if testing, pass `test.bitpay.com` here instead
    apiPort: 443
  }});

client.on('error', function(err) {
  // handle client errors here
  console.log(err);
});

//Client will take a second to automatically load your tokens, after which it will emit this ready event
//You must wait for the ready event before issuing requests
client.on('ready', function(){
  var data = {
    price: 1,
    currency: 'USD'
  };

  // NOTE: the .as('pos') is necessary for Point of Sale requests, use as('merchant') if you have a merchant token instead
  client.as('pos').post('invoices', data, function(err, invoice) {
    if (err){
      // more error handling
      console.log(err);
    }
    else{
      // success
      console.log('invoice data', invoice);
    }
  });
});

```

When resources are returned, they get extended with the same methods as the
`client`, so you can chain requests onto them. For instance, to get the refunds
associated with the first invoice returned from the example above:

```js
client.get('invoices', function(err, invoices) {
    invoices[0].get('refunds', function(err, refunds) {
        console.log(err || refunds);
    });
});
```

Arguments for creating invoices can be viewed [here](https://bitpay.com/api#resource-Invoices)

### Assuming a Different Facade

Some operations in the API are only available to certain "facades", which
restrict access to different functionality. By default, all requests are sent
using the **merchant** facade. To assume a different facade, you can use the
`as()` method.

```js
client.as('payroll').get('payouts', { status: 'new' }, function(err, payouts) {
    async.eachSeries(payouts, function(payout, done) {
        payout.put({ status: 'cancelled' }, done);
    }, function(err) {
        console.log('Cancelled all new payout requests.');
    });
});
```

### Streaming Responses

All of the `client` methods return a `Stream`, which you may use for more
custom implementations. Here is a very rudimentary example using
[Clarinet](https://github.com/dscape/clarinet), a streaming JSON parser.

```js
var parser = require('clarinet').createStream();
var count  = 0;

parser.on('key', function(key) {
  if (key === 'id') {
    parser.once('value', function(val) {
      count++;
      console.log('Got invoice: ' + val);
    });
  }
});

parser.on('end', function() {
  console.log('Streamed ' + count + ' invoices!');
});

client.get('invoices').pipe(parser);
```
