# Sync data by polling

The most primitive way to keep your dapp in sync with the contract state is by polling

## props
- No need to connect to websocket provider
- Dapp logic might be simpler

## const
- Not that reactive (depends on how long your polling interval is)
- If a contract variable rarely change then you might waste a lot of time polling it



It's still worth taking a look on how to amke a dapp polling the contract state

This is the `Hello.sol`:

```js
contract Hello {
    uint256 public value;

    function update() public {
        value += 1;
    }

}
```

There's only 1 public variable called `value` and we are going to sync with that. Parsing the contract `ABI` would be something worth doing

## Parsing ABI

As decribed in previous chapters. `ABI` is included inside of the JSON artifacts generated by dexon-truffle (`npm run migrate:development`)

```js
const contractInfo = (await import('../build/contracts/Hello.json')).default;
const { abi, networks } = contractInfo;
```

If we print out the ABI of `Hello.sol` you'll get something like:

```js
[
  {
    constant: true
    inputs: []
    name: "value"
    outputs: [
      { name: "", type: "uint256"}
    ],
    payable: false
    signature: "0x3fa4f245"
    stateMutability: "view"
    type: "function"
  },
  {
    constant: false
    inputs: []
    name: "update"
    outputs: []
    payable: false
    signature: "0xa2e62045"
    stateMutability: "nonpayable"
    type: "function"
  }
]
```

It gives us enough information about the public information we can get from the contract. In this example we can simply parse the abi:
```js
  // Get the list of all views
  const variableToPoll = abi.filter((item) => {
    return item.stateMutability === "view";
  });

      // This will be our object which data is in sync with contract
  const contractData = {};
  
  setInterval(async () => {
    const getEverything = variableToPoll.map((item) => {
      return (helloContract.methods[item.name]().call())
        .then((res) => {
          contractData[item.name] = res;
        });
    });

    // wait until all the response is returned
    await Promise.all(getEverything);
    console.log(contractData);
  }, 3000);
```

If you open up the developer console, you will see the `contractData` is being printed out every 3 seconds. Click `update value` and you'll notice the change to the `contractData`
```js
{value: "0"}
{value: "0"}
{value: "0"}
// ...keep coming up every 3 seconds
```

add a new public variable `value2` in `Hello.sol` and `value2` gets updated when `update` is called
```js
contract Hello {
    uint256 public value;
    uint256 public value2;

    function update() public {
        value += 1;
        value2 += 2;
    }
}
```

After `npm run migrate:development` and `npm run build:webapp`, let's refresh the browser again and we'll see:
```js
{value: "0", value2: "0"}
{value: "0", value2: "0"}
{value: "0", value2: "0"}
// ...keep coming up every 3 seconds
```

If we click `Update value` and within 3 seconds the output will be:
```js
{value: "1", value2: "2"}
{value: "1", value2: "2"}
{value: "1", value2: "2"}
// ...keep comming up every 3 seconds
```

Now we know that our data is in sync and even when the contract is changed, we don't need to modify anything but still see the new variable in dapp.

Of course real world cases won't be this simple (for example, we might have `view function` which takes parameters) but at least it gives us an idea that we are able to leverage what's in `ABI` and write less code. We should bear this in mind when designing our dapp architecture.