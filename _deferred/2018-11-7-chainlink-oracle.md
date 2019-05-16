---
layout: post
title: ChainLink预言机过程梳理
tags: solidity
categories: solidity
---
* TOC
{:toc}

#概述
![chainlink oracle flow](https://github.com/suimi/suimi.github.io/blob/master/static/img/ChainLink-Oracle-flow.png?raw=true)
#预言机调用过程
![chainlink oracle steps](https://github.com/suimi/suimi.github.io/blob/master/static/img/chainlink-oracle-steps.png?raw=true)
## 智能合约调用预言机 
### 合约调用
以`RopstenConsumer.sol.requestEthereumPrice()`为例，newRun初始化run时会把回调函数签名赋到callbackFunctionId
```
Run.initialize
self.callbackFunctionId = bytes4(keccak256(bytes(_callbackFunctionSignature)));
```
### `chainlinkRequest`生成requestId，并调用`link.transferAndCall`
### `link.transferAndCall(oracle, _amount, _run.encodeForOracle(clArgsVersion))`
`_run.encodeForOracle(clArgsVersion)`会把`requestData`函数签名带上
```
bytes4 internal constant oracleRequestDataFid = bytes4(keccak256("requestData(address,uint256,uint256,bytes32,address,bytes4,bytes32,bytes)"));
function encodeForOracle(
        Run memory self,
        uint256 _clArgsVersion
    ) internal pure returns (bytes memory) {
        return abi.encodeWithSelector(
            oracleRequestDataFid,
```
### `LinkToken.sol.transferAndCall`
### `ERC677Token.sol.contractFallback`
### `Oracle.sol.onTokenTransfer`
```
//调用预言机requestData方法
address(this).delegatecall(_data)
```
### `Oracle.sol.requestData`
存储回调，发布RunRequest事件

## 事件订阅
`subscription.go`负责事件订阅
```
var RunLogTopic = mustHash("RunRequest(bytes32,address,uint256,uint256,uint256,bytes)")
```
接收事件并执行
```
func receiveRunLog(le InitiatorSubscriptionLogEvent) {
	if !le.ValidateRunLog() {
		return
	}

	le.ToDebug()
	//会将执行结果回调的合约地址及函数签名附上
	data, err := le.RunLogJSON()
	if err != nil {
		logger.Errorw(err.Error(), le.ForLogger()...)
		return
	}

	runJob(le, data, le.Initiator)
}
```
```
func fulfillmentToJSON(el strpkg.Log) (models.JSON, error) {
	var js models.JSON
	js, err := js.Add("address", el.Address.String())
	if err != nil {
		return js, err
	}

	js, err = js.Add("dataPrefix", encodeRequestID(el.Data))
	if err != nil {
		return js, err
	}

	return js.Add("functionSelector", OracleFulfillmentFunctionID)
}
```
```
// OracleFulfillmentFunctionID is the function id of the oracle fulfillment
// method used by EthTx: bytes4(keccak256("fulfillData(uint256,bytes32)"))
// Kept in sync with solidity/contracts/Oracle.sol
//fulfillData(uint256,bytes32) 函数签名
const OracleFulfillmentFunctionID = "0x76005c26"
```

## 事件执行过程
### 执行过程
```
receiveRunLog -> runJob -> ExecuteJob -> saveAndTrigger -> store.RunChannel.Send -> store.RunChannel.Receive() ->job_runner.go.channelForRun -> workerLoop -> executeTask
```
### 执行完成后回调
`executeTask`中执行完成时会调用`adapter.Perform(input, store)`，调用预言机fulfillData方法

```
EthTx.Perform
func (etx *EthTx) Perform(input models.RunResult, store *store.Store) models.RunResult {
	if !input.Status.PendingConfirmations() {
		return createTxRunResult(etx, input, store)
	}
	return ensureTxRunResult(input, store)
}
```
```
func createTxRunResult(
	e *EthTx,
	input models.RunResult,
	store *store.Store,
) models.RunResult {
	val, err := input.Value()
	if err != nil {
		return input.WithError(err)
	}
    //获取回调函数签名及dataPrefix
    //FunctionSelector即OracleFulfillmentFunctionID，也就是合约fulfillData(uint256,bytes32)
	data, err := utils.HexToBytes(e.FunctionSelector.String(), e.DataPrefix.String(), val)
	if err != nil {
		return input.WithError(err)
	}

    //合约地址
	tx, err := store.TxManager.CreateTx(e.Address, data)
	if err != nil {
		return input.WithError(err)
	}

	sendResult := input.WithValue(tx.Hash.String())
	return ensureTxRunResult(sendResult, store)
}
```

## 预言机回调合约
```
  function fulfillData(
    uint256 _internalId,
    bytes32 _data
  )
    public
    onlyOwner
    hasInternalId(_internalId)
    returns (bool)
  {
    Callback memory callback = callbacks[_internalId];
    withdrawableWei = withdrawableWei.add(callback.amount);
    delete callbacks[_internalId];
    // All updates to the oracle's fulfillment should come before calling the
    // callback(addr+functionId) as it is untrusted.
    // See: https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern
    return callback.addr.call(callback.functionId, callback.externalId, _data); // solium-disable-line security/no-low-level-calls
  }
```