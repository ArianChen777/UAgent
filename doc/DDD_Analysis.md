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
**数据库映射：对应sessions表**

##### 聚合根：ConversationAggregate
```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ConversationAggregate {
    // 聚合标识（对应sessions.session_id）
    private String conversationId;        // 对话ID（主键）
    private String userId;                // 用户ID（外键）
    
    // 基本信息（对应sessions表字段）
    private String title;                 // 对话标题
    private String description;           // 对话描述
    
    // 模型配置（对应sessions表字段）
    private String modelId;               // 默认模型ID
    private String providerId;            // 默认服务商ID
    private String apiKeyId;              // 默认API密钥ID
    
    // 参数设置（对应sessions表字段）
    private BigDecimal temperature;       // 温度参数
    private Integer maxTokens;            // 最大token数
    private BigDecimal topP;              // top_p参数
    private BigDecimal frequencyPenalty;  // 频率惩罚
    private BigDecimal presencePenalty;   // 存在惩罚
    
    // 功能开关（对应sessions表字段）
    private Boolean enableKnowledgeBase;  // 启用知识库
    private List<String> knowledgeBaseIds; // 知识库ID列表
    private Boolean enableFunctionCalling; // 启用函数调用
    
    // 统计信息（对应sessions表字段）
    private Integer messageCount;         // 消息总数
    private Long totalInputTokens;        // 输入token总数
    private Long totalOutputTokens;       // 输出token总数
    
    // 聚合状态（对应sessions表字段）
    private ConversationStateVO state;    // 对话状态值对象
    private Boolean isPinned;             // 是否置顶
    private Date lastMessageAt;           // 最后消息时间
    private Date createTime;              // 创建时间
    private Date updateTime;              // 更新时间
    
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
            .modelId(modelId != null ? modelId : this.modelId)
            .sequenceNumber(this.messageCount + 1)
            .createTime(new Date())
            .build();
        
        // 3. 更新聚合状态
        this.messageCount++;
        this.lastMessageAt = new Date();
        this.updateTime = new Date();
        
        return message;
    }
    
    public MessageEntity receiveAIResponse(String replyToMessageId, String responseContent, 
                                         Integer inputTokens, Integer outputTokens) {
        // 创建AI响应消息
        MessageEntity aiMessage = MessageEntity.builder()
            .messageId(generateMessageId())
            .conversationId(this.conversationId)
            .content(responseContent)
            .role(MessageRoleVO.ASSISTANT)
            .modelId(this.modelId)
            .providerId(this.providerId)
            .sequenceNumber(this.messageCount + 1)
            .inputTokens(inputTokens)
            .outputTokens(outputTokens)
            .totalTokens(inputTokens + outputTokens)
            .createTime(new Date())
            .build();
        
        // 更新聚合统计信息
        this.messageCount++;
        this.totalInputTokens += inputTokens;
        this.totalOutputTokens += outputTokens;
        this.lastMessageAt = new Date();
        this.updateTime = new Date();
        
        return aiMessage;
    }
    
    public void archive() {
        if (this.state != ConversationStateVO.ACTIVE) {
            throw new IllegalStateException("只有活跃状态的会话才能归档");
        }
        
        this.state = ConversationStateVO.ARCHIVED;
        this.updateTime = new Date();
    }
    
    public void updateModelConfig(String modelId, String providerId, String apiKeyId) {
        this.modelId = modelId;
        this.providerId = providerId;
        this.apiKeyId = apiKeyId;
        this.updateTime = new Date();
    }
    
    public void updateParameters(BigDecimal temperature, Integer maxTokens, 
                               BigDecimal topP, BigDecimal frequencyPenalty, 
                               BigDecimal presencePenalty) {
        this.temperature = temperature;
        this.maxTokens = maxTokens;
        this.topP = topP;
        this.frequencyPenalty = frequencyPenalty;
        this.presencePenalty = presencePenalty;
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
**数据库映射：对应conversations表**

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class MessageEntity {
    // 实体标识（对应conversations.conversation_id）
    private String messageId;             // 消息ID（唯一标识）
    private String conversationId;        // 所属会话ID（对应session_id）
    private String userId;                // 用户ID
    
    // 业务属性（对应conversations表字段）
    private String content;               // 消息内容
    private MessageRoleVO role;           // 消息角色（值对象）
    private ContentTypeVO contentType;    // 内容类型（新增）
    
    // 序号信息（对应conversations表字段）
    private Integer sequenceNumber;       // 序号（新增）
    
    // 模型调用信息（仅AI回复，对应conversations表字段）
    private String modelId;               // 使用的模型ID
    private String providerId;            // 服务商ID
    private String modelName;             // 模型名称
    
    // Token统计（仅AI回复，对应conversations表字段）
    private Integer inputTokens;          // 输入token数
    private Integer outputTokens;         // 输出token数
    private Integer totalTokens;          // 总token数
    
    // 扩展信息（对应conversations表字段）
    private Map<String, Object> metadata; // 元数据
    private List<Object> attachments;     // 附件列表
    
    // 状态管理（对应conversations表字段）
    private MessageStatusVO status;       // 消息状态
    
    // 时间属性
    private Date createTime;              // 创建时间
    private Date updateTime;              // 更新时间
    
    // 实体业务方法
    public boolean isUserMessage() {
        return MessageRoleVO.USER.equals(this.role);
    }
    
    public boolean isAIMessage() {
        return MessageRoleVO.ASSISTANT.equals(this.role);
    }
    
    public boolean isSystemMessage() {
        return MessageRoleVO.SYSTEM.equals(this.role);
    }
    
    public boolean isFunctionMessage() {
        return MessageRoleVO.FUNCTION.equals(this.role);
    }
    
    public boolean isEmpty() {
        return content == null || content.trim().isEmpty();
    }
    
    public boolean isTextContent() {
        return ContentTypeVO.TEXT.equals(this.contentType);
    }
    
    public void validate() {
        if (isEmpty()) {
            throw new IllegalArgumentException("消息内容不能为空");
        }
        if (content.length() > 10000) { // 调整为更合理的长度
            throw new IllegalArgumentException("消息内容过长");
        }
        if (sequenceNumber != null && sequenceNumber <= 0) {
            throw new IllegalArgumentException("序号必须大于0");
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
**数据库映射：对应sessions.status字段**

```java
@Getter
@AllArgsConstructor
public enum ConversationStateVO {
    ACTIVE("ACTIVE", "活跃中"),
    ARCHIVED("ARCHIVED", "已归档"),
    DELETED("DELETED", "已删除");
    
    private final String code;
    private final String desc;
    
    public boolean canSendMessage() {
        return this == ACTIVE;
    }
    
    public boolean canArchive() {
        return this == ACTIVE;
    }
    
    public static ConversationStateVO fromCode(String code) {
        for (ConversationStateVO state : values()) {
            if (state.getCode().equals(code)) {
                return state;
            }
        }
        throw new IllegalArgumentException("未知的对话状态: " + code);
    }
}
```

#### 5.2 消息角色值对象（MessageRoleVO）
**数据库映射：对应conversations.role字段**

```java
@Getter
@AllArgsConstructor
public enum MessageRoleVO {
    USER("user", "用户"),
    ASSISTANT("assistant", "AI助手"),
    SYSTEM("system", "系统"),
    FUNCTION("function", "函数调用");
    
    private final String code;
    private final String desc;
    
    public static MessageRoleVO fromCode(String code) {
        for (MessageRoleVO role : values()) {
            if (role.getCode().equals(code)) {
                return role;
            }
        }
        throw new IllegalArgumentException("未知的消息角色: " + code);
    }
}
```

#### 5.3 内容类型值对象（ContentTypeVO）
**数据库映射：对应conversations.content_type字段**

```java
@Getter
@AllArgsConstructor
public enum ContentTypeVO {
    TEXT("text", "文本"),
    IMAGE("image", "图片"),
    FILE("file", "文件"),
    CODE("code", "代码");
    
    private final String code;
    private final String desc;
    
    public static ContentTypeVO fromCode(String code) {
        for (ContentTypeVO type : values()) {
            if (type.getCode().equals(code)) {
                return type;
            }
        }
        throw new IllegalArgumentException("未知的内容类型: " + code);
    }
}
```

#### 5.4 消息状态值对象（MessageStatusVO）
**数据库映射：对应conversations.status字段**

```java
@Getter
@AllArgsConstructor
public enum MessageStatusVO {
    NORMAL("normal", "正常"),
    HIDDEN("hidden", "隐藏"),
    DELETED("deleted", "已删除");
    
    private final String code;
    private final String desc;
    
    public static MessageStatusVO fromCode(String code) {
        for (MessageStatusVO status : values()) {
            if (status.getCode().equals(status)) {
                return status;
            }
        }
        throw new IllegalArgumentException("未知的消息状态: " + code);
    }
}
```

#### 5.5 服务商类型值对象（ProviderTypeVO）
**数据库映射：对应ai_providers.provider_code字段**

```java
@Getter
@AllArgsConstructor
public enum ProviderTypeVO {
    OPENAI("openai", "OpenAI"),
    ANTHROPIC("anthropic", "Anthropic Claude"),
    GOOGLE("google", "Google AI"),
    ZHIPU("zhipu", "智谱AI");
    
    private final String code;
    private final String desc;
    
    public static ProviderTypeVO fromCode(String code) {
        for (ProviderTypeVO type : values()) {
            if (type.getCode().equals(code)) {
                return type;
            }
        }
        throw new IllegalArgumentException("未知的服务商类型: " + code);
    }
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

### 7. Repository接口设计（基于big-market模式）

#### 7.1 对话仓储接口（IConversationRepository）

```java
/**
 * 对话仓储接口 - 面向聚合的操作
 * 参考big-market的saveCreatePartakeOrderAggregate模式
 */
public interface IConversationRepository {
    
    /**
     * 保存创建对话聚合
     * 对应数据库操作：插入sessions表
     */
    void saveCreateConversationAggregate(CreateConversationAggregate createConversationAggregate);
    
    /**
     * 保存发送消息聚合
     * 对应数据库操作：
     * 1. 插入用户消息到conversations表
     * 2. 更新sessions表的统计信息
     * 3. 插入AI响应消息到conversations表（如果有）
     * 4. 再次更新sessions表的统计信息
     */
    void saveSendMessageAggregate(SendMessageAggregate sendMessageAggregate);
    
    /**
     * 保存归档对话聚合
     * 对应数据库操作：更新sessions表状态
     */
    void saveArchiveConversationAggregate(ArchiveConversationAggregate archiveConversationAggregate);
    
    /**
     * 查询对话聚合
     * 对应数据库操作：查询sessions表
     */
    ConversationAggregate queryConversationAggregate(String conversationId);
    
    /**
     * 分页查询用户对话列表
     */
    List<ConversationAggregate> queryUserConversations(String userId, int page, int size);
    
    /**
     * 分页查询对话消息
     * 对应数据库操作：查询conversations表
     */
    List<MessageEntity> queryConversationMessages(String conversationId, int page, int size);
    
    /**
     * 查询最新的N条消息（用于上下文构建）
     */
    List<MessageEntity> queryRecentMessages(String conversationId, int limit);
}
```

#### 7.2 聚合对象设计（面向事务的聚合）

##### 7.2.1 创建对话聚合
```java
/**
 * 创建对话聚合 - 对应“创建新对话”业务场景
 * 参考big-market的CreatePartakeOrderAggregate
 */
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class CreateConversationAggregate {
    // 聚合标识
    private String conversationId;
    private String userId;
    
    // 要创建的对话实体
    private ConversationEntity conversationEntity;
    
    // 创建信息
    private String createdBy;
    private Date createTime;
}
```

##### 7.2.2 发送消息聚合
```java
/**
 * 发送消息聚合 - 对应“发送消息并获取AI响应”业务场景
 * 参考big-market的CreatePartakeOrderAggregate
 */
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class SendMessageAggregate {
    // 聚合标识
    private String conversationId;
    private String userId;
    
    // 要更新的对话实体（统计信息）
    private ConversationEntity conversationEntity;
    
    // 用户消息实体（要插入）
    private MessageEntity userMessage;
    
    // AI响应消息实体（要插入，可能为空）
    private MessageEntity aiMessage;
    
    // 业务标识
    private boolean hasAIResponse = false;    // 是否有AI响应
    private boolean needUpdateStats = true;   // 是否需要更新统计信息
}
```

##### 7.2.3 归档对话聚合
```java
/**
 * 归档对话聚合 - 对应“归档对话”业务场景
 */
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ArchiveConversationAggregate {
    // 聚合标识
    private String conversationId;
    private String userId;
    
    // 要更新的对话实体
    private ConversationEntity conversationEntity;
    
    // 操作信息
    private Date archiveTime;
}
```

### 8. API层业务编排设计（参考big-market的draw方法）

#### 8.1 ConversationController - 发送消息接口

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
     * 采用聚合保存模式，一次性完成所有数据库操作
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
            
            // 3. 构建发送消息聚合 - 参考big-market的聚合构建模式
            SendMessageAggregate sendMessageAggregate = buildSendMessageAggregate(request, aiModel);
            
            // 4. 执行AI处理（调用chat领域服务）
            String aiResponse = chatProcessService.processMessage(
                sendMessageAggregate.getUserMessage().getContent(),
                request.getModelId(),
                request.getConversationId()
            );
            
            // 5. 填充AI响应到聚合中
            if (StringUtils.isNotBlank(aiResponse)) {
                MessageEntity aiMessage = buildAIMessage(sendMessageAggregate, aiResponse, aiModel);
                sendMessageAggregate.setAiMessage(aiMessage);
                sendMessageAggregate.setHasAIResponse(true);
            }
            
            // 6. 一次性保存整个聚合（事务边界）- 参考big-market模式
            conversationRepository.saveSendMessageAggregate(sendMessageAggregate);
            
            log.info("消息发送完成 userId:{} messageId:{} hasAIResponse:{}", 
                    request.getUserId(), 
                    sendMessageAggregate.getUserMessage().getMessageId(),
                    sendMessageAggregate.isHasAIResponse());
            
            // 7. 返回结果
            return Response.<MessageResponse>builder()
                .code(ResponseCode.SUCCESS.getCode())
                .info(ResponseCode.SUCCESS.getInfo())
                .data(buildMessageResponse(sendMessageAggregate))
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
     * 创建对话接口 - 采用聚合保存模式
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
            
            // 2. 构建创建对话聚合
            CreateConversationAggregate createConversationAggregate = buildCreateConversationAggregate(request);
            
            // 3. 一次性保存聚合（事务边界）
            conversationRepository.saveCreateConversationAggregate(createConversationAggregate);
            
            log.info("对话创建完成 userId:{} conversationId:{}", 
                    request.getUserId(), 
                    createConversationAggregate.getConversationId());
            
            // 4. 返回结果
            return Response.<ConversationResponse>builder()
                .code(ResponseCode.SUCCESS.getCode())
                .info(ResponseCode.SUCCESS.getInfo())
                .data(ConversationResponse.builder()
                    .conversationId(createConversationAggregate.getConversationId())
                    .title(request.getTitle())
                    .state(ConversationStateVO.ACTIVE.getCode())
                    .createTime(createConversationAggregate.getCreateTime())
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

### 9. 总结

#### 9.1 修正后的设计关键点

1. **概念统一**：
   - 数据库sessions表 ↔️ DDD的ConversationAggregate
   - 数据库conversations表 ↔️ DDD的MessageEntity
   - 字段映射关系完全一致

2. **枚举类型完善**：
   - MessageRoleVO添加FUNCTION角色支持
   - ConversationStateVO移除PAUSED状态，与数据库保持一致
   - 新增ContentTypeVO和MessageStatusVO

3. **聚合设计优化**：
   - ConversationAggregate添加模型配置、统计信息、功能开关
   - MessageEntity添加序号、内容类型、Token统计等字段
   - 移除不必要的复杂字段（parent_id, response_time_ms, user_rating等）

4. **Repository模式**：
   - 参考big-market的saveCreatePartakeOrderAggregate模式
   - 面向聚合的操作：saveCreateConversationAggregate、saveSendMessageAggregate
   - 确保事务边界和数据一致性

5. **API层优化**：
   - 采用聚合保存模式，一次性完成所有数据库操作
   - 遵循事务边界，确保数据一致性

#### 9.2 简化设计的关键点

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

#### 9.3 设计优势

- **架构清晰**：严格按照big-market的六边形架构，但去除不必要部分
- **职责分离**：聚合职责单一，API层负责业务编排
- **易于实现**：当前需求下的最小可行设计
- **扩展友好**：为后续功能扩展预留空间

#### 9.4 后续实施建议

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