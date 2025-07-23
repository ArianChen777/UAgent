# AgentU DDD 领域建模分析文档

## 第一部分：对话领域（Conversation Domain）

### 1. 领域概述

#### 1.1 业务背景
AgentU 对话领域是整个平台的核心业务，负责管理用户与AI助手的完整对话体验。该领域借鉴了big-market项目的DDD设计思路，采用清晰的领域划分和简化的架构设计。

#### 1.2 领域划分

##### 核心领域
- **conversation（对话领域）**：管理对话会话生命周期、参与者管理
- **chat（聊天领域）**：处理消息发送、AI调用、响应生成  
- **model（模型领域）**：用户AI模型配置、服务商管理

##### 支撑领域
- **user（用户领域）**：用户管理、认证授权（后续设计）

#### 1.3 核心域识别
- **核心域**：conversation、chat（核心竞争力）
- **支撑域**：model（支撑业务）
- **通用域**：user（通用功能）

### 2. 事件风暴分析

#### 2.1 业务场景分析

##### 用户故事（对话场景）
- 作为用户，我希望能够创建新的对话会话
- 作为用户，我希望能够在会话中发送消息并获得AI响应
- 作为用户，我希望能够查看历史对话记录
- 作为用户，我希望能够暂停和恢复对话

##### 关键业务流程
1. **会话创建流程**：用户创建会话 → 初始化上下文 → 会话状态为活跃
2. **消息处理流程**：发送消息 → 内容验证 → AI调用 → 响应生成 → 保存消息
3. **会话管理流程**：会话状态变更 → 历史归档

#### 2.2 领域事件识别（仅关键跨聚合事件）

##### 对话领域事件（简化版）
- `MessageSentEvent` - 消息发送（触发AI处理）
- `ConversationArchivedEvent` - 会话归档（触发数据清理）

#### 2.3 命令识别

##### 对话管理命令
- `CreateConversationCommand` - 创建对话
- `SendMessageCommand` - 发送消息
- `ArchiveConversationCommand` - 归档对话

### 3. 聚合设计（参考big-market设计思路）

#### 3.1 对话聚合（ConversationAggregate）

##### 聚合根：ConversationAggregate
```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ConversationAggregate {
    // 聚合标识
    private String conversationId;        // 对话ID（主键）
    private String userId;                // 用户ID（外键）
    
    // 聚合状态
    private ConversationStateVO state;    // 对话状态值对象
    private Date createTime;              // 创建时间
    private Date updateTime;              // 更新时间
    
    // 聚合内实体集合
    private List<MessageEntity> messages; // 消息实体列表
    
    // 聚合业务方法
    public MessageEntity sendMessage(String content, String modelId) {
        // 1. 验证会话状态
        if (!state.canSendMessage()) {
            throw new IllegalStateException("当前会话状态不允许发送消息");
        }
        
        // 2. 创建消息实体
        MessageEntity message = MessageEntity.builder()
            .messageId(generateMessageId())
            .conversationId(this.conversationId)
            .content(content)
            .role(MessageRoleVO.USER)
            .modelId(modelId)
            .createTime(new Date())
            .build();
        
        // 3. 添加到聚合
        this.messages.add(message);
        this.updateTime = new Date();
        
        return message;
    }
    
    public void receiveAIResponse(String messageId, String responseContent) {
        // 创建AI响应消息
        MessageEntity aiMessage = MessageEntity.builder()
            .messageId(generateMessageId())
            .conversationId(this.conversationId)
            .content(responseContent)
            .role(MessageRoleVO.ASSISTANT)
            .replyToMessageId(messageId)
            .createTime(new Date())
            .build();
        
        this.messages.add(aiMessage);
        this.updateTime = new Date();
    }
    
    public void archive() {
        if (this.state != ConversationStateVO.ACTIVE) {
            throw new IllegalStateException("只有活跃状态的会话才能归档");
        }
        
        this.state = ConversationStateVO.ARCHIVED;
        this.updateTime = new Date();
    }
}
```

#### 3.2 模型配置聚合（ModelConfigAggregate）

##### 聚合根：ModelConfigAggregate
```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ModelConfigAggregate {
    // 聚合标识
    private String userId;                // 用户ID
    private String configId;              // 配置ID
    
    // 聚合内实体集合
    private List<ModelProviderEntity> providers; // 模型服务商列表
    
    // 业务方法
    public void configureProvider(ProviderTypeVO type, String apiKey, String baseUrl) {
        ModelProviderEntity provider = ModelProviderEntity.builder()
            .providerId(generateProviderId())
            .userId(this.userId)
            .type(type)
            .apiKey(apiKey)
            .baseUrl(baseUrl)
            .status(ProviderStatusVO.CONFIGURED)
            .createTime(new Date())
            .build();
            
        // 验证配置有效性
        provider.validateConfig();
        
        this.providers.add(provider);
    }
    
    public List<AIModelEntity> getAvailableModels() {
        return providers.stream()
            .filter(provider -> provider.getStatus() == ProviderStatusVO.ACTIVE)
            .flatMap(provider -> provider.getModels().stream())
            .collect(Collectors.toList());
    }
}
```

### 4. 实体设计

#### 4.1 消息实体（MessageEntity）
```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class MessageEntity {
    // 实体标识
    private String messageId;             // 消息ID（唯一标识）
    private String conversationId;        // 所属会话ID
    
    // 业务属性
    private String content;               // 消息内容
    private MessageRoleVO role;           // 消息角色（值对象）
    private String modelId;               // 使用的模型ID
    private String replyToMessageId;      // 回复的消息ID
    
    // 时间属性
    private Date createTime;              // 创建时间
    
    // 实体业务方法
    public boolean isUserMessage() {
        return MessageRoleVO.USER.equals(this.role);
    }
    
    public boolean isAIMessage() {
        return MessageRoleVO.ASSISTANT.equals(this.role);
    }
    
    public boolean isEmpty() {
        return content == null || content.trim().isEmpty();
    }
    
    public void validate() {
        if (isEmpty()) {
            throw new IllegalArgumentException("消息内容不能为空");
        }
        if (content.length() > 4000) {
            throw new IllegalArgumentException("消息内容过长");
        }
    }
}
```

#### 4.2 模型服务商实体（ModelProviderEntity）
```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ModelProviderEntity {
    // 实体标识
    private String providerId;            // 服务商ID
    private String userId;                // 用户ID
    
    // 业务属性
    private ProviderTypeVO type;          // 服务商类型
    private String apiKey;                // API密钥
    private String baseUrl;               // API地址
    private ProviderStatusVO status;      // 服务商状态
    
    // 聚合内实体
    private List<AIModelEntity> models;   // 模型列表
    
    // 时间属性
    private Date createTime;              // 创建时间
    
    // 业务方法
    public void validateConfig() {
        if (apiKey == null || apiKey.trim().isEmpty()) {
            throw new IllegalArgumentException("API Key不能为空");
        }
        if (baseUrl == null || baseUrl.trim().isEmpty()) {
            throw new IllegalArgumentException("Base URL不能为空");
        }
        // 这里可以添加实际的API连接测试
    }
    
    public void activate() {
        if (this.status != ProviderStatusVO.CONFIGURED) {
            throw new IllegalStateException("服务商必须先配置才能激活");
        }
        this.status = ProviderStatusVO.ACTIVE;
    }
}
```

### 5. 值对象设计

#### 5.1 对话状态值对象（ConversationStateVO）
```java
@Getter
@AllArgsConstructor
public enum ConversationStateVO {
    ACTIVE("active", "活跃中"),
    PAUSED("paused", "已暂停"),
    ARCHIVED("archived", "已归档"),
    DELETED("deleted", "已删除");
    
    private final String code;
    private final String desc;
    
    public boolean canSendMessage() {
        return this == ACTIVE;
    }
    
    public boolean canArchive() {
        return this == ACTIVE || this == PAUSED;
    }
}
```

#### 5.2 消息角色值对象（MessageRoleVO）
```java
@Getter
@AllArgsConstructor
public enum MessageRoleVO {
    USER("user", "用户"),
    ASSISTANT("assistant", "AI助手"),
    SYSTEM("system", "系统");
    
    private final String code;
    private final String desc;
}
```

#### 5.3 服务商类型值对象（ProviderTypeVO）
```java
@Getter
@AllArgsConstructor
public enum ProviderTypeVO {
    OPENAI("openai", "OpenAI"),
    CLAUDE("claude", "Anthropic Claude"),
    GEMINI("gemini", "Google Gemini"),
    QIANWEN("qianwen", "阿里通义千问");
    
    private final String code;
    private final String desc;
}
```

### 6. 项目目录结构（简化版，去除暂不需要的部分）

```
agentu/
├── app/                          # 应用启动层
│   ├── src/main/java/cn/agentu/
│   │   ├── Application.java             # Spring Boot启动类
│   │   └── config/                      # 应用配置
├── domain/                       # 领域核心层
│   └── src/main/java/cn/agentu/domain/
│       ├── conversation/                # 对话子域
│       │   ├── model/
│       │   │   ├── entity/              # 实体
│       │   │   │   └── MessageEntity.java
│       │   │   ├── valobj/              # 值对象
│       │   │   │   ├── ConversationStateVO.java
│       │   │   │   └── MessageRoleVO.java
│       │   │   └── aggregate/           # 聚合
│       │   │       └── ConversationAggregate.java
│       │   ├── service/                 # 领域服务
│       │   │   └── IConversationService.java
│       │   └── repository/              # 仓储接口
│       │       └── IConversationRepository.java
│       ├── chat/                        # 聊天子域
│       │   ├── service/
│       │   │   ├── IChatProcessService.java
│       │   │   └── chain/               # 责任链模式
│       │   │       ├── IMessageProcessChain.java
│       │   │       ├── impl/
│       │   │       │   ├── ValidationChain.java
│       │   │       │   ├── ContentFilterChain.java
│       │   │       │   └── AIProcessChain.java
│       │   │       └── factory/
│       │   │           └── MessageProcessChainFactory.java
│       │   └── repository/
│       │       └── IChatProcessRepository.java
│       └── model/                       # 模型配置子域
│           ├── model/
│           │   ├── entity/
│           │   │   ├── ModelProviderEntity.java
│           │   │   └── AIModelEntity.java
│           │   ├── valobj/
│           │   │   ├── ProviderTypeVO.java
│           │   │   └── ProviderStatusVO.java
│           │   └── aggregate/
│           │       └── ModelConfigAggregate.java
│           ├── service/
│           │   └── IModelConfigService.java
│           └── repository/
│               └── IModelConfigRepository.java
├── infrastructure/               # 基础设施层
│   └── src/main/java/cn/agentu/infrastructure/
│       ├── persistent/                  # 持久化
│       │   ├── dao/
│       │   ├── po/                      # 持久化对象
│       │   └── repository/              # 仓储实现
│       │       ├── ConversationRepository.java
│       │       └── ModelConfigRepository.java
│       └── adapter/                     # 适配器
│           └── model/                   # 模型适配器
│               ├── OpenAIAdapter.java
│               └── ClaudeAdapter.java
├── trigger/                      # 触发器层
│   └── src/main/java/cn/agentu/trigger/
│       └── http/                        # HTTP接口
│           ├── ConversationController.java
│           └── ModelConfigController.java
├── api/                          # API定义层
│   └── src/main/java/cn/agentu/api/
│       ├── IConversationService.java
│       ├── IModelConfigService.java
│       └── dto/                         # 数据传输对象
│           ├── request/
│           │   ├── CreateConversationRequest.java
│           │   └── SendMessageRequest.java
│           └── response/
│               ├── ConversationResponse.java
│               └── MessageResponse.java
├── agentu-types/                        # 类型定义层
│   └── src/main/java/cn/agentu/types/
│       ├── common/
│       │   ├── Constants.java
│       │   └── Response.java
│       └── enums/
│           ├── ResponseCode.java
│           └── ModelType.java
└── agentu-test/                         # 测试层
    └── src/test/java/cn/agentu/
        ├── domain/                      # 领域测试
        └── infrastructure/              # 基础设施测试
```

### 7. API层业务编排设计（参考big-market的draw方法）

#### 7.1 ConversationController - 发送消息接口

```java
@Slf4j
@RestController
@RequestMapping("/api/v1/conversation/")
public class ConversationController implements IConversationService {
    
    @Resource
    private IConversationService conversationService;
    @Resource
    private IModelConfigService modelConfigService;
    @Resource
    private IChatProcessService chatProcessService;
    
    /**
     * 发送消息接口 - 参考big-market的draw方法业务编排
     * 
     * @param request 发送消息请求
     * @return 消息响应结果
     */
    @RequestMapping(value = "send_message", method = RequestMethod.POST)
    @Override
    public Response<MessageResponse> sendMessage(@RequestBody SendMessageRequest request) {
        try {
            log.info("发送消息开始 userId:{} conversationId:{}", request.getUserId(), request.getConversationId());
            
            // 1. 参数校验
            if (StringUtils.isBlank(request.getUserId()) || 
                StringUtils.isBlank(request.getConversationId()) ||
                StringUtils.isBlank(request.getContent())) {
                throw new AppException(ResponseCode.ILLEGAL_PARAMETER.getCode(), ResponseCode.ILLEGAL_PARAMETER.getInfo());
            }
            
            // 2. 验证模型可用性（调用model领域服务）
            AIModelEntity aiModel = modelConfigService.getModel(request.getModelId());
            if (aiModel == null || !aiModel.isAvailable()) {
                throw new AppException(ResponseCode.MODEL_NOT_AVAILABLE.getCode(), "模型不可用");
            }
            
            // 3. 发送消息到对话聚合（调用conversation领域服务）
            MessageEntity userMessage = conversationService.sendMessage(
                request.getConversationId(), 
                request.getContent(), 
                request.getModelId()
            );
            log.info("用户消息已保存 userId:{} messageId:{}", request.getUserId(), userMessage.getMessageId());
            
            // 4. 执行AI处理（调用chat领域服务）
            String aiResponse = chatProcessService.processMessage(
                userMessage.getMessageId(), 
                request.getContent(),
                request.getModelId()
            );
            log.info("AI处理完成 messageId:{} responseLength:{}", userMessage.getMessageId(), aiResponse.length());
            
            // 5. 保存AI响应到对话聚合
            conversationService.receiveAIResponse(
                request.getConversationId(),
                userMessage.getMessageId(),
                aiResponse
            );
            
            // 6. 返回结果
            return Response.<MessageResponse>builder()
                .code(ResponseCode.SUCCESS.getCode())
                .info(ResponseCode.SUCCESS.getInfo())
                .data(MessageResponse.builder()
                    .messageId(userMessage.getMessageId())
                    .response(aiResponse)
                    .build())
                .build();
                
        } catch (AppException e) {
            log.error("发送消息失败 userId:{} conversationId:{}", request.getUserId(), request.getConversationId(), e);
            return Response.<MessageResponse>builder()
                .code(e.getCode())
                .info(e.getInfo())
                .build();
        } catch (Exception e) {
            log.error("发送消息失败 userId:{} conversationId:{}", request.getUserId(), request.getConversationId(), e);
            return Response.<MessageResponse>builder()
                .code(ResponseCode.UN_ERROR.getCode())
                .info(ResponseCode.UN_ERROR.getInfo())
                .build();
        }
    }
    
    /**
     * 创建对话接口
     */
    @RequestMapping(value = "create", method = RequestMethod.POST)
    @Override
    public Response<ConversationResponse> createConversation(@RequestBody CreateConversationRequest request) {
        try {
            log.info("创建对话开始 userId:{}", request.getUserId());
            
            // 1. 参数校验
            if (StringUtils.isBlank(request.getUserId())) {
                throw new AppException(ResponseCode.ILLEGAL_PARAMETER.getCode(), ResponseCode.ILLEGAL_PARAMETER.getInfo());
            }
            
            // 2. 创建对话聚合（调用conversation领域服务）
            ConversationAggregate conversation = conversationService.createConversation(request.getUserId(), request.getTitle());
            
            // 3. 返回结果
            return Response.<ConversationResponse>builder()
                .code(ResponseCode.SUCCESS.getCode())
                .info(ResponseCode.SUCCESS.getInfo())
                .data(ConversationResponse.builder()
                    .conversationId(conversation.getConversationId())
                    .title(request.getTitle())
                    .state(conversation.getState().getCode())
                    .createTime(conversation.getCreateTime())
                    .build())
                .build();
                
        } catch (Exception e) {
            log.error("创建对话失败 userId:{}", request.getUserId(), e);
            return Response.<ConversationResponse>builder()
                .code(ResponseCode.UN_ERROR.getCode())
                .info(ResponseCode.UN_ERROR.getInfo())
                .build();
        }
    }
}
```

#### 7.2 责任链处理模式（简化版）

```java
// 消息处理责任链接口
public interface IMessageProcessChain {
    MessageProcessResult process(MessageProcessEntity entity);
    IMessageProcessChain appendNext(IMessageProcessChain next);
}

// 抽象责任链
public abstract class AbstractMessageProcessChain implements IMessageProcessChain {
    private IMessageProcessChain next;
    
    @Override
    public IMessageProcessChain appendNext(IMessageProcessChain next) {
        this.next = next;
        return next;
    }
    
    @Override
    public MessageProcessResult process(MessageProcessEntity entity) {
        MessageProcessResult result = doProcess(entity);
        if (!result.isSuccess() || next == null) {
            return result;
        }
        return next.process(entity);
    }
    
    protected abstract MessageProcessResult doProcess(MessageProcessEntity entity);
}

// 具体实现
@Component
public class ValidationChain extends AbstractMessageProcessChain {
    @Override
    protected MessageProcessResult doProcess(MessageProcessEntity entity) {
        if (entity.getContent() == null || entity.getContent().trim().isEmpty()) {
            return MessageProcessResult.fail("消息内容不能为空");
        }
        return MessageProcessResult.success();
    }
}

@Component
public class AIProcessChain extends AbstractMessageProcessChain {
    @Resource
    private OpenAIAdapter openAIAdapter;
    
    @Override
    protected MessageProcessResult doProcess(MessageProcessEntity entity) {
        try {
            String response = openAIAdapter.chat(entity.getContent(), entity.getModelId());
            return MessageProcessResult.success(response);
        } catch (Exception e) {
            return MessageProcessResult.fail("AI处理失败: " + e.getMessage());
        }
    }
}

// 工厂类
@Service
public class MessageProcessChainFactory {
    @Resource
    private ValidationChain validationChain;
    @Resource
    private AIProcessChain aiProcessChain;
    
    public IMessageProcessChain createProcessChain() {
        return validationChain.appendNext(aiProcessChain);
    }
}
```

### 8. 总结

#### 8.1 简化设计的关键点

1. **移除过度设计**：
   - 删除了MQ、Job等暂时不需要的组件
   - 简化了事件系统，只保留必要的跨聚合事件
   - 移除了复杂的ChatProcessAggregate，简化为服务层处理

2. **修正项目结构**：
   - 项目名改为`agentu`而不是`agentu-conversation`
   - 模块名改为`domain`、`infrastructure`等通用名称
   - 为后续其他领域预留空间

3. **API层业务编排**：
   - 参考big-market的draw方法，在Controller层做业务编排
   - 依次调用：参数校验 → 模型验证 → 对话处理 → AI调用 → 结果保存
   - 统一的异常处理和响应格式

4. **责任链简化**：
   - 只保留核心的验证链和AI处理链
   - 易于扩展，但不过度设计

#### 8.2 设计优势

- **架构清晰**：严格按照big-market的六边形架构，但去除不必要部分
- **职责分离**：聚合职责单一，API层负责业务编排
- **易于实现**：当前需求下的最小可行设计
- **扩展友好**：为后续功能扩展预留空间

#### 8.3 后续实施建议

1. **实现优先级**：
   - 先实现conversation聚合和基础的消息处理
   - 再实现model配置功能
   - 最后实现责任链扩展

2. **当前场景适配**：
   - 暂时不需要复杂的异步处理
   - 暂时不需要MQ和定时任务
   - 专注于核心的对话功能实现

3. **架构演进**：
   - 随着业务复杂度增加，可以逐步引入事件驱动
   - 可以根据需要添加缓存、MQ等组件

这个简化后的设计更加实用，符合当前业务需求，同时为未来扩展预留了空间。

---

*文档版本：v2.1*  
*更新时间：2025-07-23*  
*参考项目：big-market*