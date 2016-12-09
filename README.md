# TinyOracleHttp

##Использование

Как эта штука вообще работает. На вашей хостовой машине, где размернут etherium должен быть установлен nodejs.
например это написано здесь: https://nodejs.org/en/download/package-manager/

Код который дергает api написан здесь https://github.com/Robert-SZ/tinyoracleHttp/blob/master/tinyoracleHttp
Там есть метод callHttp, если вам нужно просто дернуть api то его можно не менять.
Этот метод дергает заданный url методом GET

###Установка пакета tinyoracleHttp
После установки nodejs сделайте
npm -i g tinyoracleHttp

Пакет будет установлен.

###1. Компиляция скриптов через Browser Solidity https://ethereum.github.io/browser-solidity/

Версия solidity version=soljson-v0.4.0+

Создать в browser solidity 4 файла:

1. api.sol
В этой файле будет заменить адрес  lookupContract = 0xc031180519370e8b0f5f585e681fabfdb26cec72- это описано в разделе step by step
```js
//
// This is the API file to be included by a user of this oracle
//
pragma solidity ^0.4.0;

// This must match the signature in dispatch.sol
contract TinyOracle {
  function query(bytes _query) returns (uint256 id);
}

// This must match the signature in lookup.sol
contract TinyOracleLookup {
  function getQueryAddress() constant returns (address);
  function getResponseAddress() constant returns (address);
}

// The actual part to be included in a client contract
contract usingTinyOracle {
  address constant lookupContract = 0xc031180519370e8b0f5f585e681fabfdb26cec72;

  modifier onlyFromTinyOracle {
    TinyOracleLookup lookup = TinyOracleLookup(lookupContract);
    if (msg.sender != lookup.getResponseAddress())
      throw;
    _;
  }

  function queryTinyOracle(bytes query) internal returns (uint256 id) {
    TinyOracleLookup lookup = TinyOracleLookup(lookupContract);
    TinyOracle tinyOracle = TinyOracle(lookup.getQueryAddress());
    return tinyOracle.query(query);
  }
}
```
2. dispatch.sol
```js
//
// This is where the magic happens
//
// This contract will receive the actual query from the caller
// contract. Assign a unique (well, sort of) identifier to each
// incoming request, and emit an event our RPC client is listening
// for.
//
pragma solidity ^0.4.0;

contract TinyOracleDispatch {
  event Incoming(uint256 id, address recipient, bytes query);

  function query(bytes _query) external returns (uint256 id) {
    id = uint256(sha3(block.number, now, _query, msg.sender));
    Incoming(id, msg.sender, _query);
  }

  // The basic housekeeping

  address owner;

  modifier owneronly { if (msg.sender == owner) _; }

  function setOwner(address _owner) owneronly {
    owner = _owner;
  }

  function TinyOracleDispatch() {
    owner = msg.sender;
  }

  function transfer(uint value) owneronly {
    transfer(msg.sender, value);
  }

  function transfer(address _to, uint value) owneronly {
    _to.send(value);
  }

  function kill() owneronly {
    suicide(msg.sender);
  }
}
```

3. lookup.sol
```js
//
// The lookup contract for storing both the query and responder addresses
//
pragma solidity ^0.4.0;
contract TinyOracleLookup {
  address owner;
  address query;
  address response;

  modifier owneronly { if (msg.sender == owner) _; }

  function setOwner(address _owner) owneronly {
    owner = _owner;
  }

  function TinyOracleLookup() {
    owner = msg.sender;
  }

  function setQueryAddress(address addr) owneronly {
    query = addr;
  }

  function getQueryAddress() constant returns (address) {
    return query;
  }

  function setResponseAddress(address addr) owneronly {
    response = addr;
  }

  function getResponseAddress() constant returns (address) {
    return response;
  }

  function kill() owneronly {
    suicide(msg.sender);
  }
}
```

4. sample.sol

Этот файл вам нужен только для того чтобы принять запрос определенного контракта с нектороми параметрами в данном случае параметр 1- url

```js
import "api.sol";

contract SampleHttpClient is usingTinyOracle {
  bytes public response;

  function __tinyOracleCallback(uint256 id, bytes _response) onlyFromTinyOracle external {
    response = _response;
  }

  function query(string url) {
    string memory tmp = url;
    query(bytes(tmp));
  }

  function query(bytes query) {
    queryTinyOracle(query);
  }
}
```





## Step by step

1. Have ```geth``` fully set up, including an account with ethers. (Shorthand in the following sections for this account is *sender*.)

2. Start the RPC server and unlock the account *sender* used for sending responses:
```
geth --rpc --rpcaddr "127.0.0.1" --rpcport "8545" --unlock 0
```
For the account list use:
```
geth account list
```

3. Deploy ```dispatch.sol```. Take note of its address (shorthand is *dispatch*).

4. Deploy ```lookup.sol```. Take note of its address (shorthand is *lookup*). Call ```setQueryAddress()``` with *dispatch*, and ```setResponseAddress()``` with *sender*.

5. Edit ```api.sol```: replace the address of ```lookupContact``` with the value of *lookup*.

6. Run the RPC listener & dispatch code:
```
tinyoracle --rpcport 8545 --rpchost 127.0.0.1 --interval 10 --dispatch <dispatch> --sender <sender>
```
Interval above is the frequency of polling for incoming requests, where 10 means every 10 seconds. Replace all the parameters with the ones set up earlier.

7. Deploy ```sample-client.sol```. Transact with ```query()``` and call ```getResponse()``` to verify the response was received.

8. Next steps: further improve TinyOracle and send a pull request.

## Important notes

As you are sending responses back as a transaction, which costs ether, it would make sense to charge the clients a fee.

**This code is not intended for use in production.** Any failure to process the event can mean it is lost forever and a response will never be sent. It is suggested to store the received events in a database and process the responses in a separate thread or application.

Offering data through this toolkit to the public will not make it a trusted source. Your user can trust it as long as it trusts you and that the lookup contract is controlled by you.

## Future improvements

1. Received events should be stored in a queue or database and processed asynchronously.

2. There should be a storage for requests in the dispatcher, so that in case of RPC errors, events can still be recalled. This has a cost on storage.

## Under the hood

TinyOracle is running on **Node.js** and uses [json-rpc2](https://github.com/pocesar/node-jsonrpc2) to communicate with the Ethereum RPC endpoint. Data is decoded and encoded using [ethereumjs-abi](https://github.com/axic/ethereumjs-abi).

## Acknowledgements

Thanks goes to [Oraclize.it](http://www.oraclize.it/home/features) who run an actual useful oracle service available today on Ethereum and have inspired this guide.

## License

    Copyright (C) 2015 Alex Beregszaszi

    Permission is hereby granted, free of charge, to any person obtaining a copy of
    this software and associated documentation files (the "Software"), to deal in
    the Software without restriction, including without limitation the rights to
    use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
    the Software, and to permit persons to whom the Software is furnished to do so,
    subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
    FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
    COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
    IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
    CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
