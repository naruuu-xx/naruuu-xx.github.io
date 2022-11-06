---
title: "【小程序开发实战】使用WxJava实现手机号获取"
date: 2022-11-06T10:33:29+08:00
draft: true
---
之前编写小程序获取手机号相关的代码均为自行封装，最近发现了 WxJava 的存在，方便了开发。

本篇文章将讲解如何使用WxJava进行手机号的获取。

** 旧版本 **
[微信接口官方文档](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/deprecatedGetPhoneNumber.html)

## 思路

需要将 button 组件 `open-type` 的值设置为 `getPhoneNumber`，当用户点击并同意之后，可以通过 `bindgetphonenumber` 事件回调获取到微信服务器返回的加密数据， 然后在第三方服务端结合 `session_key` 以及 `app_id` 进行解密获取手机号。

总结为以下两个步骤：
1. 通过使用 button getPhoneNumber 可获取到 返回的加密数据。
2. 然后将加密数据结合 session_key 以及 app_id 进行解密获取手机号。

因此想要获得使用者的手机号，我们需要传入 session_key, app_id 以及 前端调用微信服务器返回的加密数据。

- 如何获取 session_key 和 app_id 呢？
	前端获取jsCode后并调用 auth.code2Session 接口即可。

因此我们只需要将 jsCode 以及 加密数据 传给接口即可。

### 整体思路总结

1. 前端给后端传递 jsCode，加密数据参数（iv，encryptedData）。
2. 然后使用 jsCode，绑定的 appId 以及 appSecret 调用 auth.code2Session 接口 获取到 session_key，app_id 。
3. 使用 session_key，app_id 对加密数据进行解密获取到用户手机号。

## 代码实现

这里使用了 wxjava。因此代码非常的简洁。我们只需要理解整体的思路就好了。

### 整体架构
![[Pasted image 20221105171024.png]]


wxMaConfigration
```java

  
/**  
 * @author <a href="https://github.com/binarywang">Binary Wang</a>  
 */@Slf4j  
@Configuration  
@EnableConfigurationProperties(WxMaProperties.class)  
public class WxMaConfiguration {  
    private final WxMaProperties properties;  
  
    @Autowired  
    public WxMaConfiguration(WxMaProperties properties) {  
        this.properties = properties;  
    }  
  
    @Bean  
    public WxMaService wxMaService() {  
        List<WxMaProperties.Config> configs = this.properties.getConfigs();  
        if (configs == null) {  
            throw new WxRuntimeException("大哥，拜托先看下项目首页的说明（readme文件），添加下相关配置，注意别配错了！");  
        }  
        WxMaService maService = new WxMaServiceImpl();  
        maService.setMultiConfigs(  
            configs.stream()  
                .map(a -> {  
                    WxMaDefaultConfigImpl config = new WxMaDefaultConfigImpl();  
//                WxMaDefaultConfigImpl config = new WxMaRedisConfigImpl(new JedisPool());  
                    // 使用上面的配置时，需要同时引入jedis-lock的依赖，否则会报类无法找到的异常  
                    config.setAppid(a.getAppid());  
                    config.setSecret(a.getSecret());  
                    config.setToken(a.getToken());  
                    config.setAesKey(a.getAesKey());  
                    config.setMsgDataFormat(a.getMsgDataFormat());  
                    return config;  
                }).collect(Collectors.toMap(WxMaDefaultConfigImpl::getAppid, a -> a, (o, n) -> o)));  
        return maService;  
    }  
  
    @Bean  
    public WxMaMessageRouter wxMaMessageRouter(WxMaService wxMaService) {  
        final WxMaMessageRouter router = new WxMaMessageRouter(wxMaService);  
        router  
            .rule().handler(logHandler).next()  
            .rule().async(false).content("订阅消息").handler(subscribeMsgHandler).end()  
            .rule().async(false).content("文本").handler(textHandler).end()  
            .rule().async(false).content("图片").handler(picHandler).end()  
            .rule().async(false).content("二维码").handler(qrcodeHandler).end();  
        return router;  
    }  
  
    private final WxMaMessageHandler subscribeMsgHandler = (wxMessage, context, service, sessionManager) -> {  
        service.getMsgService().sendSubscribeMsg(WxMaSubscribeMessage.builder()  
            .templateId("此处更换为自己的模板id")  
            .data(Lists.newArrayList(  
                new WxMaSubscribeMessage.MsgData("keyword1", "339208499")))  
            .toUser(wxMessage.getFromUser())  
            .build());  
        return null;  
    };  
  
    private final WxMaMessageHandler logHandler = (wxMessage, context, service, sessionManager) -> {  
        log.info("收到消息：" + wxMessage.toString());  
        service.getMsgService().sendKefuMsg(WxMaKefuMessage.newTextBuilder().content("收到信息为：" + wxMessage.toJson())  
            .toUser(wxMessage.getFromUser()).build());  
        return null;  
    };  
  
    private final WxMaMessageHandler textHandler = (wxMessage, context, service, sessionManager) -> {  
        service.getMsgService().sendKefuMsg(WxMaKefuMessage.newTextBuilder().content("回复文本消息")  
            .toUser(wxMessage.getFromUser()).build());  
        return null;  
    };  
  
    private final WxMaMessageHandler picHandler = (wxMessage, context, service, sessionManager) -> {  
        try {  
            WxMediaUploadResult uploadResult = service.getMediaService()  
                .uploadMedia("image", "png",  
                    ClassLoader.getSystemResourceAsStream("tmp.png"));  
            service.getMsgService().sendKefuMsg(  
                WxMaKefuMessage  
                    .newImageBuilder()  
                    .mediaId(uploadResult.getMediaId())  
                    .toUser(wxMessage.getFromUser())  
                    .build());  
        } catch (WxErrorException e) {  
            e.printStackTrace();  
        }  
  
        return null;  
    };  
  
    private final WxMaMessageHandler qrcodeHandler = (wxMessage, context, service, sessionManager) -> {  
        try {  
            final File file = service.getQrcodeService().createQrcode("123", 430);  
            WxMediaUploadResult uploadResult = service.getMediaService().uploadMedia("image", file);  
            service.getMsgService().sendKefuMsg(  
                WxMaKefuMessage  
                    .newImageBuilder()  
                    .mediaId(uploadResult.getMediaId())  
                    .toUser(wxMessage.getFromUser())  
                    .build());  
        } catch (WxErrorException e) {  
            e.printStackTrace();  
        }  
  
        return null;  
    };  
  
}
```

wxMaProperties

```java
@Data  
@ConfigurationProperties(prefix = "wx.miniapp")  
public class WxMaProperties {  
  
    private List<Config> configs;  
  
    @Data  
    public static class Config {  
        /**  
         * 设置微信小程序的appid  
         */        private String appid;  
  
        /**  
         * 设置微信小程序的Secret  
         */        private String secret;  
  
        /**  
         * 设置微信小程序消息服务器配置的token  
         */        private String token;  
  
        /**  
         * 设置微信小程序消息服务器配置的EncodingAESKey  
         */        private String aesKey;  
  
        /**  
         * 消息格式，XML或者JSON  
         */        private String msgDataFormat;  
    }  
  
}
```



wxcontroller
```java
@RestController  
@RequestMapping("/wx")  
@AllArgsConstructor  
public class WxController {  
  
    private final WxMaService wxMaService;  
  
  
    @PostMapping("/decrypt")  
    public JSONObject decrypt(@RequestBody AuthDto authDto) throws WxErrorException {  
        WxMaJscode2SessionResult wxMaJscode2SessionResult  = new WxMaJscode2SessionResult();  
        try{  
         wxMaJscode2SessionResult =  wxMaService.jsCode2SessionInfo(authDto.getJsCode());  
        }  
        catch (Exception e){  
            throw e;  
        }  
  
  
        WxMaPhoneNumberInfo phoneNumberInfo = wxMaService.getUserService().getPhoneNoInfo(wxMaJscode2SessionResult.getSessionKey(),authDto.getEncryptedData(), authDto.getIv());  
        JSONObject result = new JSONObject();  
        HashMap<String,Object> resultMap = new HashMap<>();  
        resultMap.put("data",phoneNumberInfo);  
        result.put("result",resultMap);  
        return result;  
    }  
  
  
}
```

WxController
```java
  
@RestController  
@RequestMapping("/wx")  
@AllArgsConstructor  
public class WxController {  
  
    private final WxMaService wxMaService;  
  
  
    @PostMapping("/decrypt")  
    public JSONObject decrypt(@RequestBody AuthDto authDto) throws WxErrorException {  
        WxMaJscode2SessionResult wxMaJscode2SessionResult  = new WxMaJscode2SessionResult();  
        try{  
         wxMaJscode2SessionResult =  wxMaService.jsCode2SessionInfo(authDto.getJsCode());  
        }  
        catch (Exception e){  
            throw e;  
        }  
  
  
        WxMaPhoneNumberInfo phoneNumberInfo = wxMaService.getUserService().getPhoneNoInfo(wxMaJscode2SessionResult.getSessionKey(),authDto.getEncryptedData(), authDto.getIv());  
        JSONObject result = new JSONObject();  
        HashMap<String,Object> resultMap = new HashMap<>();  
        resultMap.put("data",phoneNumberInfo);  
        result.put("result",resultMap);  
        return result;  
    }  
  
  
}
```

application.yml

```yml
wx:  
  miniapp:  
    configs:  
          appid: '自己的小程序appid'  
          secret: 'appsecret'  
          token: #微信小程序消息服务器配置的token  
          aesKey: #微信小程序消息服务器配置的EncodingAESKey  
          msgDataFormat: JSON
```