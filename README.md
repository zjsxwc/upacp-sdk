# upacp-sdk

银联官网下载的sdk我稍微改了下，用法很简单


#### 第零步，引入sdk代码

######## 直接引入

就是直接代码里 `include_once` 我们的 `acp_service.php` 文件

######## 使用composer的path引入本地目录方式

就是在你的主项目的`composer.json`文件的repositories字段里添加本地目录作为一个仓库，比如我的`composer.json`:

```
    "repositories": {
        "upacpsdk": {
            "type": "path",
            "url": "local3rdpackages/upacp-sdk",//填sdk库目录对于主项目composer.json文件的相对路径；比如这里，也就是要保证sdk的composer.json文件就在local3rdpackages/upacp-sdk目录下，主项目的composer.json与目录local3rdpackages同一级别
            "options": {
                "symlink": false
            }
        },
        "packagist": {
            "type": "composer",
            "url": "https://packagist.phpcomposer.com"
        }
    }
```



#### 第一步，参考`acp_sdk.ini`文件，创建一个我们自己的ini配置文件


#### 第二步，用户浏览器发起支付

```php

use com\unionpay\acp\sdk\AcpService;
use com\unionpay\acp\sdk\LogUtil;
use com\unionpay\acp\sdk\PhpLog;
use com\unionpay\acp\sdk\SDKConfig;

    /**
     * 发起支付
     * @param UserOrder $userOrder
     * @param string $returnUrl 用户在银联支付成功后，银联回跳到我们哪个地址
     */
    private function startUpacpChannelPay($userOrder, $returnUrl)
    {
    	/** @var string $acpSdkConfigFilePath 我们第一步创建的ini配置文件地址 */
        $acpSdkConfigFilePath = $this->getParameter("upacp_sdk_config_path");
        SDKConfig::initSDKConfig($acpSdkConfigFilePath); //根据我们的ini文件，初始化银联sdk

        //notifyUrl用于银联通知我们支付成功，也就是回调地址
        $notifyUrl = $this->generateUrl("paymentCallback", ["channelName" => self::CHANNEL_NAME_UPACP], UrlGeneratorInterface::ABSOLUTE_URL);

        $paymentOrderId = $userOrder->getId() . "T" . time();//由于不能重复下单，原订单号后面追加时间戳
        /** @noinspection PhpUndefinedFieldInspection */
        $params = array(
            //以下信息非特殊情况不需要改动
            'version' => SDKConfig::getSDKConfig()->version,                 //版本号
            'encoding' => 'utf-8',                  //编码方式
            'txnType' => '01',                      //交易类型
            'txnSubType' => '01',                  //交易子类
            'bizType' => '000201',                  //业务类型
            'frontUrl' => $returnUrl,  //前台通知地址
            'backUrl' => $notifyUrl,      //后台通知地址
            'signMethod' => SDKConfig::getSDKConfig()->signMethod,                  //签名方法
            'channelType' => '08',                  //渠道类型，07-PC，08-手机
            'accessType' => '0',                  //接入类型
            'currencyCode' => '156',              //交易币种，境内商户固定156

            //TODO 以下信息需要填写
            'merId' => SDKConfig::getSDKConfig()->merId,        //商户代码，请改自己的测试商户号，此处默认取demo演示页面传递的参数
            'orderId' => $paymentOrderId,    //商户订单号，8-32位数字字母，不能含“-”或“_”，此处默认取demo演示页面传递的参数，可以自行定制规则
            'txnTime' => date('YmdHis'),    //订单发送时间，格式为YYYYMMDDhhmmss，取北京时间，此处默认取demo演示页面传递的参数
            'txnAmt' => strval(intval($userOrder->getPayAmount() * 100)),    //交易金额，单位分，此处默认取demo演示页面传递的参数

            // 订单超时时间。
            // 超过此时间后，除网银交易外，其他交易银联系统会拒绝受理，提示超时。 跳转银行网银交易如果超时后交易成功，会自动退款，大约5个工作日金额返还到持卡人账户。
            // 此时间建议取支付时的北京时间加15分钟。
            // 超过超时时间调查询接口应答origRespCode不是A6或者00的就可以判断为失败。
            'payTimeout' => date('YmdHis', strtotime('+15 minutes')),
        );

        AcpService::sign($params);
        /** @noinspection PhpUndefinedFieldInspection */
        $uri = SDKConfig::getSDKConfig()->frontTransUrl;
        $html = AcpService::createAutoFormHtml($params, $uri);
        echo $html;

        exit;
    }

```


#### 第三步，处理银联回调通知我们支付成功

```php

use com\unionpay\acp\sdk\AcpService;
use com\unionpay\acp\sdk\LogUtil;
use com\unionpay\acp\sdk\PhpLog;
use com\unionpay\acp\sdk\SDKConfig;

    /**
     * @throws \Doctrine\DBAL\ConnectionException
     * @throws \Exception
     */
    private function handleUpacpCallback()
    {
        $acpSdkConfigFilePath = $this->getParameter("upacp_sdk_config_path");
        SDKConfig::initSDKConfig($acpSdkConfigFilePath);

        /** @var PhpLog $logger */
        $logger = LogUtil::getLogger();
        /** @noinspection PhpParamsInspection */
        $logger->LogInfo("receive back notify: " . \com\unionpay\acp\sdk\createLinkString ( $_POST, false, true ));

        $isValid = AcpService::validate($_POST);
        if (!$isValid) {
            exit;
        }
        $paymentOrderId = $_POST ['orderId']; //其他字段也可用类似方式获取
        $respCode = $_POST ['respCode'];
        if (in_array($respCode, ["00", "A6"])) {
            $tryQueryTimes = 0;
            startQueryOrderStatus:
            /** @noinspection PhpUndefinedFieldInspection */
            $params = array(
                //以下信息非特殊情况不需要改动
                'version' => SDKConfig::getSDKConfig()->version,          //版本号
                'encoding' => 'utf-8',          //编码方式
                'signMethod' => SDKConfig::getSDKConfig()->signMethod,          //签名方法
                'txnType' => '00',              //交易类型
                'txnSubType' => '00',          //交易子类
                'bizType' => '000000',          //业务类型
                'accessType' => '0',          //接入类型
                'channelType' => '07',          //渠道类型

                //TODO 以下信息需要填写
                'orderId' => $paymentOrderId,    //请修改被查询的交易的订单号，8-32位数字字母，不能含“-”或“_”，此处默认取demo演示页面传递的参数
                'merId' => SDKConfig::getSDKConfig()->merId,        //商户代码，请改自己的测试商户号，此处默认取demo演示页面传递的参数
                'txnTime' => $_POST["txnTime"],    //请修改被查询的交易的订单发送时间，格式为YYYYMMDDhhmmss，此处默认取demo演示页面传递的参数
            );
            AcpService::sign($params);
            /** @noinspection PhpUndefinedFieldInspection */
            $url = SDKConfig::getSDKConfig()->singleQueryUrl;

            /** @noinspection PhpParamsInspection */
            $queryResultArr = AcpService::post($params, $url);
            if (count($queryResultArr) <= 0) { //没收到200应答的情况
                exit;
            }
            if (!AcpService::validate ($queryResultArr) ){
                exit;
            }
            $tryQueryTimes++;

            $isPaidSuccess = false;
            if ($queryResultArr["respCode"] == "00"){
                if ($queryResultArr["origRespCode"] == "00"){
                    $isPaidSuccess = true;
                } else {
                    if ($queryResultArr["origRespCode"] == "03"
                        || $queryResultArr["origRespCode"] == "04"
                        || $queryResultArr["origRespCode"] == "05"){
                        if ($tryQueryTimes < 5) {
                            sleep(1);
                            goto startQueryOrderStatus;
                        }
                    }
                }
            } else {
                if ($queryResultArr["respCode"] == "03"
                    || $queryResultArr["respCode"] == "04"
                    || $queryResultArr["respCode"] == "05" ){
                    if ($tryQueryTimes < 5) {
                        sleep(1);
                        goto startQueryOrderStatus;
                    }
                }
            }

            if ($isPaidSuccess) {
                //由于不能重复下单，原订单号后面追加时间戳，获取原订单号
                $userOrderIdSegments = explode("T", $paymentOrderId);
                $userOrderId = intval($userOrderIdSegments[0]);
                $userOrderRepo = $this->getDoctrine()->getRepository("OrderBundle:UserOrder");
                /** @var UserOrder $userOrder */
                $userOrder = $userOrderRepo->find($userOrderId);
                if (!$userOrder) {
                    exit;
                }

                $paidAmount = floatval(intval($queryResultArr["txnAmt"])/100.00);
                if ($paidAmount < floatval($userOrder->getPayAmount())) {
                    exit;
                }

                //到这一步说明支付成功了，就可以修改订单了

                $callbackParams = [
                    "CHANNEL" => self::CHANNEL_NAME_UPACP,
                    "POST" => $_POST,
                    "GET" => $_GET,
                    "PHP_INPUT" => @file_get_contents('php://input')
                ];
                $userOrder->setCallbackParams($callbackParams);
                $userOrder->setPaymentOrderId($paymentOrderId);

                $this->get("order.pay_success_handler")->handle($userOrder, UserOrder::PAID_TYPE_UPACP);
                echo "OK";
            }
        }
        exit;
    }



```


#### 处理退款

```php


        if ($userOrder->getPaidType() == UserOrder::PAID_TYPE_UPACP) {
            //处理银联支付退款
            $paymentOrderId = $userOrder->getPaymentOrderId();
            if (!$paymentOrderId) {
                throw new \RuntimeException("Item order cannot be refunded because upacp paymentOrderId lost");
            }

            SDKConfig::initSDKConfig($this->acpSdkConfigFilePath);

            $callbackParams = $userOrder->getCallbackParams();
            if (
                $callbackParams
                && is_array($callbackParams)
                && isset($callbackParams["POST"])
                && is_array($callbackParams["POST"])
                && $callbackParams["POST"]
            ) {
                $callbackPostParams = $callbackParams["POST"];
                /** @noinspection PhpUndefinedFieldInspection */
                $params = array(
                    //以下信息非特殊情况不需要改动
                    'version' => SDKConfig::getSDKConfig()->version,		      //版本号
                    'encoding' => 'utf-8',		      //编码方式
                    'signMethod' => SDKConfig::getSDKConfig()->signMethod,		      //签名方法
                    'txnType' => '04',		          //交易类型
                    'txnSubType' => '00',		      //交易子类
                    'bizType' => '000201',		      //业务类型
                    'accessType' => '0',		      //接入类型
                    'channelType' => '07',		      //渠道类型
                    'backUrl' => SDKConfig::getSDKConfig()->backUrl, //后台通知地址

                    //TODO 以下信息需要填写
                    'orderId' => $paymentOrderId,	    //商户订单号，8-32位数字字母，不能含“-”或“_”，可以自行定制规则，重新产生，不同于原消费，此处默认取demo演示页面传递的参数
                    'merId' => SDKConfig::getSDKConfig()->merId,	        //商户代码，请改成自己的测试商户号，此处默认取demo演示页面传递的参数
                    'origQryId' => strval($callbackPostParams["origQryId"]), //原消费的queryId，可以从查询接口或者通知接口中获取，此处默认取demo演示页面传递的参数
                    'txnTime' => strval($callbackPostParams["txnTime"]),	    //订单发送时间，格式为YYYYMMDDhhmmss，重新产生，不同于原消费，此处默认取demo演示页面传递的参数
                    'txnAmt' => strval(intval($refundAmount*100)),       //交易金额，退货总金额需要小于等于原消费
                );
                AcpService::sign ( $params ); // 签名
                /** @noinspection PhpUndefinedFieldInspection */
                $url = SDKConfig::getSDKConfig()->backTransUrl;

                $tryTimes = 0;
                startUpacpRefundRequest:

                /** @noinspection PhpParamsInspection */
                $refundResultArr = AcpService::post ( $params, $url);
                if(count($refundResultArr)<=0) { //没收到200应答的情况
                    throw new \RuntimeException("Item order cannot be refunded because server cannot connect to upacp");
                }
                if (!AcpService::validate ($refundResultArr)) {
                    throw new \RuntimeException("Item order cannot be refunded because upacp sign failed");
                }
                $tryTimes++;

                $isRefundSuccess = false;
                if ($refundResultArr["respCode"] == "00") {
                    $isRefundSuccess = true;
                }
                if ($refundResultArr["respCode"] == "03"
                    || $refundResultArr["respCode"] == "04"
                    || $refundResultArr["respCode"] == "05" ){

                    if ($tryTimes < 5) {
                        goto startUpacpRefundRequest;
                    }
                }
                if (!$isRefundSuccess) {
                    throw new \RuntimeException("Item order cannot be refunded because upacp failed: ". json_encode($refundResultArr));
                }
            } else {
                throw new \RuntimeException("Item order cannot be refunded because upacp no callback params");
            }
        }

```


