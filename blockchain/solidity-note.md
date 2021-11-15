## abi.encode / abi.encodePacked / call

- `abi.encode` encode its parameters using the [ABI specs](https://docs.soliditylang.org/en/v0.8.0/abi-spec.html). The ABI was designed to make calls to contracts. Parameters are padded to 32 bytes. If you are making calls to a contract you likely have to use `abi.encode`
- `abi.encodePacked` encode its parameters using the minimal space required by the type. Encoding an uint8 it will use 1 byte. It is used when you want to save some space, and **not** calling a contract.
- `abi.encodeWithSignature` same as encode but with the function signature as first parameter. Use when the signature is known and don't want to calculate the selector.
- `abi.encodeWithSelector` same as encode but selector is the first parameter. It almost equal to encodeWithSignature, use whatever fits best.

https://ethereum.stackexchange.com/questions/91826/why-are-there-two-methods-encoding-arguments-abi-encode-and-abi-encodepacked

```sql
(success, ) = address(c).call(abi.encodeWithSignature("myfunction(uint256,uint256)", 400,500));
contract_instance.myfunction(400,500);
```

The second one is more expensive and safer than other cases:

> Due to the fact that the EVM considers a call to a non-existing contract to always succeed, Solidity includes an extra check using the "extcodesize" opcode when performing external calls. This ensures that the contract that is about to be called either actually exists (it contains code) or an exception is raised.

> The low-level calls which operate on addresses rather than contract instances (i.e. .call(), .delegatecall(), .staticcall(), .send() and .transfer()) **do not include** this check, which makes them **cheaper** in terms of gas but also **less safe**.

