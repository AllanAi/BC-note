Math in Solidity https://medium.com/coinmonks/math-in-solidity-part-1-numbers-384c8377f26d



## Sqrt

https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/libraries/Math.sol

```js
// babylonian method (https://en.wikipedia.org/wiki/Methods_of_computing_square_roots#Babylonian_method)
function sqrt(uint y) internal pure returns (uint z) {
    if (y > 3) {
        z = y;
        uint x = y / 2 + 1;
        while (x < z) {
            z = x;
            x = (y / x + x) / 2;
        }
    } else if (y != 0) {
        z = 1;
    }
}
```

# UniswapLib

https://github.com/Uniswap/uniswap-lib

TODO

