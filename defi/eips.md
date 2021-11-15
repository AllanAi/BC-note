## EIP 55: 混合大小写校验和地址编码

优点:

- 向后兼容许多接受混合大小写的十六进制解析器，允许它随着时间的推移轻松引入
- 保持长度为40个字符
- 平均每个地址将有15个校验位，如果输入错误，随机生成的地因抄写错误意外通过检查的净概率为0.0247％。 这比ICAP提高了约50倍，但不如4字节检查代码好。

> ICAP: Inter-exchange Client Address Protocal 是与国际银行账号（IBAN）编码部分兼容的以太坊地址编码形式。

```js
const createKeccakHash = require('keccak')

function toChecksumAddress (address) {
  address = address.toLowerCase().replace('0x', '')
  var hash = createKeccakHash('keccak256').update(address).digest('hex')
  var ret = '0x'

  for (var i = 0; i < address.length; i++) {
    if (parseInt(hash[i], 16) >= 8) {
      ret += address[i].toUpperCase()
    } else {
      ret += address[i]
    }
  }

  return ret
}
```

```text
> toChecksumAddress('0xfb6916095ca1df60bb79ce92ce3ea74c37c5d359')
'0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359'
```



