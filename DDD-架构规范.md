# AgentU 领域驱动设计(DDD)架构规范

## 1. DDD概述与原则

### 1.1 什么是轻量级DDD
轻量级DDD专注于DDD的核心价值：**通用语言(Ubiquitous Language)**和**界限上下文(Bounded Context)**，避免过度设计和复杂的战术模式。

### 1.2 核心原则
- **业务优先**：技术服务于业务，而非相反
- **通用语言**：业务专家和开发者使用一致的术语
- **界限上下文**：明确的业务边界和职责划分
- **持久化无关**：领域层不依赖具体的数据存储技术
- **基础设施无关**：核心业务逻辑与基础设施解耦

## 2. 架构分层设计

### 2.1 四层架构
```
┌─────────────────────────────────────────┐
│          表现层 (Presentation)            │  API控制器、DTO、请求验证
├─────────────────────────────────────────┤
│           应用层 (Application)            │  用例编排、事务管理、安全控制
├─────────────────────────────────────────┤
│            领域层 (Domain)               │  业务逻辑、领域模型、业务规则
├─────────────────────────────────────────┤
│          基础设施层 (Infrastructure)       │  数据持久化、外部服务、技术实现
└─────────────────────────────────────────┘
```

### 2.2 各层职责

#### 表现层 (Presentation Layer)
- **职责**：处理HTTP请求/响应，数据格式转换，输入验证
- **组件**：Controller、DTO、RequestValidator、ExceptionHandler
- **依赖方向**：只能依赖应用层

#### 应用层 (Application Layer)
- **职责**：用例编排、事务边界、权限控制、领域服务协调
- **组件**：ApplicationService、UseCase、Command/Query、EventHandler
- **依赖方向**：依赖领域层，不依赖基础设施层

#### 领域层 (Domain Layer)
- **职责**：核心业务逻辑、业务规则、领域模型
- **组件**：Entity、ValueObject、DomainService、Repository接口、DomainEvent
- **依赖方向**：不依赖任何其他层，完全独立

#### 基础设施层 (Infrastructure Layer)
- **职责**：技术实现、数据持久化、外部系统集成
- **组件**：RepositoryImpl、ExternalService、MessageQueue、Database配置
- **依赖方向**：实现领域层定义的接口

## 3. AgentU界限上下文设计

### 3.1 上下文映射图
```
┌─────────────────┐    协作    ┌─────────────────┐
│   用户上下文     │ ◄─────────► │   对话上下文     │
│ User Context    │            │Conversation Ctx │
└─────────────────┘            └─────────────────┘
        │                              │
        │ 鉴权                         │ 检索
        ▼                              ▼
┌─────────────────┐    调用    ┌─────────────────┐
│   工具上下文     │ ◄─────────► │   知识库上下文   │
│  Tool Context   │            │Knowledge Ctx    │
└─────────────────┘            └─────────────────┘
```

### 3.2 界限上下文详细设计

#### 3.2.1 用户上下文 (User Context)
**业务职责**：用户身份认证、权限管理、API配置管理

**核心概念**：
- `User` - 用户实体
- `Role` - 角色
- `Permission` - 权限
- `ApiConfig` - API配置

**主要用例**：
- 用户注册/登录
- 角色权限管理
- API Key配置
- 使用配额管理

#### 3.2.2 对话上下文 (Conversation Context)
**业务职责**：AI对话管理、会话状态、历史记录

**核心概念**：
- `Conversation` - 对话会话
- `Message` - 消息
- `AiModel` - AI模型
- `ChatHistory` - 对话历史

**主要用例**：
- 创建对话会话
- 发送/接收消息
- 管理对话历史
- 模型切换

#### 3.2.3 知识库上下文 (Knowledge Context)
**业务职责**：文档管理、向量化存储、语义检索

**核心概念**：
- `KnowledgeBase` - 知识库
- `Document` - 文档
- `DocumentChunk` - 文档片段
- `VectorIndex` - 向量索引

**主要用例**：
- 上传文档
- 文档分割处理
- 向量化存储
- 语义检索

#### 3.2.4 工具上下文 (Tool Context)
**业务职责**：MCP工具管理、工具调用执行

**核心概念**：
- `McpTool` - MCP工具
- `ToolExecution` - 工具执行
- `ToolMarket` - 工具市场
- `ToolPermission` - 工具权限

**主要用例**：
- 工具注册/发现
- 工具调用执行
- 工具权限控制
- 执行结果处理

## 4. 领域模型设计规范

### 4.1 实体(Entity)设计
```java
// 示例：对话实体
@Entity
public class Conversation {
    private ConversationId id;           // 唯一标识
    private UserId userId;               // 所属用户
    private String title;                // 对话标题
    private ConversationStatus status;   // 对话状态
    private LocalDateTime createdAt;     // 创建时间
    private LocalDateTime updatedAt;     // 更新时间
    
    // 业务行为
    public void addMessage(Message message) { /* 业务逻辑 */ }
    public void archive() { /* 业务逻辑 */ }
    public boolean canAddMessage() { /* 业务规则 */ }
}
```

### 4.2 值对象(Value Object)设计
```java
// 示例：对话ID值对象
public record ConversationId(String value) {
    public ConversationId {
        Objects.requireNonNull(value);
        if (value.trim().isEmpty()) {
            throw new IllegalArgumentException("ConversationId不能为空");
        }
    }
    
    public static ConversationId generate() {
        return new ConversationId(UUID.randomUUID().toString());
    }
}
```

### 4.3 领域服务(Domain Service)设计
```java
// 示例：RAG检索领域服务
@DomainService
public class RagRetrievalService {
    
    public RetrievalResult retrieveRelevantChunks(
            KnowledgeBaseId knowledgeBaseId, 
            String query, 
            RetrievalStrategy strategy) {
        // 复杂的检索逻辑
        // 涉及多个聚合的协调
    }
}
```

### 4.4 仓储(Repository)接口设计
```java
// 领域层定义接口
public interface ConversationRepository {
    ConversationId save(Conversation conversation);
    Optional<Conversation> findById(ConversationId id);
    List<Conversation> findByUserId(UserId userId);
    void delete(ConversationId id);
}

// 基础设施层实现
@Repository
public class JpaConversationRepository implements ConversationRepository {
    // JPA实现
}
```

## 5. 应用服务设计规范

### 5.1 应用服务职责
- 用例编排和协调
- 事务边界管理
- 安全控制
- 异常处理
- 领域事件发布

### 5.2 应用服务示例
```java
@ApplicationService
@Transactional
public class ConversationApplicationService {
    
    private final ConversationRepository conversationRepo;
    private final AiModelService aiModelService;
    private final EventPublisher eventPublisher;
    
    public SendMessageResponse sendMessage(SendMessageCommand command) {
        // 1. 权限检查
        checkPermission(command.getUserId(), command.getConversationId());
        
        // 2. 加载聚合
        Conversation conversation = conversationRepo.findById(command.getConversationId())
            .orElseThrow(() -> new ConversationNotFound(command.getConversationId()));
        
        // 3. 执行业务逻辑
        Message userMessage = Message.create(command.getContent(), MessageType.USER);
        conversation.addMessage(userMessage);
        
        // 4. AI处理
        String aiResponse = aiModelService.generateResponse(conversation.getHistory());
        Message aiMessage = Message.create(aiResponse, MessageType.AI);
        conversation.addMessage(aiMessage);
        
        // 5. 持久化
        conversationRepo.save(conversation);
        
        // 6. 发布事件
        eventPublisher.publish(new MessageSentEvent(conversation.getId(), aiMessage));
        
        return new SendMessageResponse(aiMessage);
    }
}
```

## 6. 事件驱动设计

### 6.1 领域事件
```java
// 领域事件
public record DocumentProcessedEvent(
    DocumentId documentId,
    KnowledgeBaseId knowledgeBaseId,
    int chunkCount,
    LocalDateTime processedAt
) implements DomainEvent {}
```

### 6.2 事件处理
```java
@EventHandler
public class DocumentProcessedEventHandler {
    
    @EventListener
    public void handle(DocumentProcessedEvent event) {
        // 更新知识库统计
        // 通知用户处理完成
        // 触发向量索引构建
    }
}
```

## 7. 依赖注入与配置

### 7.1 Spring配置类
```java
@Configuration
public class DomainConfiguration {
    
    @Bean
    public RagRetrievalService ragRetrievalService(
            VectorStoreService vectorStore,
            EmbeddingService embedding) {
        return new RagRetrievalService(vectorStore, embedding);
    }
    
    @Bean
    public ConversationApplicationService conversationService(
            ConversationRepository repository,
            AiModelService aiModel,
            EventPublisher eventPublisher) {
        return new ConversationApplicationService(repository, aiModel, eventPublisher);
    }
}
```

## 8. 测试策略

### 8.1 测试金字塔
```
           ┌─────────────┐
           │   E2E Tests │  少量，关键业务流程
           ├─────────────┤
           │ Integration │  中量，层间集成
           │    Tests    │
           ├─────────────┤
           │   Unit      │  大量，核心业务逻辑
           │   Tests     │
           └─────────────┘
```

### 8.2 单元测试示例
```java
class ConversationTest {
    
    @Test
    void shouldAddMessage() {
        // Given
        Conversation conversation = Conversation.create(userId, "Test Chat");
        Message message = Message.create("Hello", MessageType.USER);
        
        // When
        conversation.addMessage(message);
        
        // Then
        assertThat(conversation.getMessageCount()).isEqualTo(1);
        assertThat(conversation.getLastMessage()).isEqualTo(message);
    }
    
    @Test
    void shouldThrowExceptionWhenAddingMessageToArchivedConversation() {
        // Given
        Conversation conversation = Conversation.create(userId, "Test Chat");
        conversation.archive();
        
        // When & Then
        assertThatThrownBy(() -> conversation.addMessage(message))
            .isInstanceOf(ConversationArchivedException.class);
    }
}
```

## 9. 包结构规范

```
com.agentu
├── user/                           # 用户上下文
│   ├── domain/
│   │   ├── model/                  # 领域模型
│   │   ├── service/                # 领域服务
│   │   └── repository/             # 仓储接口
│   ├── application/                # 应用服务
│   ├── infrastructure/             # 基础设施实现
│   └── presentation/               # 表现层
├── conversation/                   # 对话上下文
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── presentation/
├── knowledge/                      # 知识库上下文
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── presentation/
├── tool/                          # 工具上下文
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── presentation/
└── shared/                        # 共享内核
    ├── domain/                    # 共享领域概念
    ├── application/               # 共享应用组件
    └── infrastructure/            # 共享基础设施
```

## 10. 数据库设计原则

### 10.1 聚合边界即事务边界
- 每个聚合根对应一个事务
- 跨聚合操作通过事件最终一致性

### 10.2 表设计规范
```sql
-- 对话表
CREATE TABLE conversations (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    title VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    version BIGINT NOT NULL DEFAULT 1,  -- 乐观锁
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- 消息表
CREATE TABLE messages (
    id VARCHAR(36) PRIMARY KEY,
    conversation_id VARCHAR(36) NOT NULL,
    content TEXT NOT NULL,
    message_type VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (conversation_id) REFERENCES conversations(id)
);
```

## 11. 监控与可观测性

### 11.1 领域指标监控
- 对话创建数量
- 消息发送频率
- 知识库查询性能
- 工具调用成功率

### 11.2 业务事件追踪
```java
@Component
public class BusinessMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void onMessageSent(MessageSentEvent event) {
        meterRegistry.counter("conversation.message.sent",
            "user_id", event.getUserId().value(),
            "model_type", event.getModelType().name())
            .increment();
    }
}
```

## 12. 演进策略

### 12.1 逐步重构原则
1. **陌生人模式**：新功能使用DDD，旧代码保持现状
2. **防腐层模式**：新旧代码间建立适配层
3. **绞杀者模式**：逐步用新实现替换旧代码

### 12.2 架构演进路径
```
Phase 1: 建立基础 → Phase 2: 完善模型 → Phase 3: 优化性能 → Phase 4: 扩展生态
```

---

*规范版本：v1.0*  
*更新时间：2025-07-22*  
*维护团队：架构组*

## 附录A：参考资料
- Eric Evans《Domain-Driven Design》
- Vaughn Vernon《Implementing Domain-Driven Design》
- Martin Fowler《Microservices》
- Clean Architecture by Robert Martin

## 附录B：工具推荐
- **建模工具**：PlantUML、Miro、EventStorming
- **代码生成**：JHipster Domain Language
- **测试工具**：TestContainers、WireMock
- **监控工具**：Micrometer、Zipkin