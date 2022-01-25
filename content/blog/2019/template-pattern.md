---
title: '【设计模式-实践】使用 模板模式 实现 微信微信模板消息推送'
date: 2019-02-19 18:40:57
updated: 2019-02-19 18:41:00
categories: 
- review
tags: 
- 设计模式
- 微信模板消息
- review
description: 使用 模板模式 实现 微信微信模板消息推送。

---
> 使用 模板模式 实现 微信微信模板消息推送。

<!--more-->

# 背景
公司需要向客户推送模板消息，包括：
- 服务节点变更
- 服务人员变更
- 订单支付成功通知
- 社保代缴通知

# 实现
服务对外暴露 http 接口。
所有的微信模板相关的信息都保存在 该服务中，应用服务只知道接口，`保持高内聚`。
在内部 发送消息使用 MQ。因为消息通知的`即时性要求不高`，且`在时间上的局部性强`。

流程：
- 服务接收到 发送消息的请求，在消息对象中加入 `templateCode`，发送到 MQ。
- MQ 消费者 调用 `WxConsumer.process()` 进行消费。
- WxConsumer 根据 传入dto 的 templateCode，调用相应的发送方法。
- 发送方法中
  - 调用 `template.fillTemplate(WxNotifyDto)` 拼装 消息体
  - 调用 微信 或者 短信服务商的 http 接口
  - 持久化消息

```java
public class WxController {
    @ApiOperation(value = "服务人员变更")
    @ApiImplicitParams({@ApiImplicitParam(name = Constants.AUTH_KEY, required = true, dataType = "string", paramType = "header")})
    @PostMapping("/notify/person")
    public ResultData crmServPersModNotify(@RequestBody WxPersonModifyDto dto) {
        dto.setTemplate(WxTemplateCode.PERSON_MODIFY);
        rabbitTemplate.convertAndSend(MQConfig.WEIXIN_NOTIFY_QUEUE, dto);
        return new ResultData();
    }
}

@Service
public class MessageHandler {
    @Autowired
    private WxConsumer wxConsumer;

    @RabbitListener(queues = MQConfig.WEIXIN_NOTIFY_QUEUE)
    public void sendWxNotify(WxNotifyDto wxNotifyDto) {
        wxConsumer.process(wxNotifyDto);
    }
}
```
## WxConsumer
```java
public class WxConsumer {
   public boolean process(WxNotifyDto notifyDto) {
        WxTemplateCode template = notifyDto.getTemplate();
        try {
            switch (template) {
                case SERVICE_PROCESS:
                    return processServiceProcess((WxServiceProcessDto) notifyDto, new WxServiceProcessTemplate());
                case PERSON_MODIFY:
                    return processPersonModify((WxPersonModifyDto) notifyDto, new WxPersonModifyTemplate());
                case SOCIAL_SECURITY:
                    return processSocialSecurity((WxSocialSecurityDto) notifyDto, new SocialSecurityTemplate());
                default:
                    log.error("没有匹配的消息模板，dto={}", notifyDto);
            }
        } catch (Exception ex) {
            log.error("[WxService#process]");
            throw new AmqpRejectAndDontRequeueException(ex);
        }
        return false;
    }

    private boolean processServiceProcess(WxServiceProcessDto dto, WxServiceProcessTemplate template) throws Exception {
        WxServiceProcess info = WxServiceProcess.fromDto(dto);
        ResultDto result = send(template.fillTemplate(dto));
    }
}
```
## WxTemplate
WxTemplate 用于定义模板的格式。
```java
public abstract class WxTemplate {
    private String touser = "";
    // 为了匹配微信的消息格式，这里没有使用驼峰命名。
    // 当然也可以，引入 序列化的类库进行处理。
    private String template_id = "";
    private String url = "";
    private String topcolor = "";
    // 要填充的数据
    // WxData 只有 value，color 两个 field，这里不列出了。
    private Map<String, WxData> data = new HashMap<>();

    // 使用 builder 模式，方便调用
    // 对于保留关键字，提供单独的实现接口，这里的 first 和 remark。
    WxTemplate first(WxData wxData) {
        addData("first", wxData);
        return this;
    }
    WxTemplate remark(WxData wxData) {
        addData("remark", wxData);
        return this;
    }
    public WxTemplate addData(String name, WxData wxData) {
        data.put(name, wxData);
        return this;
    }
    // 其他 填充参数的方法

    // 核心方法
    public abstract WxTemplate fillTemplate(WxNotifyDto wxNotifyDto);
    public abstract WxTemplate fillTemplate(WxNotifyDto wxNotifyDto, String title);
}
```
## WxNotifyDto
其中，WxTemplateCode 是个枚举类，保存了 微信公众平台配置的 模板 ID。
```java
public abstract class WxNotifyDto {
    protected String targetOpenId;
    protected WxTemplateCode template;
    protected String remark;
}
```

# 使用
需要引入新的模板消息时，
需要新建
- 一个 WxNotifyDto 的子类
- 一个 WxTemplate 的子类
- 在 `WxConsumer` 中定义自己的发送事件。
- 在 `WxConsumer.process` 方法中注册自己的发送事件

发送事件内容：
- 根据手机号，查询相关客户信息（openId，客户姓名 等等）
- 判断是否有 openId
  - 如果有 openId 的话，获取 微信 access_token，发送模板消息
    - 如果发送失败，发送短信。
  - 如果没有 openId，发送短信。
- 保存 消息发送结果到 DB。如果有失败的话，后续人工进行重发。

# 消息保证
这里 消息保证的要求不高。
采取的策略是 最少一次。

如果需要消息保证的话，那么应该加上：
- 发布者确认
- 死信队列
- 对于发送失败的消息，不进行重试，而是直接保存到 DB，后续人工操作。