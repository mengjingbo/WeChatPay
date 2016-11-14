# WeChatPay
Android微信支付
#### 1.阅读文档，配置信息
- 移动应用微信支付商户接入指导文档(按照微信需求填写信息,申请商户ID):
https://open.weixin.qq.com/cgi-bin/showdocument?action=dir_list&t=resource/res_list&verify=1&id=open1419317780&token=&lang=zh_CN
- 开发文档:
https://pay.weixin.qq.com/wiki/doc/api/app/app.php?chapter=9_1
- 开发工具包和SDK下载：
 https://open.weixin.qq.com/cgi-bin/showdocument?action=dir_list&t=resource/res_list&verify=1&id=open1419319167&lang=zh_CN

>按照以上要求配置信息得到商户Id，在商户平台生成的密钥，AppId，AppSecret。

一般将这些信息写在一个常量类里面方便维护，如下：
```java
   /**
    * 微信支付必备参数
    */
    public calss WeChatConstants{

         public static final String WECHAT_APP_ID = "在微信开发平台生成的AppId";

         public static final String WECHAT_APP_SECRET = "在微信开发平台生成的AppSecret";

         public static final String WECHAT_PARTNER_ID = "商户ID";

         public static final String WECHAT_PARTNER_KEY = "在商户平台生成的密钥";

         public static final String WECHAT_RESULT_NOTIFY = "微信回调服务器接口地址";

         public static final String WECHAT_UNIFIED_ORDER = "微信统一下单接口地址";
    }
```
可根据项目需求替换以上参数值供自己使用。

- 自定义微信支付model ，方便参数设置。

```javascript
   /**
    * 描述 ：只需要定义 key, value 即可。
    */
    public class WeChatPayBean<K, V> {

        private K key;

        private V value;

        public WeChatPayBean(K key, V value) {
            this.key = key;
            this.value = value;
        }
    
        public K getKey() {
            return key;
        }

        public void setKey(K key) {
            this.key = key;
        }

        public V getValue() {
            return value;
        }

        public void setValue(V value) {
            this.value = value;
        }
    }
```
#### 2.开始支付
> 首先商户系统先调用该接口在微信支付服务后台生成预支付交易单，返回正确的预支付交易回话标识后再在APP里面调起支付。也就是说服务端调用该接口从微信那边获得一个与支付订单号。

- 使用AsyncTask获取订单信息
- 微信支付统一下单接口地址:https://api.mch.weixin.qq.com/pay/unifiedorder

```java
    private Map<String, String> mResult ;// 定义一个Map集合，用来接收得到的订单信息
    
    public void getPrepayId(final Activity activity, final IWXAPI mApi) {
        Toast.makeTextactivity, "获取订单中...", Toast.LENGTH_SHORT).show();
        new AsyncTask<Void, Void, Map<String, String>>() {

            @Override
            protected void onPostExecute(Map<String, String> result) {
                if (result != null) {
                    mResult = result;
                    getPayReq(mApi);
                }
            }

            @Override
            protected Map<String, String> doInBackground(Void... voids) {
                String url = String.format(WeChatConstants.WECHAT_UNIFIED_ORDER);// 统一下单接口地址
                byte[] buf = HttpUtils.httpPost(url, getProductArgs());// HttpUtils.httpPost(); 为post请求
                try {
                    return decodeXml(new String(buf));
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return null;
            }
        }.execute();
    }
```
- 申请调起支付

```java
    private void getPayReq(IWXAPI mApi) {
        PayReq mPayReq = new PayReq();
        mPayReq.appId = WeChatConstants.WECHAT_APP_ID;// AppId
        mPayReq.partnerId = WeChatConstants.WECHAT_PARTNER_ID;// 微信支付分配的商户号
        mPayReq.prepayId = mResult.get("prepay_id");// 支付订单Id
        mPayReq.packageValue = "Sign=WXPay";// 扩展字段，暂填写固定值Sign=WXPay
        mPayReq.nonceStr = getNonceString();// 随机数字符串
        mPayReq.timeStamp = String.valueOf(getTimeStamp());// 时间戳
        List<WeChatPayBean> signParams = new AraaryList<WeChatPayBean>();
        signParams.add(new WeChatPayBean("appid", mPayReq.appId));
        signParams.add(new WeChatPayBean("noncestr", mPayReq.nonceStr));
        signParams.add(new WeChatPayBean("package", mPayReq.packageValue));
        signParams.add(new WeChatPayBean("partnerid", mPayReq.partnerId));
        signParams.add(new WeChatPayBean("prepayid", mPayReq.prepayId));
        signParams.add(new WeChatPayBean("timestamp", mPayReq.timeStamp));
        mPayReq.sign = genAppSign(signParams);
        mApi.sendReq(mPayReq);
    }
```
- 统一下单请求参数设置(这里只设置必要参数)：

```java
   /**
    * 根据微信官方支付文档设置需要的参数
    */
    private String getProductArgs() {  
        StringBuffer xml = new StringBuffer();
        try {
            String nonceString = getNonceString();// 获取随机字符串，方法在本贴下面给出
            String bodyString = "xxx充值";// 商品描述交易描述。例如：腾讯充值中心-QQ会员充值
            xml.append("</xml>");
            List<WeChatPayBean> packageParams = new ArrayList<WeChatPayBean>();
            packageParams.add(new WeChatPayBean("appid", WeChatConstants.WECHAT_APP_ID));// AppId
            packageParams.add(new WeChatPayBean("body", bodyString));
            packageParams.add(new WeChatPayBean("mch_id", WeChatConstants.WECHAT_PARTNER_ID));// 微信支付分配的商户号
            packageParams.add(new WeChatPayBean("nonce_str", nonceString));// 随机字符串
            packageParams.add(new WeChatPayBean("notify_url", WeChatConstants.WECHAT_RESULT_NOTIFY ));// 微信回调服务端地址
            packageParams.add(new WeChatPayBean("out_trade_no", "支付订单号"));// 服务端返回的支付订单号
            packageParams.add(new WeChatPayBean("spbill_create_ip", AppUtils.getPhoneIP(AppContext.instance)));
            packageParams.add(new WeChatPayBean("total_fee", "订单金额"));// 订单总金额，单位为分。根据商户商品价格定义，此处需自行设置
            packageParams.add(new WeChatPayBean("trade_type", "APP"));// 交易类型
            packageParams.add(new WeChatPayBean("sign", getAppSign(packageParams)));// 签名，方法在本贴下面
            String xmls = toXml(packageParams);
            return new String(xmls.toString().getBytes(), "ISO8859-1");// 设置编码格式
        } catch (Exception e) {
            return null;
        }
    }
```
- 得到一个随机字符串

```java
    private String getNonceString() {
        Random random = new Random();
        return MD5.getMessageDigest(String.valueOf(random.nextInt(10000)).getBytes());
    }
```
- 微信支付MD5加密工具类

```java
   public class MD5 {    
            
       private MD5() { }    
            
       public final static String getMessageDigest(byte[] buffer) {
            char hexDigits[] = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f'};        
            try {            
                    MessageDigest mdTemp = MessageDigest.getInstance("MD5");
                    mdTemp.update(buffer);
                    byte[] md = mdTemp.digest();
                    int j = md.length;
                    char str[] = new char[j * 2];
                    int k = 0;
                    for (int i = 0; i < j; i++) {
                          byte byte0 = md[i];
                          str[k++] = hexDigits[byte0 >>> 4 & 0xf];
                          str[k++] = hexDigits[byte0 & 0xf];
                    }
                    return new String(str);
            } catch (Exception e) {
                    return null;
            }
        }
   }
```
- 签名
>签名规范文档:https://pay.weixin.qq.com/wiki/doc/api/app/app.php?chapter=4_3

```java
    private String getAppSign(List<PayBean> params) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < params.size(); i++) {
            sb.append(params.get(i).getKey());
            sb.append('=');
            sb.append(params.get(i).getValue());
            sb.append('&');
        }
        sb.append("key=");
        sb.append(WeChatConstants.WECHAT_PARTNER_KEY);// 商户平台生成的密钥
        String appSign = MD5.getMessageDigest(sb.toString().getBytes()).toUpperCase();
        return appSign;
    }
```
- 时间戳

```java
    private long getTimeStamp() {
        SimpleDateFormat time = new SimpleDateFormat("yyyyMMddkkmmss");
        return Long.valueOf(time.format(new Date()));
    }
```
- 生成XML

```java
    private String toXml(List<WeChatPayBean> params) {
        StringBuilder sb = new StringBuilder();
        sb.append("<xml>");
        for (int i = 0; i < params.size(); i++) {
            sb.append("<" + params.get(i).getKey() + ">");
            sb.append(params.get(i).getValue());
            sb.append("</" + params.get(i).getKey() + ">");
        }
        sb.append("</xml>");
        return sb.toString();
    }
```
- 解析XML

```java
    public Map<String, String> decodeXml(String content) {
        try {
            Map<String, String> xml = new HashMap<String, String>();
            XmlPullParser parser = Xml.newPullParser();
            parser.setInput(new StringReader(content));
            int event = parser.getEventType();
            while (event != XmlPullParser.END_DOCUMENT) {
                String nodeName = parser.getName();
                switch (event) {
                    case XmlPullParser.START_DOCUMENT:
                        break;
                    case XmlPullParser.START_TAG:
                        if ("xml".equals(nodeName) == false) {
                            xml.put(nodeName, parser.nextText());
                        }
                        break;
                    case XmlPullParser.END_TAG:
                        break;
                }
                event = parser.next();
            }
            return xml;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
```
#### 3.微信支付扣款成功后，微信会通过回调告诉商户后台扣款成功，此时我们需要调用后台接口查询订单状态。