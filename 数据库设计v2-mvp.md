# AgentU 数据库设计 v2.0 - MVP版本

## 概述

基于v1.0版本的优化改进，专注MVP核心功能，保持设计简洁实用。主要改进数据完整性、基础安全性和必要的性能优化。

## 核心改进点

1. **数据完整性**：添加外键约束、检查约束
2. **基础安全**：密码安全、基础审计
3. **性能优化**：关键索引优化
4. **代码质量**：统一数据类型、规范命名

## MVP核心表结构

### 1. users - 用户表（优化）

```sql
CREATE TABLE users (
    -- 主键
    user_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 基本信息
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    salt VARCHAR(32) NOT NULL,
    
    -- 用户信息
    nickname VARCHAR(100),
    avatar_url VARCHAR(500),
    phone VARCHAR(20),
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    email_verified BOOLEAN DEFAULT FALSE,
    phone_verified BOOLEAN DEFAULT FALSE,
    
    -- 安全增强
    login_attempts INT DEFAULT 0,
    locked_until TIMESTAMP,
    
    -- 偏好设置
    language VARCHAR(10) DEFAULT 'zh-CN',
    timezone VARCHAR(50) DEFAULT 'Asia/Shanghai',
    theme VARCHAR(20) DEFAULT 'light',
    
    -- 配额信息
    monthly_token_limit BIGINT DEFAULT 1000000,
    monthly_token_used BIGINT DEFAULT 0,
    quota_reset_date DATE,
    
    -- 登录信息
    last_login_at TIMESTAMP,
    last_login_ip INET,
    login_count INT DEFAULT 0,
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    
    -- 检查约束
    CONSTRAINT chk_user_status CHECK (status IN ('ACTIVE', 'INACTIVE', 'SUSPENDED')),
    CONSTRAINT chk_email_format CHECK (email ~* '^[A-Za-z0-9._%-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    CONSTRAINT chk_login_attempts CHECK (login_attempts >= 0)
);

-- 索引
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_login ON users(last_login_at);
```

### 2. ai_providers - AI服务提供商表（优化）

```sql
CREATE TABLE ai_providers (
    -- 主键
    provider_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 基本信息
    provider_name VARCHAR(100) NOT NULL UNIQUE,
    provider_code VARCHAR(50) NOT NULL UNIQUE,
    display_name VARCHAR(100) NOT NULL,
    description TEXT,
    
    -- 配置信息
    base_url VARCHAR(500) NOT NULL,
    api_version VARCHAR(20),
    default_headers JSONB DEFAULT '{}',
    
    -- 速率限制配置
    rate_limit_config JSONB DEFAULT '{
        "requests_per_minute": 60,
        "tokens_per_minute": 100000
    }',
    
    -- 超时配置
    timeout_seconds INT DEFAULT 60,
    max_retries INT DEFAULT 3,
    
    -- 服务类型（新增）
    service_type VARCHAR(20) NOT NULL DEFAULT 'USER_PROVIDED',  -- USER_PROVIDED, OFFICIAL_FREE, OFFICIAL_PAID
    
    -- 官方服务配置（当service_type为OFFICIAL_*时使用）
    official_api_key TEXT,  -- 官方服务的API密钥（加密存储）
    free_quota_per_user_monthly BIGINT DEFAULT 0,  -- 每用户每月免费配额（tokens）
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    is_system_default BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    -- 检查约束
    CONSTRAINT chk_provider_status CHECK (status IN ('ACTIVE', 'INACTIVE', 'MAINTENANCE')),
    CONSTRAINT chk_service_type CHECK (service_type IN ('USER_PROVIDED', 'OFFICIAL_FREE', 'OFFICIAL_PAID')),
    CONSTRAINT chk_timeout CHECK (timeout_seconds > 0),
    CONSTRAINT chk_retries CHECK (max_retries >= 0)
);

-- 索引
CREATE INDEX idx_ai_providers_code ON ai_providers(provider_code);
CREATE INDEX idx_ai_providers_status ON ai_providers(status);
CREATE INDEX idx_ai_providers_service_type ON ai_providers(service_type);
CREATE UNIQUE INDEX idx_ai_providers_default ON ai_providers(is_system_default) WHERE is_system_default = TRUE;
```

### 3. ai_models - AI模型表（优化）

```sql
CREATE TABLE ai_models (
    -- 主键
    model_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 关联信息
    provider_id VARCHAR(36) NOT NULL,
    
    -- 模型信息
    model_name VARCHAR(100) NOT NULL,
    model_code VARCHAR(100) NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    description TEXT,
    version VARCHAR(50),
    
    -- 模型规格
    context_window INT DEFAULT 4096,
    max_tokens INT DEFAULT 4096,
    input_price_per_1m_tokens DECIMAL(10,4) DEFAULT 0,
    output_price_per_1m_tokens DECIMAL(10,4) DEFAULT 0,
    
    -- 能力标识
    supports_streaming BOOLEAN DEFAULT TRUE,
    supports_function_calling BOOLEAN DEFAULT FALSE,
    supports_vision BOOLEAN DEFAULT FALSE,
    supports_json_mode BOOLEAN DEFAULT FALSE,
    
    -- 参数范围
    temperature_min DECIMAL(3,2) DEFAULT 0,
    temperature_max DECIMAL(3,2) DEFAULT 2,
    temperature_default DECIMAL(3,2) DEFAULT 0.7,
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    is_recommended BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    -- 外键约束已移除（MVP版本不使用外键）
    
    -- 检查约束
    CONSTRAINT chk_model_status CHECK (status IN ('ACTIVE', 'DEPRECATED', 'BETA')),
    CONSTRAINT chk_model_specs CHECK (context_window > 0 AND max_tokens > 0),
    CONSTRAINT chk_model_prices CHECK (input_price_per_1m_tokens >= 0 AND output_price_per_1m_tokens >= 0),
    CONSTRAINT chk_temperature_range CHECK (temperature_min <= temperature_default AND temperature_default <= temperature_max),
    
    -- 唯一约束
    CONSTRAINT uk_ai_models_provider_code UNIQUE (provider_id, model_code)
);

-- 索引
CREATE INDEX idx_ai_models_provider ON ai_models(provider_id);
CREATE INDEX idx_ai_models_status ON ai_models(status);
CREATE INDEX idx_ai_models_recommended ON ai_models(is_recommended, sort_order);
```

### 4. user_api_keys - 用户API密钥表（优化）

```sql
CREATE TABLE user_api_keys (
    -- 主键
    api_key_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 关联信息
    user_id VARCHAR(36) NOT NULL,
    provider_id VARCHAR(36) NOT NULL,
    
    -- 密钥信息
    api_key_name VARCHAR(100) NOT NULL,
    encrypted_api_key TEXT,  -- 用户自己的API密钥（官方服务时可为空）
    key_prefix VARCHAR(20),
    
    -- 服务类型标识（新增）
    key_type VARCHAR(20) NOT NULL DEFAULT 'USER_KEY',  -- USER_KEY, OFFICIAL_FREE, OFFICIAL_PAID
    
    -- 配置信息
    is_default BOOLEAN DEFAULT FALSE,
    priority INT DEFAULT 1,
    
    -- 使用统计（简化）
    total_requests INT DEFAULT 0,
    total_tokens BIGINT DEFAULT 0,
    last_used_at TIMESTAMP,
    
    -- 官方服务配额使用情况（仅当key_type为OFFICIAL_*时使用）
    monthly_free_quota_used BIGINT DEFAULT 0,  -- 当月已使用的免费配额
    quota_reset_date DATE,  -- 配额重置日期
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(36) NOT NULL,
    
    -- 外键约束已移除（MVP版本不使用外键）
    
    -- 检查约束
    CONSTRAINT chk_api_key_status CHECK (status IN ('ACTIVE', 'INACTIVE', 'EXPIRED')),
    CONSTRAINT chk_key_type CHECK (key_type IN ('USER_KEY', 'OFFICIAL_FREE', 'OFFICIAL_PAID')),
    
    -- 唯一约束：每个用户每个提供商只能有一个默认密钥
    CONSTRAINT uk_api_keys_user_provider_default UNIQUE (user_id, provider_id, is_default) DEFERRABLE INITIALLY DEFERRED
);

-- 索引
CREATE INDEX idx_user_api_keys_user ON user_api_keys(user_id);
CREATE INDEX idx_user_api_keys_provider ON user_api_keys(provider_id);
CREATE INDEX idx_user_api_keys_status ON user_api_keys(user_id, status);
CREATE INDEX idx_user_api_keys_type ON user_api_keys(key_type);
CREATE INDEX idx_user_api_keys_quota ON user_api_keys(user_id, key_type, quota_reset_date) WHERE key_type LIKE 'OFFICIAL_%';
```

### 5. sessions - 会话表（优化）

```sql
CREATE TABLE sessions (
    -- 主键
    session_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 关联信息
    user_id VARCHAR(36) NOT NULL,
    
    -- 会话信息
    title VARCHAR(200),
    description TEXT,
    
    -- 模型配置
    model_id VARCHAR(36),
    provider_id VARCHAR(36),
    api_key_id VARCHAR(36),
    
    -- 参数设置
    temperature DECIMAL(3,2) DEFAULT 0.7,
    max_tokens INT,
    top_p DECIMAL(3,2),
    frequency_penalty DECIMAL(3,2),
    presence_penalty DECIMAL(3,2),
    
    -- 功能开关
    enable_knowledge_base BOOLEAN DEFAULT FALSE,
    knowledge_base_ids JSONB DEFAULT '[]',
    enable_function_calling BOOLEAN DEFAULT FALSE,
    
    -- 统计信息
    message_count INT DEFAULT 0,
    total_input_tokens BIGINT DEFAULT 0,
    total_output_tokens BIGINT DEFAULT 0,
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    is_pinned BOOLEAN DEFAULT FALSE,
    last_message_at TIMESTAMP,
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(36) NOT NULL,
    
    -- 外键约束已移除（MVP版本不使用外键）
    
    -- 检查约束
    CONSTRAINT chk_session_status CHECK (status IN ('ACTIVE', 'ARCHIVED', 'DELETED')),
    CONSTRAINT chk_parameters CHECK (
        (temperature IS NULL OR temperature BETWEEN 0 AND 2) AND
        (top_p IS NULL OR top_p BETWEEN 0 AND 1) AND
        (frequency_penalty IS NULL OR frequency_penalty BETWEEN -2 AND 2) AND
        (presence_penalty IS NULL OR presence_penalty BETWEEN -2 AND 2)
    )
);

-- 索引
CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_status ON sessions(user_id, status);
CREATE INDEX idx_sessions_last_message ON sessions(user_id, last_message_at DESC);
CREATE INDEX idx_sessions_pinned ON sessions(user_id, is_pinned) WHERE is_pinned = TRUE;
```

### 6. conversations - 对话记录表（优化）

```sql
CREATE TABLE conversations (
    -- 主键
    conversation_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 关联信息
    session_id VARCHAR(36) NOT NULL,
    user_id VARCHAR(36) NOT NULL,
    
    -- 对话信息
    role VARCHAR(20) NOT NULL,
    content TEXT NOT NULL,
    content_type VARCHAR(20) DEFAULT 'text',
    
    -- 序号信息
    sequence_number INT NOT NULL,
    parent_id VARCHAR(36),
    
    -- 模型调用信息（仅AI回复）
    model_id VARCHAR(36),
    provider_id VARCHAR(36),
    model_name VARCHAR(100),
    
    -- Token统计（仅AI回复）
    input_tokens INT DEFAULT 0,
    output_tokens INT DEFAULT 0,
    total_tokens INT DEFAULT 0,
    
    -- 性能指标（仅AI回复）
    response_time_ms INT DEFAULT 0,
    
    -- 扩展信息
    metadata JSONB DEFAULT '{}',
    attachments JSONB DEFAULT '[]',
    
    -- 反馈信息
    user_rating INT CHECK (user_rating BETWEEN 1 AND 5),
    user_feedback TEXT,
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'normal',
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    -- 外键约束已移除（MVP版本不使用外键）
    
    -- 检查约束
    CONSTRAINT chk_conversation_role CHECK (role IN ('system', 'user', 'assistant', 'function')),
    CONSTRAINT chk_conversation_status CHECK (status IN ('normal', 'hidden', 'deleted')),
    CONSTRAINT chk_content_type CHECK (content_type IN ('text', 'image', 'file', 'code'))
);

-- 索引
CREATE INDEX idx_conversations_session ON conversations(session_id, sequence_number);
CREATE INDEX idx_conversations_user ON conversations(user_id);
CREATE INDEX idx_conversations_role ON conversations(session_id, role);
CREATE INDEX idx_conversations_created_at ON conversations(created_at);
CREATE INDEX idx_conversations_parent ON conversations(parent_id);
```

### 7. knowledge_bases - 知识库表（优化）

```sql
CREATE TABLE knowledge_bases (
    -- 主键
    kb_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 关联信息
    user_id VARCHAR(36) NOT NULL,
    
    -- 基本信息
    kb_name VARCHAR(200) NOT NULL,
    description TEXT,
    
    -- 配置信息
    embedding_model VARCHAR(100) DEFAULT 'text-embedding-ada-002',
    chunk_size INT DEFAULT 1000,
    chunk_overlap INT DEFAULT 200,
    vector_dimension INT DEFAULT 1536,
    
    -- 统计信息
    document_count INT DEFAULT 0,
    total_chunks INT DEFAULT 0,
    total_size_bytes BIGINT DEFAULT 0,
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    is_public BOOLEAN DEFAULT FALSE,
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(36) NOT NULL,
    
    -- 外键约束已移除（MVP版本不使用外键）
    
    -- 检查约束
    CONSTRAINT chk_kb_status CHECK (status IN ('ACTIVE', 'INDEXING', 'ERROR', 'ARCHIVED')),
    CONSTRAINT chk_chunk_config CHECK (chunk_size > 0 AND chunk_overlap >= 0 AND chunk_overlap < chunk_size),
    
    -- 唯一约束
    CONSTRAINT uk_knowledge_bases_user_name UNIQUE (user_id, kb_name)
);

-- 索引
CREATE INDEX idx_knowledge_bases_user ON knowledge_bases(user_id);
CREATE INDEX idx_knowledge_bases_status ON knowledge_bases(status);
CREATE INDEX idx_knowledge_bases_public ON knowledge_bases(is_public);
```

### 8. documents - 文档表（优化）

```sql
CREATE TABLE documents (
    -- 主键
    document_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 关联信息
    kb_id VARCHAR(36) NOT NULL,
    user_id VARCHAR(36) NOT NULL,
    
    -- 文档信息
    file_name VARCHAR(255) NOT NULL,
    original_name VARCHAR(255) NOT NULL,
    file_type VARCHAR(50) NOT NULL,
    file_size BIGINT NOT NULL,
    file_path VARCHAR(1000),
    file_url VARCHAR(1000),
    
    -- 内容信息
    title VARCHAR(500),
    content_preview TEXT,
    total_pages INT,
    total_characters INT,
    
    -- 处理状态
    processing_status VARCHAR(20) DEFAULT 'pending',
    processing_progress INT DEFAULT 0 CHECK (processing_progress BETWEEN 0 AND 100),
    processing_error TEXT,
    
    -- 分块信息
    chunk_count INT DEFAULT 0,
    embedding_status VARCHAR(20) DEFAULT 'pending',
    
    -- 元数据
    metadata JSONB DEFAULT '{}',
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(36) NOT NULL,
    
    -- 外键约束已移除（MVP版本不使用外键）
    
    -- 检查约束
    CONSTRAINT chk_document_status CHECK (status IN ('ACTIVE', 'PROCESSING', 'ERROR', 'DELETED')),
    CONSTRAINT chk_processing_status CHECK (processing_status IN ('pending', 'processing', 'completed', 'failed')),
    CONSTRAINT chk_embedding_status CHECK (embedding_status IN ('pending', 'processing', 'completed', 'failed')),
    CONSTRAINT chk_file_size CHECK (file_size > 0)
);

-- 索引
CREATE INDEX idx_documents_kb ON documents(kb_id);
CREATE INDEX idx_documents_user ON documents(user_id);
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_processing ON documents(processing_status);
CREATE INDEX idx_documents_embedding ON documents(embedding_status);
CREATE INDEX idx_documents_file_type ON documents(file_type);
```

### 9. document_chunks - 文档分块表（优化）

```sql
CREATE TABLE document_chunks (
    -- 主键
    chunk_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 关联信息
    document_id VARCHAR(36) NOT NULL,
    kb_id VARCHAR(36) NOT NULL,
    
    -- 分块信息
    chunk_index INT NOT NULL,
    content TEXT NOT NULL,
    content_length INT NOT NULL,
    
    -- 位置信息
    start_page INT,
    end_page INT,
    start_offset INT,
    end_offset INT,
    
    -- 向量信息
    embedding VECTOR(1536),
    embedding_model VARCHAR(100),
    
    -- 元数据
    metadata JSONB DEFAULT '{}',
    
    -- 统计信息
    search_count INT DEFAULT 0,
    last_searched_at TIMESTAMP,
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    -- 外键约束已移除（MVP版本不使用外键）
    
    -- 检查约束
    CONSTRAINT chk_content_length CHECK (content_length > 0),
    
    -- 唯一约束
    CONSTRAINT uk_document_chunks_index UNIQUE (document_id, chunk_index)
);

-- 索引
CREATE INDEX idx_document_chunks_document ON document_chunks(document_id);
CREATE INDEX idx_document_chunks_kb ON document_chunks(kb_id);
CREATE INDEX idx_document_chunks_index ON document_chunks(document_id, chunk_index);

-- 向量相似度搜索索引（需要pgvector扩展）
CREATE INDEX idx_document_chunks_embedding ON document_chunks 
USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

### 10. system_configs - 系统配置表（简化）

```sql
CREATE TABLE system_configs (
    -- 主键
    config_id VARCHAR(36) NOT NULL PRIMARY KEY,
    
    -- 配置信息
    config_key VARCHAR(100) NOT NULL UNIQUE,
    config_value TEXT,
    config_type VARCHAR(20) NOT NULL DEFAULT 'string',
    
    -- 描述信息
    display_name VARCHAR(200),
    description TEXT,
    group_name VARCHAR(100),
    
    -- 约束信息
    is_required BOOLEAN DEFAULT FALSE,
    is_encrypted BOOLEAN DEFAULT FALSE,
    
    -- 状态管理
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    
    -- 审计字段
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by VARCHAR(36),
    
    -- 检查约束
    CONSTRAINT chk_config_status CHECK (status IN ('ACTIVE', 'INACTIVE')),
    CONSTRAINT chk_config_type CHECK (config_type IN ('string', 'integer', 'decimal', 'boolean', 'json'))
);

-- 索引
CREATE INDEX idx_system_configs_key ON system_configs(config_key);
CREATE INDEX idx_system_configs_group ON system_configs(group_name);
CREATE INDEX idx_system_configs_status ON system_configs(status);
```

## 基础触发器

### 1. 更新时间戳触发器

```sql
-- 通用的updated_at触发器函数
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 为主要表创建触发器
CREATE TRIGGER trigger_users_updated_at 
    BEFORE UPDATE ON users 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trigger_sessions_updated_at 
    BEFORE UPDATE ON sessions 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trigger_conversations_updated_at 
    BEFORE UPDATE ON conversations 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trigger_knowledge_bases_updated_at 
    BEFORE UPDATE ON knowledge_bases 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trigger_documents_updated_at 
    BEFORE UPDATE ON documents 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 2. 配额检查函数（简化版）

```sql
-- 简单的token配额检查
CREATE OR REPLACE FUNCTION check_user_token_quota()
RETURNS TRIGGER AS $$
DECLARE
    user_limit BIGINT;
    user_used BIGINT;
BEGIN
    -- 仅在AI回复时检查
    IF NEW.role != 'assistant' THEN
        RETURN NEW;
    END IF;
    
    -- 获取用户配额信息
    SELECT monthly_token_limit, monthly_token_used 
    INTO user_limit, user_used
    FROM users 
    WHERE user_id = NEW.user_id;
    
    -- 检查是否超出配额
    IF user_used + NEW.total_tokens > user_limit THEN
        RAISE EXCEPTION 'Monthly token quota exceeded for user %', NEW.user_id;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 在conversations表上创建触发器
CREATE TRIGGER trigger_check_token_quota
    BEFORE INSERT ON conversations
    FOR EACH ROW EXECUTE FUNCTION check_user_token_quota();
```

## 初始化数据

### 1. AI服务提供商数据

```sql
INSERT INTO ai_providers (provider_id, provider_name, provider_code, display_name, description, base_url, service_type, free_quota_per_user_monthly, status, is_system_default, sort_order) VALUES
-- 用户自备API密钥的服务
('openai-001', 'OpenAI', 'openai', 'OpenAI', 'OpenAI GPT models - 用户自备API密钥', 'https://api.openai.com', 'USER_PROVIDED', 0, 'ACTIVE', FALSE, 1),
('anthropic-001', 'Anthropic', 'anthropic', 'Anthropic', 'Claude models - 用户自备API密钥', 'https://api.anthropic.com', 'USER_PROVIDED', 0, 'ACTIVE', FALSE, 2),
('google-001', 'Google', 'google', 'Google AI', 'Gemini models - 用户自备API密钥', 'https://generativelanguage.googleapis.com', 'USER_PROVIDED', 0, 'ACTIVE', FALSE, 3),
('zhipu-001', '智谱AI', 'zhipu', '智谱AI', 'GLM models - 用户自备API密钥', 'https://open.bigmodel.cn', 'USER_PROVIDED', 0, 'ACTIVE', FALSE, 4),

-- 官方免费服务（使用平台API密钥，为用户提供免费额度）
('openai-official-001', 'OpenAI官方服务', 'openai-official', 'OpenAI (官方免费)', 'OpenAI GPT models - 平台提供免费额度', 'https://api.openai.com', 'OFFICIAL_FREE', 50000, 'ACTIVE', TRUE, 10),
('anthropic-official-001', 'Claude官方服务', 'anthropic-official', 'Claude (官方免费)', 'Claude models - 平台提供免费额度', 'https://api.anthropic.com', 'OFFICIAL_FREE', 30000, 'ACTIVE', FALSE, 11),

-- 官方付费服务（用户付费使用平台API密钥，通常价格更优惠）
('openai-premium-001', 'OpenAI高级服务', 'openai-premium', 'OpenAI (高级付费)', 'OpenAI GPT models - 平台优惠价格', 'https://api.openai.com', 'OFFICIAL_PAID', 0, 'ACTIVE', FALSE, 20);
```

### 2. AI模型数据

```sql
INSERT INTO ai_models (model_id, provider_id, model_name, model_code, display_name, description, context_window, max_tokens, input_price_per_1m_tokens, output_price_per_1m_tokens, supports_streaming, supports_function_calling, is_recommended, sort_order) VALUES
-- 用户自备API密钥的模型
('gpt-4-001', 'openai-001', 'gpt-4', 'gpt-4', 'GPT-4 (自备密钥)', 'Most capable GPT-4 model', 8192, 4096, 30.0, 60.0, TRUE, TRUE, TRUE, 1),
('gpt-35-001', 'openai-001', 'gpt-3.5-turbo', 'gpt-3.5-turbo', 'GPT-3.5 Turbo (自备密钥)', 'Fast and efficient model', 16385, 4096, 1.5, 2.0, TRUE, TRUE, TRUE, 2),
('claude3-sonnet-001', 'anthropic-001', 'claude-3-sonnet-20240229', 'claude-3-sonnet', 'Claude 3 Sonnet (自备密钥)', 'Balanced Claude model', 200000, 4096, 3.0, 15.0, TRUE, TRUE, TRUE, 3),
('claude3-haiku-001', 'anthropic-001', 'claude-3-haiku-20240307', 'claude-3-haiku', 'Claude 3 Haiku (自备密钥)', 'Fastest Claude model', 200000, 4096, 0.25, 1.25, TRUE, TRUE, FALSE, 4),
('gemini-pro-001', 'google-001', 'gemini-pro', 'gemini-pro', 'Gemini Pro (自备密钥)', 'Google\'s capable model', 32768, 8192, 0.5, 1.5, TRUE, TRUE, TRUE, 5),
('glm-4-001', 'zhipu-001', 'glm-4', 'glm-4', 'GLM-4 (自备密钥)', '智谱最新一代大模型', 32768, 4096, 1.0, 2.0, TRUE, TRUE, TRUE, 6),

-- 官方免费服务的模型
('gpt-35-official-001', 'openai-official-001', 'gpt-3.5-turbo', 'gpt-3.5-turbo', 'GPT-3.5 Turbo (官方免费)', 'Fast and efficient model - 免费使用', 16385, 4096, 0.0, 0.0, TRUE, TRUE, TRUE, 10),
('claude3-haiku-official-001', 'anthropic-official-001', 'claude-3-haiku-20240307', 'claude-3-haiku', 'Claude 3 Haiku (官方免费)', 'Fastest Claude model - 免费使用', 200000, 4096, 0.0, 0.0, TRUE, TRUE, TRUE, 11),

-- 官方付费服务的模型（价格更优惠）
('gpt-4-premium-001', 'openai-premium-001', 'gpt-4', 'gpt-4', 'GPT-4 (高级付费)', 'GPT-4 with discounted pricing', 8192, 4096, 25.0, 50.0, TRUE, TRUE, TRUE, 20),
('gpt-35-premium-001', 'openai-premium-001', 'gpt-3.5-turbo', 'gpt-3.5-turbo', 'GPT-3.5 Turbo (高级付费)', 'GPT-3.5 with discounted pricing', 16385, 4096, 1.2, 1.6, TRUE, TRUE, TRUE, 21);
```

### 3. 系统配置数据

```sql
INSERT INTO system_configs (config_id, config_key, config_value, config_type, display_name, description, group_name, is_required) VALUES
-- 基础配置
('sys-001', 'system.name', 'AgentU', 'string', '系统名称', '系统显示名称', 'system', TRUE),
('sys-002', 'system.version', '2.0.0', 'string', '系统版本', '当前系统版本号', 'system', TRUE),

-- 安全配置
('sec-001', 'security.password_min_length', '8', 'integer', '密码最小长度', '用户密码最小长度要求', 'security', TRUE),
('sec-002', 'security.max_login_attempts', '5', 'integer', '最大登录尝试次数', '账户锁定前的最大失败登录次数', 'security', TRUE),

-- API配置
('api-001', 'api.default_rate_limit', '100', 'integer', '默认API速率限制', '每分钟API调用次数限制', 'api', TRUE),
('api-002', 'api.timeout_seconds', '60', 'integer', 'API超时时间', 'API请求超时时间（秒）', 'api', TRUE),

-- 知识库配置
('kb-001', 'knowledge_base.max_file_size_mb', '50', 'integer', '文件最大大小', '知识库文件最大大小（MB）', 'knowledge_base', TRUE),
('kb-002', 'knowledge_base.default_chunk_size', '1000', 'integer', '默认分块大小', '文档分块默认大小', 'knowledge_base', TRUE);
```

## 后续扩展考虑的表

### 企业功能扩展

```sql
-- 1. 组织管理
-- organizations - 多租户组织表
-- user_roles - 用户角色权限表
-- team_members - 团队成员表

-- 2. 审计监控
-- audit_logs - 详细审计日志表（分区）
-- system_logs - 系统操作日志表
-- error_logs - 错误日志表

-- 3. 统计分析
-- usage_metrics - 使用统计表（分区）
-- performance_metrics - 性能指标表
-- cost_analytics - 成本分析表
-- user_behavior_analytics - 用户行为分析表

-- 4. 计费系统
-- billing_plans - 订阅计划表
-- billing_records - 账单记录表
-- payment_history - 支付历史表
-- quota_usage_history - 配额使用历史表

-- 5. 高级功能
-- function_definitions - 自定义函数定义表
-- workflow_templates - 工作流模板表
-- scheduled_tasks - 定时任务表
-- notification_settings - 通知设置表

-- 6. 数据导出
-- export_tasks - 数据导出任务表
-- export_history - 导出历史记录表

-- 7. 内容安全
-- content_filters - 内容过滤规则表
-- sensitive_data_detection - 敏感数据检测记录表

-- 8. 缓存优化
-- embedding_cache - 向量嵌入缓存表
-- response_cache - 响应缓存表
-- search_cache - 搜索结果缓存表

-- 9. 外部集成
-- webhook_configs - Webhook配置表
-- api_integrations - 第三方API集成表
-- sso_configs - 单点登录配置表
```

### 性能优化扩展

```sql
-- 1. 分区表策略
-- 对conversations、sessions表按时间分区
-- 对usage_metrics表按日期分区

-- 2. 物化视图
-- mv_user_daily_stats - 用户每日统计物化视图
-- mv_model_performance - 模型性能统计物化视图
-- mv_knowledge_base_stats - 知识库统计物化视图

-- 3. 索引优化
-- 复合索引优化
-- 部分索引优化
-- 表达式索引

-- 4. 数据归档
-- archived_conversations - 归档对话表
-- archived_sessions - 归档会话表
-- 定时归档策略和清理脚本
```

## 数据关系图

```
用户体系:
users (1) ——— (N) user_api_keys
users (1) ——— (N) sessions
users (1) ——— (N) knowledge_bases

AI体系:
ai_providers (1) ——— (N) ai_models
ai_providers (1) ——— (N) user_api_keys

会话体系:
sessions (1) ——— (N) conversations
ai_models (1) ——— (N) sessions
ai_models (1) ——— (N) conversations

知识库体系:
knowledge_bases (1) ——— (N) documents
documents (1) ——— (N) document_chunks
knowledge_bases (1) ——— (N) document_chunks
```

## 服务类型说明

### 三种服务模式

#### 1. USER_PROVIDED - 用户自备API密钥
- **使用场景**：用户有自己的OpenAI、Claude等API密钥
- **特点**：用户完全控制API使用，无需经过平台API密钥
- **计费**：用户直接承担API成本，平台不参与计费
- **配置**：用户在user_api_keys表中添加自己的encrypted_api_key

#### 2. OFFICIAL_FREE - 官方免费服务
- **使用场景**：平台为新用户提供免费试用额度
- **特点**：使用平台的API密钥，为用户提供有限的免费使用量
- **计费**：平台承担API成本，用户免费使用
- **限制**：每月有配额限制（free_quota_per_user_monthly）
- **配置**：用户的user_api_keys记录中encrypted_api_key为空，key_type为'OFFICIAL_FREE'

#### 3. OFFICIAL_PAID - 官方付费服务
- **使用场景**：用户付费使用平台提供的API服务，通常有价格优势
- **特点**：使用平台API密钥，用户按使用量付费给平台
- **计费**：平台统一采购API，为用户提供更优惠的价格
- **优势**：批量采购带来的价格优势，简化用户API管理
- **配置**：类似免费服务，但需要用户账户余额或付费

### 实现逻辑

```sql
-- 示例：用户注册时自动创建官方免费服务记录
INSERT INTO user_api_keys (api_key_id, user_id, provider_id, api_key_name, key_type, monthly_free_quota_used, quota_reset_date, status, created_by) 
SELECT 
    gen_random_uuid(),
    NEW.user_id,
    provider_id,
    provider_name || ' (免费额度)',
    'OFFICIAL_FREE',
    0,
    DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '1 month',
    'ACTIVE',
    NEW.user_id
FROM ai_providers 
WHERE service_type = 'OFFICIAL_FREE';
```

### 前端展示建议

用户在选择AI服务时，可以看到：
- **GPT-3.5 Turbo (免费)** - 每月50,000 tokens，剩余45,230 tokens
- **GPT-4 (自备密钥)** - 需要添加OpenAI API密钥
- **GPT-4 (高级付费)** - $25/百万tokens，比自备密钥便宜17%

## 总结

这个MVP版本的设计：

### 优点
1. **保持简洁**：专注核心功能，避免过度设计
2. **数据完整性**：添加必要的检查约束（移除外键以简化MVP）
3. **基础安全**：密码安全、登录保护、基础审计
4. **性能合理**：关键索引优化，避免过度索引
5. **易于扩展**：预留扩展接口，为后续升级做准备

6. **商业化支持**：支持三种服务模式，满足不同用户需求和商业模式

### 改进点
- 移除外键约束简化MVP开发和维护
- 增强了检查约束确保数据质量
- 价格单位改为百万tokens，符合行业标准
- 添加官方服务支持，为用户提供免费额度和优惠价格
- 支持多种商业模式：免费试用、自备密钥、付费服务
- 优化了索引策略提升查询性能
- 统一了数据类型和命名规范

### 商业化优势
- **用户体验**：新用户即可免费试用，降低使用门槛
- **灵活性**：支持用户自备密钥，满足不同用户需求
- **盈利能力**：官方付费服务提供新的收入来源
- **成本控制**：免费配额限制避免滥用
- **价格优势**：批量采购为用户提供更优惠价格

这个设计既适合快速开发MVP，又为商业化运营提供了完整的基础架构。