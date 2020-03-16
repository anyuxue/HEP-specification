# 网站DApp网站开发指南
网站DApp提供Python、PHP、Java编程语言SDK。根据开发需要，选择不同版本的SDK。

## 安装SDK

### python SDK
```bash
pip install hep-rest-api
```

### PHP

[PHP Tutorial](tutorial\tutorial-dweb-zh-php.md)

### Java
```bash
TBD
```

## 步骤

### 生成DApp公私钥对

### 创建DApp
TBD

### 基础参数配置

* Python
```python
import sys

# 配置基础参数
HEP_HOST = "https://hep.testnet.newtonproject.org" # testnet url
DAPP_ID = "{dapp_id}"
DAPP_KEY = "{dapp_key}"
DAPP_SECERT = "{dapp_secret}"
DAPP_PRIVATE_KEY_PATH = "{dapp_private_key_path}"
DAPP_PROTOCOL = "HEP"
DAPP_PROTOCOL_VERSION = "1.0"
CHAIN_ID = 1012  # dev environment is 1002, test environment is 1002, mainnet environment is 1012

# ACTION 参数
ACTION_LOGIN = "hep.auth.login"
ACTION_PAY = "hep.pay.order"
ACTION_PROOF_SUBMIT = "hep.proof.submit"

# 基础配置信息
base_parameters = {
    'dapp_key': DAPP_KEY,
    'protocol': DAPP_PROTOCOL,
    'version': DAPP_PROTOCOL_VERSION,
    'os': sys.platform,
    'language': 'en' # Your language
}

## 创建api client
configuration = hep_rest_api.api_client.Configuration()
configuration.host = HEP_HOST
api_client = hep_rest_api.RestApi(hep_rest_api.ApiClient(configuration))
```

* javascript 基础定义
```javascript
https://github.com/newtonproject/HEP-specification/blob/master/hep-provider/user-agent.md
```


### NewID授权

#### 初始化授权Helper
* python
```python
auth_helper = AuthHelper(_get_api_client(), base_parameters, DAPP_ID, DAPP_SECERT, DAPP_PRIVATE_KEY_PATH, chain_id=CHAIN_ID)
```

#### 生成授权签名
* python
```python
session_id = '{session_id}': # 会话标识
auth_response = auth_helper.generate_auth_request(uuid=session_id)
```

#### PC网站生成QRCode
* python
```python
qr_code_str = auth_helper.generate_qrcode_string(auth_response.auth_hash)
```

#### 移动网站调用内置Javascript函数，<a href="https://github.com/newtonproject/HEP-specification/blob/master/interact-specification/DWeb-callback-error-messages.md#example">回调的数据结构</a>

* 获取登录参数，并且传递参数到 NewPay, 获取 profile 信息
```javascript
function h5login() {
    let url = "/request/login/h5/";
    $.ajax({
        url: url,
        async: true,
        type: 'post',
        success: function (res) {
            if(res.error_code === 1) {
                let params = res.result;
                if(hep) {   // 确保 newpay 注入成功，由 NewPay 完成
                    hep.auth.login(params, function (response) {
                        if(response.status_code === 200) {
                            var profile = response.result;   // 成功的结果数据
                            let url = "/post/profile/";
                            $.post(url, profile, function (res) {
                                if(res.error_code === 1) {
                                    window.location.href = "/user"
                                }
                            }, "json")
                        } else {
                            alert(response.message);  // 错误信息
                        }
                    });
                }else {
                    alert("hep is not inject");
                }
            }
        }
    });
}
```

#### 使用NewPay完成授权，将授权数据发送到网站回调接口
callback的接受参数格式见 https://github.com/newtonproject/HEP-specification/blob/master/interact-specification/DWeb-NewPay.md#newpay-return-to-dweb-app-or-server


#### 网站验证用户返回授权数据
* python
```python
# 如果返回为 True， 那么用户的信息已确认
verify_status = auth_helper.validate_auth_callback(data)


from hep_rest_api.scenarios.auth import AuthHelper
from hep_rest_api.scenarios.pay import PayHelper
from hep_rest_api.scenarios.proof import ProofHelper
```
#### 接收成功 返回 http code 为 200，其他 code 为失败

## 支付

#### 获取支付的 PayHelper

```python
def _get_pay_helper():
    if pay_helper:
        return pay_helper
    return PayHelper(_get_api_client(), base_parameters, DAPP_ID, DAPP_SECERT, DAPP_PRIVATE_KEY_PATH, chain_id=CHAIN_ID)
```

#### 生成支付签名
* python
```python
data = {'orderInformation'}
pay_response = pay_helper.generate_pay_request(data)
```

#### PC网站生成QRCode
* python
```python
qr_code_str = pay_helper.generate_qrcode_string(pay_helper.pay_hash)
```

#### 移动网站调用内置Javascript函数

* 获取支付参数，并且传递支付参数到 NewPay, 获取支付结果信息
```javascript
function h5pay() {
    let url = "/request/pay/h5/";
    $.ajax({
        url: url,
        async: true,
        type: 'post',
        data: {'order_number': 'orderNumber'},
        success: function (res) {
            if(res.error_code === 1) {
                if(hep) {
                    let params = res.result;
                    hep.pay.order(params, function (response) {
                        if(response.status_code === 200) {
                            var pay_info = response.result;
                            let url = "/receive/pay/";
                            $.ajax({
                                url: url,
                                async: true,
                                type: 'post',
                                data: pay_info,
                                success: function (res) {
                                    if(res.error_code == 1) {
                                        window.location.href = "/placeorder/"
                                    }
                                }
                            });
                        } else {
                            alert(response.message);
                        }
                    })
                } else {
                    alert("hep is not inject");
                }
            }
        }
    });
}

```

#### 使用NewPay完成支付，将支付数据发送到网站回调接口
callback的接受参数格式见 https://github.com/newtonproject/HEP-specification/blob/master/interact-specification/DWeb-NewPay.md#return-transaction-information-to-dapp-js-or-server

#### 网站验证用户返回支付数据
```python
def verify_pay(params):
    # 验证签名是否通过
    is_valid = _get_pay_helper().validate_pay_callback(params)
    if is_valid:
        # 根据txid获取确认的订单信息，返回结果包括 订单唯一编号，支付接受地址，支付发送方地址，支付金额等
        response = _get_pay_helper().get_confirmed_transaction(params.get('txid'))
        # 参见: https://github.com/newtonproject/HEP-specification/blob/master/hep-protocol/README.md#return-transaction-information-to-dapp
        return response
    return None
```
#### 接收成功 返回 http code 为 200，其他 code 均为失败

## 上链

#### 获取上链的 ProofHelper

```python
def _get_proof_helper():
    if proof_helper:
        return proof_helper
    return ProofHelper(_get_api_client(), base_parameters, DAPP_ID, DAPP_SECERT, DAPP_PRIVATE_KEY_PATH, chain_id=CHAIN_ID)
```

#### 生成上链签名
* python
```python
content = {'proofinformation'}
uuid = {'session_id'}
proof_response = proof_helper.generate_proof_request(content.to_dict(), uuid)
```

#### PC网站生成QRCode
* python
```python
qr_code_str = proof_helper.generate_qrcode_string(proof_response.proof_hash)
```

#### 移动网站调用内置Javascript函数

* 获取上链的参数，并且传递上链参数到 NewPay
```javascript
function h5proof() {
    let url = "/request/proof/h5/";
    $.ajax({
        url: url,
        async: true,
        type: 'post',
        data: {'order_number': 'orderNumber'},
        success: function (res) {
            if(res.error_code === 1) {
                if(hep) {
                    let params = res.result;
                    hep.proof.submit(params, function(response) {
                        if(response.status_code === 200) {
                            var proof_info = response.result;
                            let url = "/receive/proof/";
                            $.ajax({
                                url: url,
                                async: true,
                                type: 'post',
                                data: proof_info,
                                success: function (res) {
                                    $('#tip').val("success")
                                }
                            })
                        } else {
                            alert(response.message); 
                        }
                    });
                } else {
                    alert("hep is not inject");
                }
            }
        }
    });
}

```

#### 使用NewPay完成上链，将支付数据发送到网站回调接口
callback的接受参数格式见 https://github.com/newtonproject/HEP-specification/blob/master/interact-specification/DWeb-NewPay.md#result-to-dweb-app-or-server

#### 网站验证用户返回上链数据
```python
def verify_proof(data):
    # 验证上链信息的签名
    is_valid = _get_proof_helper().validate_proof_callback(data)
    if is_valid:
        # 可以批量验证 proof_hash 的状态
        proof_hashes = [data.get('proof_hash')]
        response = _get_proof_helper().get_status_of_proofs(proof_hashes)
        # 如果只校验一条的话取第一个数组的值，返回结果包括 proof_hash, 和 proof_status
        # proof_status 为字符串类型 SUBMIT(已提交), PROCESSING(处理中),CONFIRMED(已上链), CANCELED(已取消), PART_CANCELED(部分取消/退货)
        # 参见 https://github.com/newtonproject/HEP-specification/blob/master/hep-node/REST-API.md#proof_getproofreceipts
        return response.receipts[0]
    return None
```

#### 接收成功 返回 http code 为 200，其他 code 均为失败

