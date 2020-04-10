# KRC20 Token 对接指南

## 参考资料
* https://github.com/qtumproject/documents/blob/master/zh/%E6%89%8B%E5%8A%A8%E6%9E%84%E9%80%A0Qtum%E5%90%88%E7%BA%A6%E4%BA%A4%E6%98%93%E7%9A%84%E8%AF%B4%E6%98%8E.md
* https://docs.qtum.site/en/QRC20-integration.html

## 1. 全节点配置
运行kpgd的时候需要带上参数 `-logevents` 或者在`kpg.conf`中添加 `logevents=1`。  
如果之前运行过全节点，带上此参数首次运行的时候需要加上`--reindex`，此后不再需要。

## 2. 到账监测
对每一个新区块，调用RPC `searchlogs {height} {height}`
例如：
```
{
    "jsonrpc": "1.0", 
    "id":"curltest", 
    "method": "searchlogs", 
    "params": [18587, 18587]
}
```

得到返回结果如下：
```
[
  {
    "blockHash": "89e7ecbc8213389a582df61f58044db64e0f7d8ac9a8b574ce79b653f77c99a9",
    "blockNumber": 18587,
    "transactionHash": "ae273fdc7fc24040b7b0d80308525dca77c8c56a2e4965ea8ea6322cf9b46b3e",
    "transactionIndex": 2,
    "outputIndex": 0,
    "from": "7d90f1cbbb35afc72db58747588dc3cdf23c7c83",
    "to": "68048b799fe3e5c1484ba6fe3d000fdc3256579d",
    "cumulativeGasUsed": 36615,
    "gasUsed": 36615,
    "contractAddress": "68048b799fe3e5c1484ba6fe3d000fdc3256579d",
    "excepted": "None",
    "exceptedMessage": "",
    "stateRoot": "b78e248927a01519dbfbe6ac9d92d6b8524827cbe4c8b2b551c38fd87d3c635e",
    "utxoRoot": "56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
    "createdContracts": [
    ],
    "destructedContracts": [
    ],
    "log": [
      {
        "address": "68048b799fe3e5c1484ba6fe3d000fdc3256579d",
        "topics": [
          "ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",
          "0000000000000000000000007d90f1cbbb35afc72db58747588dc3cdf23c7c83",
          "000000000000000000000000793e1a00fd4f5de6e58331cc7ff555da0746286a"
        ],
        "data": "0000000000000000000000000000000000000000033b2e3c9fd0803ce8000000"
      }
    ]
  }
]
```

需要重点关注`log`字段的内容，其中`address`代表合约地址，`topics`代表eventlog，`data`表示数值。  

根据KRC20的协议标准，转账对应的Event为 `event Transfer(address indexed _from, address indexed _to, uint256 _value);`, 通过 eth-abi 对其编码后结果为`ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`。

```
{
"address": "68048b799fe3e5c1484ba6fe3d000fdc3256579d",
"topics": [
    "ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",
    "0000000000000000000000007d90f1cbbb35afc72db58747588dc3cdf23c7c83",
    "000000000000000000000000793e1a00fd4f5de6e58331cc7ff555da0746286a"
],
"data": "0000000000000000000000000000000000000000033b2e3c9fd0803ce8000000"
}
```
上述log中，合约地址为`68048b799fe3e5c1484ba6fe3d000fdc3256579d`，对应Token `UNP`,   
topics[0] == `ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef` 表明这是一笔 KRC20-Token转账交易，  
topics[1] == `0000000000000000000000007d90f1cbbb35afc72db58747588dc3cdf23c7c83` 表明转账发送方为`0000000000000000000000007d90f1cbbb35afc72db58747588dc3cdf23c7c83`, 通过调用RPC `fromhexaddress` 将其转化为普通地址格式，得到：`KJf5QndRfAGWZ7LNsTHk2BBgA3ZrYBaP1H`，  
同理 topics[2] == `000000000000000000000000793e1a00fd4f5de6e58331cc7ff555da0746286a` 表明交易接受方为`KJGDTzVJVMY74PAqiuuYdnXE5BTA1dSpS9`.  

我们将data的值由16进制转化为10进制，可以得到`0x0000000000000000000000000000000000000000033b2e3c9fd0803ce8000000 = 1000000000000000000000000000`,表明转账的金额为 1000000000000000000000000000 sat UNP，UNP的decimal为18，即 1000000000000000000000000000 / 10^18 = 1000000000 UNP  


除了通过 `searchlogs` 获取event 信息外，还可以通过 `gettransactionreceipt` 获取某个特定tx的event。

## 3. 查询余额
以`KQ754HU3wmdrCWHZuGXiJ1tPdNH4ow2YvY`为例，首先通过RPC `gethexaddress KQ754HU3wmdrCWHZuGXiJ1tPdNH4ow2YvY` 得到其16进制地址:`b95417e635df8daa8d5d87025f63d0112c2e8501`, 然后对左侧补0使字符串长度为64 得到`000000000000000000000000b95417e635df8daa8d5d87025f63d0112c2e8501`  


查询余额的合约函数为 `function balanceOf(address _owner) public view returns (uint256 balance)`，通过eth-abi对其编码得到`70a08231`,    

将上述两组数据进行拼接得到`70a08231000000000000000000000000b95417e635df8daa8d5d87025f63d0112c2e8501`

最后调用RPC `callcontract 68048b799fe3e5c1484ba6fe3d000fdc3256579d 70a08231000000000000000000000000b95417e635df8daa8d5d87025f63d0112c2e8501`
得到结果如下：
```
{
  "address": "68048b799fe3e5c1484ba6fe3d000fdc3256579d",
  "executionResult": {
    "gasUsed": 23314,
    "excepted": "None",
    "newAddress": "68048b799fe3e5c1484ba6fe3d000fdc3256579d",
    "output": "0000000000000000000000000000000000000000033b2e3c9fd0803ce8000000",
    "codeDeposit": 0,
    "gasRefunded": 0,
    "depositSize": 0,
    "gasForDeposit": 0,
    "exceptedMessage": ""
  },
  "transactionReceipt": {
    "stateRoot": "3faed9fe1736cc2b5be4bf8363377795a1cd8e0dbb049fe56e32fdc737019be5",
    "utxoRoot": "56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421",
    "gasUsed": 23314,
    "bloom": "00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
    "createdContracts": [
    ],
    "destructedContracts": [
    ],
    "log": [
    ]
  }
}
```

其中`executionResult`中的`output`就是我们的返回结果，`0x0000000000000000000000000000000000000000033b2e3c9fd0803ce8000000 = 1000000000000000000000000000`

## 4. 创建一笔Token转账交易
合约交易是在output中的`ScriptPubKey`里实现的，
转账调用的合约函数为 `function transfer(address _to, uint256 _value) returns (bool success)`，通过eth-abi对其编码得到 `a9059cbb`

例如:
```
01040390d003012844a9059cbb00000000000000000000000081f9fc3ee3667397b58f3a53c60e7556e98cf595000000000000000000000000000000000000000000000000000009184e72a00014e21bc819674c8f7cc7d76b618914ecff082107b3c2
```

* `0104`代表EVM版本，固定不变；  
* `0390d003`代表gas_limit，即最多消耗多少gas，`0x03d090 => 250000`, 这个通常也不需要改变；  
* `0128`代表gas_price，即gas的单价，`0x28 => 40`, 这个通常也不需要改变；  
* `44a9059cbb00000000000000000000000081f9fc3ee3667397b58f3a53c60e7556e98cf595000000000000000000000000000000000000000000000000000009184e72a000` 是调用合约的参数，`44`表示总长度, `a9059cbb` 表示 `transfer` 函数， `00000000000000000000000081f9fc3ee3667397b58f3a53c60e7556e98cf595` 表示接收方地址，`000000000000000000000000000000000000000000000000000009184e72a000`表示发送数额，转化为10进制为`10000000000000`
* `14e21bc819674c8f7cc7d76b618914ecff082107b3` 表示合约地址
* `c2`代表 OP_CALL, 表明这是一笔调用合约的交易。 

了解了上述规则之后，就可以自己生成`ScriptPubKey`了，同时需要注意的是，在计算手续费大小的时候，需要将gas包含进去。

## 5. 其他注意事项
1. 一般而言只有P2PKH类型的地址可以接收与发送Token。
2. 需要确保 sender 对应地址的 utxo 在 vin 中排首位。
3. 未用完的 gas 会在所在区块的 coinstake 交易中返还。


## 6. 使用 RPC 发送 KRC Token
在第4节中我们介绍了通过代码构建Token交易的步骤，这一节我们介绍一下通过RPC发送Token的方式。

调用RPC：`sendtocontract {TOKEN_CONTRACT_ADDRESS} a9059cbb{to32bytesArg(addressToHash160(RECEIVER_ADDRESS))}{to32bytesArg(addDecimals($amount))} 0 250000 4 {SENDER_ADDRESS}`

参数说明：
* `{TOKEN_CONTRACT_ADDRESS}`: Token的合约地址
* `{to32bytesArg(addressToHash160(RECEIVER_ADDRESS))}`: 将接收方地址转化为hash160地址，并在左侧补0，使字符串长度为64
* `{to32bytesArg(addDecimals($amount))}`: 将要发送的金额数目转化为16进制，并在左侧补0，使字符串长度为64
* `0 250000 4`：分别代表给合约发送KPG金额、gas_limit、gas_price，建议不要改动
* `{SENDER_ADDRESS}`: 发送方地址(钱包内需要有该地址的私钥)

参考文献：https://github.com/qtumproject/documents/blob/master/zh/QRC20%E9%9B%86%E6%88%90%E6%96%87%E6%A1%A3.md

