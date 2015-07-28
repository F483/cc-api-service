# cc-api-service

Provides a REST API for ChromaWallet colored coin libraries: construct transactions, broadcast transactions, query data.

Run:

    npm install
    node api-service.js --port 4444 --testnet


## Command-line options

Parameter | Meaning
----------|------------------------------------
port      | service port, defaults to 4444
testnet   | testnet or mainnet mode, defaults to mainnet
chromanode| chromanode URL, defaults to v1.livenet.bitcoin.chromanode.net

## API calls

### General conventions

Methods should be called using HTTP POST. Data should be encoded in JSON. Response is JSON-encoded. Transaction ids and transactions are hex-encoded. Server will respond with HTTP status 400 if request is not understood and HTTP status 500 in case of an error.

Call which construct transactions (`createIssueTx`, `createTransferTx`) accept 'transaction spec' which includes source addresses and targets.

Colors are identified using color descriptors such as `epobc:<txid>:0:0`, an empty string "" stands for uncolored bitcoins.

`source_addresses` is an associative array providing a list of addresses for each color, e.g.

     "source_addresses": {
        "": ["mpBEGSTSuNeGtKiXqo3V4CocHx8vWSF3Mo"],
        "epobc:b95323a763fa507110a89ab857af8e949810cf1e67e91104cd64222a04ccd0bb:0:180679": 
             ["mz6WNJtK5UKGWP4L7Fp2wH25SbMzrxyM3k"]
      }

Indicates that uncolored coins should be taken from address `mpBEGSTSuNeGtKiXqo3V4CocHx8vWSF3Mo`, while colored ones should come from `mz6WNJtK5UKGWP4L7Fp2wH25SbMzrxyM3k`.
Both P2KH (ordinary) and P2SH addresses are supported. (At least I hope so, never tried...)

`change_address` should provide a single address for each color used in a transaction. Change address needs to be provided only in case of a partial spend.

`targets` is an array of targets, that is, outputs transaction should have. Each target should include following fields:

name | description
-----|------------
color| color descriptor, use "" for uncolored bitcoins
value| number of atoms which should be sent (satoshis in case of uncolored coins)
address| receiver's address (if script is not provided)
script | (optional) output's scriptPubKey, can be used instead of address

`createIssueTx` and `createTransferTx` return a list of used coins for convenience (in addition to an hex-encoded unsigned transaction).

**WARNING**: Current version of service assumes that colored and uncolored coins are kept in different addresses. Addresses used for colored coins should never be used as sources of uncolored coins. This limitation will be removed once we switch to coloredcoinjs-lib v4.

### createIssueTx

`POST /api/createIssueTx`

Creates an unsigned issue transaction.

Necessary fields:

name | description
-----|------------
source_addresses | bitcoin source
change_address | address used for a change if coins are not spent fully
target | should include "address" (or "script") and "value"
color_kernel | should be "epobc"

[Sample input](https://github.com/chromaway/cc-api-service/blob/master/api_samples/createIssueTx.json).

Sample output  (shortened):

     {"tx":"01000.....ac00000000",
      "input_coins":[{
        "txId":"4ce9c88ac9efe6a8552d583af80d9473c88a3f74ae48f19a61719facf8ce3a43",
        "outIndex":1,
        "value":1639175,
        "script":"76a9145efe254d5f7ba5fafdcdba1b5cc1d4a0887279b088ac",
        "address":"mpBEGSTSuNeGtKiXqo3V4CocHx8vWSF3Mo"
     }]}
     
`input_coins` has information useful for signing the transaction (script is the script of corresponding output).

### createTransferTx

`POST /api/createTransferTx`

Creates an unsigned transfer transaction.

Necessary fields:

name | description
-----|------------
source_addresses | source for bitcoins and colored coins
change_address | address used for a change if coins are not spent fully
targets | see description in 'General conventions' section

[Sample input](https://github.com/chromaway/cc-api-service/blob/master/api_samples/createTransferTx.json).

Sample output: similar to output of `createIssueTx`.

Note: createTransferTx might take significant amount of time as service will compute coloring on demand. Make sure that HTTP client timeouts are high enough. Future versions of API will be faster due to pre-computing of coloring.

### broadcastTx

`POST /api/broadcastTx`

Broadcast a signed transaction.  Returns only when transaction is sent to bitcoind and indexed by chromanode. Might take up to 15 seconds (current chromanode limitations), time outs after 2 minutes.

Sample input (shortened): `{"tx":"01000.....ac00000000"}`
Sample output: `{"success": true}`

### getUnspentCoins

`POST /api/getUnspentCoins``

Can be used to obtain information about unspent coins for a specific address and color. Useful for computing current balance.

Parameters:

name | description
-----|------------
addresses | a list of addresses
color | color descriptor, "" for uncolored bitcoins

Note: When a proper color descriptor is provided, returns only coins of that color (TODO).
If "" is provided, returns _all_ coins, both colored and uncolored, and reports uncolored value.

This can be a problem if applications uses same address both for colored and uncolored coins.

Future versions will allow retrieving data for all colors at once, also correctly differentiate colored/uncolored coins. (Requires migration to coloredcoinjs-lib v4).

[Sample input](https://github.com/chromaway/cc-api-service/blob/master/api_samples/getUnspentCoins.json)

Sample output:

    {"coins":
       [{"txId":"749699eca1f0ec58d9cc770e52f1efc3bb690bbee84ea728c700f877c90f340f",
         "outIndex":1,
         "value":818000,
         "script":"76a914cbcac3ac056fbfddab68ff4c6cae976ad74e238d88ac",
         "color":"epobc:b95323a763fa507110a89ab857af8e949810cf1e67e91104cd64222a04ccd0bb:0:180679",
         "color_value":818000}]}
         
Note: "value" is bitcoin value of output, "color_value" is colorvalue for "color".

### getAllUnspentCoins

_Not implemented yet_. Should report all current unspent coins for a particular color (e.g. all asset owners). Currently this information can be obtained from [cc-scanner](https://github.com/chromaway/cc-scanner) which assembles it into a PostgreSQL database.
