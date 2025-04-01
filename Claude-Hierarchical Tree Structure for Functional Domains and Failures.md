# Claude Conversation

## Prompt:
1个根节点下有一个5层树结构，树节点类型可能是功能或者功能域。
功能是下级，功能域是上级，功能域下可以建功能或者功能域，功能下面不能建功能或者功能域。后续5层树可能变6-10层树。
然后有一个故障表，故障只能挂在功能下。
功能域/功能全路径名称 加上系统元素 加上故障名称的hash作为code，
功能/功能域表也是路径hash做code，然后2表各有id，方便连接和去重。
然后还有一个故障分类表和系统元素分类表，
元素分类表属于某个功能，
故障分类可以指定是全局或者挂载在某个功能下面，
下面故障表可以有2个字段，属于某个分类/所属某个系统元素，
设计增删改查。
以及查询树全局和局部。
还有在树上单个增删改查。
以及相关表设计，使用springboot2.7.18+mysql+mybatis+xml，给出查询代码接口
至少包含以下接口：
功能树查询
新建功能域
新建功能
修改功能/功能域名称
删除功能/功能域
新增故障分类
新建故障
修改故障名称/故障分类
删除故障名称/故障分类
------------
之前还有一个表fema，有
bussinessName functionName subFunctionName systemElement FailureMode 字段
其中bussinessName functionName是功能域，subFunctionName是功能，systemElement是系统元素，FailureMode是故障。是一张横向的关联关系表。只有汉字名称。
要写一个方法把它转义到新的表结构中。
并且每个功能、功能域、故障在新表中有新的id和code，按照新表规则。之前这个表按
bussinessName functionName subFunctionName 的hash给功能生成了code，按
bussinessName functionName subFunctionName systemElement FailureMode给故障生成了code，所以新表的code才会按上面的定义，尽量保持一致。
新加的表要有creator
createTime
modifier
modifyTime
isDeleted字段，isDeleted=0表示未删除，isDeleted等于1表示删除，使用逻辑删除保留数据，并记录操作人。最好再加几张操作记录表记录操作记录。
现在给出你对这个业务需求的归纳理解以及对应的设计。给我确认后再生成代码，使用中文回复。

## Response:
 我将对您的业务需求进行归纳理解，并提出相应的系统设计方案。

## 业务需求理解

1. **树形结构**:
   - 一个根节点下有5层树结构(未来可能扩展到6-10层)
   - 节点类型分为"功能域"和"功能"
   - 功能域是上级，功能是下级
   - 功能域下可以建功能或功能域，功能下不能建任何节点

2. **数据关系**:
   - 故障只能挂在功能下
   - 系统元素分类属于某个功能
   - 故障分类可以是全局的或挂在某个功能下

3. **编码规则**:
   - 功能域/功能表：全路径名称的hash作为code
   - 故障表：功能全路径名称+系统元素+故障名称的hash作为code

4. **表格转换**:
   - 需要从旧的FEMA表迁移数据到新结构
   - 旧表字段：bussinessName(功能域)、functionName(功能域)、subFunctionName(功能)、systemElement(系统元素)、FailureMode(故障)

5. **通用字段**:
   - 所有表需包含：creator、createTime、modifier、modifyTime、isDeleted(逻辑删除)
   - 增加操作记录表记录所有操作

## 数据库设计

### 1. 功能域/功能表 (function_domain)
```sql
CREATE TABLE `function_domain` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码(路径hash)',
  `name` varchar(128) NOT NULL COMMENT '名称',
  `type` tinyint(4) NOT NULL COMMENT '类型(1:功能域 2:功能)',
  `parent_id` bigint(20) DEFAULT NULL COMMENT '父节点ID',
  `path` varchar(512) DEFAULT NULL COMMENT '全路径',
  `level` int(11) DEFAULT NULL COMMENT '层级',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modify_time` datetime DEFAULT NULL COMMENT '修改时间',
  `is_deleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`),
  KEY `idx_parent_id` (`parent_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='功能域/功能表';
```

### 2. 故障表 (failure_mode)
```sql
CREATE TABLE `failure_mode` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码(路径+系统元素+故障名称hash)',
  `name` varchar(128) NOT NULL COMMENT '故障名称',
  `function_id` bigint(20) NOT NULL COMMENT '所属功能ID',
  `system_element_id` bigint(20) DEFAULT NULL COMMENT '关联系统元素ID',
  `failure_category_id` bigint(20) DEFAULT NULL COMMENT '故障分类ID',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modify_time` datetime DEFAULT NULL COMMENT '修改时间',
  `is_deleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`),
  KEY `idx_function_id` (`function_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='故障表';
```

### 3. 系统元素表 (system_element)
```sql
CREATE TABLE `system_element` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码',
  `name` varchar(128) NOT NULL COMMENT '系统元素名称',
  `category_id` bigint(20) DEFAULT NULL COMMENT '系统元素分类ID',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modify_time` datetime DEFAULT NULL COMMENT '修改时间',
  `is_deleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统元素表';
```

### 4. 系统元素分类表 (system_element_category)
```sql
CREATE TABLE `system_element_category` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码',
  `name` varchar(128) NOT NULL COMMENT '分类名称',
  `function_id` bigint(20) NOT NULL COMMENT '所属功能ID',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modify_time` datetime DEFAULT NULL COMMENT '修改时间',
  `is_deleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`),
  KEY `idx_function_id` (`function_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统元素分类表';
```

### 5. 故障分类表 (failure_category)
```sql
CREATE TABLE `failure_category` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码',
  `name` varchar(128) NOT NULL COMMENT '分类名称',
  `is_global` tinyint(4) DEFAULT '0' COMMENT '是否全局(0:否 1:是)',
  `function_id` bigint(20) DEFAULT NULL COMMENT '所属功能ID',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modify_time` datetime DEFAULT NULL COMMENT '修改时间',
  `is_deleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`),
  KEY `idx_function_id` (`function_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='故障分类表';
```

### 6. 操作日志表 (operation_log)
```sql
CREATE TABLE `operation_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `operation_type` varchar(32) NOT NULL COMMENT '操作类型(新增/修改/删除)',
  `operation_object` varchar(32) NOT NULL COMMENT '操作对象(功能域/功能/故障/分类等)',
  `operation_id` bigint(20) NOT NULL COMMENT '操作对象ID',
  `operation_content` text COMMENT '操作内容',
  `operator` varchar(64) DEFAULT NULL COMMENT '操作人',
  `operation_time` datetime DEFAULT NULL COMMENT '操作时间',
  PRIMARY KEY (`id`),
  KEY `idx_operation_id` (`operation_id`),
  KEY `idx_operation_time` (`operation_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='操作日志表';
```

## 项目结构设计

```
src/main/java/com/example/fema
├── config          // 配置类
├── controller      // 控制层
├── service         // 服务层
│   └── impl        // 服务实现
├── mapper          // DAO层
├── entity          // 实体类
├── dto             // 数据传输对象
├── vo              // 视图对象
├── constant        // 常量类
├── util            // 工具类
│   └── HashUtil.java // 哈希生成工具
├── exception       // 异常处理
└── Application.java // 启动类
```

## 实现功能和接口设计

1. **功能树查询**
   - 全局树查询
   - 根据某节点查询子树

2. **功能域/功能管理**
   - 新建功能域
   - 新建功能
   - 修改功能/功能域名称
   - 删除功能/功能域

3. **故障管理**
   - 新增故障分类
   - 新建故障
   - 修改故障名称/故障分类
   - 删除故障名称/故障分类

4. **数据迁移**
   - 从旧FEMA表导入数据

请确认以上设计是否符合您的需求，确认后我将为您生成对应的代码实现。

## Prompt:
新加的表要有creator
createTime
modifier
modifyTime
isDeleted字段,就使用我给的字段名，我习惯用驼峰，字段都用驼峰系统元素不用分类，系统元素表这个表只是为了迁移旧数据挂载一下之前的路径，还有故障表里把系统元素名称也冗余一下，用于生成code，如果改变路径和名称，功能，功能域，故障都要重新生成code，但转移的旧数据要有一个冗余字段标记以前表的全path，用于标记，再给个类别字段用于标记接口新建和旧表转移数据，重新设计一下

## Response:
 我将根据您的新要求重新进行业务需求归纳和系统设计。

## 业务需求理解

1. **树形结构**:
   - 一个根节点下有5层树结构(未来可能扩展到6-10层)
   - 节点类型分为"功能域"和"功能"
   - 功能域是上级，功能是下级
   - 功能域下可以建功能或功能域，功能下不能建任何节点

2. **数据关系**:
   - 故障只能挂在功能下
   - 系统元素不用分类，仅用于迁移旧数据
   - 故障分类可以是全局的或挂在某个功能下

3. **编码规则**:
   - 功能域/功能表：全路径名称的hash作为code
   - 故障表：功能全路径名称+系统元素+故障名称的hash作为code
   - 如果路径或名称变更，需要重新生成code

4. **表格转换**:
   - 需要从旧的FEMA表迁移数据到新结构
   - 旧表字段：bussinessName(功能域)、functionName(功能域)、subFunctionName(功能)、systemElement(系统元素)、FailureMode(故障)
   - 旧数据需要有冗余字段标记原表全路径
   - 需要标记数据来源(新建/旧表迁移)

5. **通用字段**:
   - 所有表需包含：creator、createTime、modifier、modifyTime、isDeleted(逻辑删除)
   - 增加操作记录表记录所有操作

## 数据库设计

### 1. 功能域/功能表 (function_domain)
```sql
CREATE TABLE `function_domain` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码(路径hash)',
  `name` varchar(128) NOT NULL COMMENT '名称',
  `type` tinyint(4) NOT NULL COMMENT '类型(1:功能域 2:功能)',
  `parentId` bigint(20) DEFAULT NULL COMMENT '父节点ID',
  `path` varchar(512) DEFAULT NULL COMMENT '全路径',
  `level` int(11) DEFAULT NULL COMMENT '层级',
  `oldPath` varchar(512) DEFAULT NULL COMMENT '旧表全路径',
  `dataSource` tinyint(4) DEFAULT '0' COMMENT '数据来源(0:新增 1:迁移)',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `createTime` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modifyTime` datetime DEFAULT NULL COMMENT '修改时间',
  `isDeleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`),
  KEY `idx_parentId` (`parentId`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='功能域/功能表';
```

### 2. 故障表 (failure_mode)
```sql
CREATE TABLE `failure_mode` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码(路径+系统元素+故障名称hash)',
  `name` varchar(128) NOT NULL COMMENT '故障名称',
  `functionId` bigint(20) NOT NULL COMMENT '所属功能ID',
  `systemElement` varchar(128) DEFAULT NULL COMMENT '系统元素名称',
  `systemElementId` bigint(20) DEFAULT NULL COMMENT '关联系统元素ID',
  `failureCategoryId` bigint(20) DEFAULT NULL COMMENT '故障分类ID',
  `oldPath` varchar(512) DEFAULT NULL COMMENT '旧表全路径',
  `dataSource` tinyint(4) DEFAULT '0' COMMENT '数据来源(0:新增 1:迁移)',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `createTime` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modifyTime` datetime DEFAULT NULL COMMENT '修改时间',
  `isDeleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`),
  KEY `idx_functionId` (`functionId`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='故障表';
```

### 3. 系统元素表 (system_element)
```sql
CREATE TABLE `system_element` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码',
  `name` varchar(128) NOT NULL COMMENT '系统元素名称',
  `oldPath` varchar(512) DEFAULT NULL COMMENT '旧表全路径',
  `dataSource` tinyint(4) DEFAULT '0' COMMENT '数据来源(0:新增 1:迁移)',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `createTime` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modifyTime` datetime DEFAULT NULL COMMENT '修改时间',
  `isDeleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统元素表';
```

### 4. 故障分类表 (failure_category)
```sql
CREATE TABLE `failure_category` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `code` varchar(64) NOT NULL COMMENT '编码',
  `name` varchar(128) NOT NULL COMMENT '分类名称',
  `isGlobal` tinyint(4) DEFAULT '0' COMMENT '是否全局(0:否 1:是)',
  `functionId` bigint(20) DEFAULT NULL COMMENT '所属功能ID',
  `creator` varchar(64) DEFAULT NULL COMMENT '创建人',
  `createTime` datetime DEFAULT NULL COMMENT '创建时间',
  `modifier` varchar(64) DEFAULT NULL COMMENT '修改人',
  `modifyTime` datetime DEFAULT NULL COMMENT '修改时间',
  `isDeleted` tinyint(4) DEFAULT '0' COMMENT '是否删除(0:未删除 1:已删除)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_code` (`code`),
  KEY `idx_functionId` (`functionId`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='故障分类表';
```

### 5. 操作日志表 (operation_log)
```sql
CREATE TABLE `operation_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `operationType` varchar(32) NOT NULL COMMENT '操作类型(新增/修改/删除)',
  `operationObject` varchar(32) NOT NULL COMMENT '操作对象(功能域/功能/故障/分类等)',
  `operationId` bigint(20) NOT NULL COMMENT '操作对象ID',
  `operationContent` text COMMENT '操作内容',
  `operator` varchar(64) DEFAULT NULL COMMENT '操作人',
  `operationTime` datetime DEFAULT NULL COMMENT '操作时间',
  PRIMARY KEY (`id`),
  KEY `idx_operationId` (`operationId`),
  KEY `idx_operationTime` (`operationTime`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='操作日志表';
```

## 项目结构设计

```
src/main/java/com/example/fema
├── config          // 配置类
├── controller      // 控制层
├── service         // 服务层
│   └── impl        // 服务实现
├── mapper          // DAO层
├── entity          // 实体类
├── dto             // 数据传输对象
├── vo              // 视图对象
├── constant        // 常量类
├── util            // 工具类
│   └── HashUtil.java // 哈希生成工具
├── exception       // 异常处理
└── Application.java // 启动类
```

## 核心实体类设计

### 功能域/功能实体
```java
@Data
public class FunctionDomain {
    private Long id;
    private String code;
    private String name;
    private Integer type; // 1:功能域 2:功能
    private Long parentId;
    private String path;
    private Integer level;
    private String oldPath;
    private Integer dataSource; // 0:新增 1:迁移
    private String creator;
    private Date createTime;
    private String modifier;
    private Date modifyTime;
    private Integer isDeleted;
}
```

### 故障实体
```java
@Data
public class FailureMode {
    private Long id;
    private String code;
    private String name;
    private Long functionId;
    private String systemElement;
    private Long systemElementId;
    private Long failureCategoryId;
    private String oldPath;
    private Integer dataSource; // 0:新增 1:迁移
    private String creator;
    private Date createTime;
    private String modifier;
    private Date modifyTime;
    private Integer isDeleted;
}
```

### 系统元素实体
```java
@Data
public class SystemElement {
    private Long id;
    private String code;
    private String name;
    private String oldPath;
    private Integer dataSource; // 0:新增 1:迁移
    private String creator;
    private Date createTime;
    private String modifier;
    private Date modifyTime;
    private Integer isDeleted;
}
```

### 故障分类实体
```java
@Data
public class FailureCategory {
    private Long id;
    private String code;
    private String name;
    private Integer isGlobal; // 0:否 1:是
    private Long functionId;
    private String creator;
    private Date createTime;
    private String modifier;
    private Date modifyTime;
    private Integer isDeleted;
}
```

## 接口设计

### 功能树相关接口
1. **功能树查询**
   - 全局树查询
   - 根据某节点查询子树

2. **功能域/功能管理**
   - 新建功能域
   - 新建功能
   - 修改功能/功能域名称
   - 删除功能/功能域

### 故障相关接口
1. **故障管理**
   - 新增故障分类
   - 新建故障
   - 修改故障名称/故障分类
   - 删除故障名称/故障分类

### 数据迁移接口
1. **数据迁移**
   - 从旧FEMA表导入数据

## 实现细节

1. **Code生成规则**
   - 功能域/功能: 全路径名称的hash
   - 故障: 功能全路径+系统元素+故障名称的hash
   - 路径或名称变更时重新生成code

2. **操作日志记录**
   - 记录所有增删改操作
   - 保存操作人、操作时间、操作内容等信息

3. **数据迁移**
   - 将旧FEMA表数据转换为新表结构
   - 保留原始路径信息
   - 标记数据来源

请确认以上设计是否符合您的需求，确认后我将为您生成对应的代码实现。

## Prompt:
符合我的需求，请我我生成详细代码和注释还有一个故障查询接口，要返回未分类故障，和故障分类放第一层，分类下有故障的放第二层，还有操作记录查询接口，你看看还缺哪些接口设计

## Response:
 非常感谢您的确认。我将为您生成详细的代码实现，包括您提到的故障查询接口和操作记录查询接口。

## 项目整体结构

```
src/main/java/com/example/fema
├── config
│   └── MyBatisConfig.java
├── controller
│   ├── FunctionDomainController.java     // 功能域/功能控制器
│   ├── FailureModeController.java        // 故障控制器
│   ├── FailureCategoryController.java    // 故障分类控制器
│   ├── SystemElementController.java      // 系统元素控制器
│   └── OperationLogController.java       // 操作日志控制器
├── service
│   ├── FunctionDomainService.java
│   ├── FailureModeService.java
│   ├── FailureCategoryService.java
│   ├── SystemElementService.java
│   ├── OperationLogService.java
│   ├── DataMigrationService.java         // 数据迁移服务
│   └── impl
│       ├── FunctionDomainServiceImpl.java
│       ├── FailureModeServiceImpl.java
│       ├── FailureCategoryServiceImpl.java
│       ├── SystemElementServiceImpl.java
│       ├── OperationLogServiceImpl.java
│       └── DataMigrationServiceImpl.java
├── mapper
│   ├── FunctionDomainMapper.java
│   ├── FailureModeMapper.java
│   ├── FailureCategoryMapper.java
│   ├── SystemElementMapper.java
│   └── OperationLogMapper.java
├── entity
│   ├── FunctionDomain.java
│   ├── FailureMode.java
│   ├── FailureCategory.java
│   ├── SystemElement.java
│   └── OperationLog.java
├── dto
│   ├── FunctionDomainDTO.java
│   ├── FailureModeDTO.java
│   ├── FailureCategoryDTO.java
│   ├── SystemElementDTO.java
│   └── FemaDataDTO.java                  // 旧数据迁移DTO
├── vo
│   ├── FunctionTreeVO.java
│   ├── FailureTreeVO.java                // 故障树VO
│   ├── ResultVO.java
│   └── PageVO.java
├── constant
│   ├── CommonConstant.java
│   ├── FunctionTypeEnum.java
│   └── ResponseCode.java
├── util
│   ├── HashUtil.java                     // 哈希工具
│   ├── TreeUtil.java                     // 树形结构工具
│   └── ResultUtil.java
├── exception
│   ├── BusinessException.java
│   └── GlobalExceptionHandler.java
└── Application.java
```

## 核心代码实现

### 1. 实体类

**FunctionDomain.java**
```java
package com.example.fema.entity;

import lombok.Data;
import java.util.Date;

/**
 * 功能域/功能实体类
 */
@Data
public class FunctionDomain {
    /**
     * 主键ID
     */
    private Long id;
    
    /**
     * 编码(路径hash)
     */
    private String code;
    
    /**
     * 名称
     */
    private String name;
    
    /**
     * 类型(1:功能域 2:功能)
     */
    private Integer type;
    
    /**
     * 父节点ID
     */
    private Long parentId;
    
    /**
     * 全路径
     */
    private String path;
    
    /**
     * 层级
     */
    private Integer level;
    
    /**
     * 旧表全路径
     */
    private String oldPath;
    
    /**
     * 数据来源(0:新增 1:迁移)
     */
    private Integer dataSource;
    
    /**
     * 创建人
     */
    private String creator;
    
    /**
     * 创建时间
     */
    private Date createTime;
    
    /**
     * 修改人
     */
    private String modifier;
    
    /**
     * 修改时间
     */
    private Date modifyTime;
    
    /**
     * 是否删除(0:未删除 1:已删除)
     */
    private Integer isDeleted;
}
```

**FailureMode.java**
```java
package com.example.fema.entity;

import lombok.Data;
import java.util.Date;

/**
 * 故障实体类
 */
@Data
public class FailureMode {
    /**
     * 主键ID
     */
    private Long id;
    
    /**
     * 编码(路径+系统元素+故障名称hash)
     */
    private String code;
    
    /**
     * 故障名称
     */
    private String name;
    
    /**
     * 所属功能ID
     */
    private Long functionId;
    
    /**
     * 系统元素名称
     */
    private String systemElement;
    
    /**
     * 关联系统元素ID
     */
    private Long systemElementId;
    
    /**
     * 故障分类ID
     */
    private Long failureCategoryId;
    
    /**
     * 旧表全路径
     */
    private String oldPath;
    
    /**
     * 数据来源(0:新增 1:迁移)
     */
    private Integer dataSource;
    
    /**
     * 创建人
     */
    private String creator;
    
    /**
     * 创建时间
     */
    private Date createTime;
    
    /**
     * 修改人
     */
    private String modifier;
    
    /**
     * 修改时间
     */
    private Date modifyTime;
    
    /**
     * 是否删除(0:未删除 1:已删除)
     */
    private Integer isDeleted;
}
```

**FailureCategory.java**
```java
package com.example.fema.entity;

import lombok.Data;
import java.util.Date;

/**
 * 故障分类实体类
 */
@Data
public class FailureCategory {
    /**
     * 主键ID
     */
    private Long id;
    
    /**
     * 编码
     */
    private String code;
    
    /**
     * 分类名称
     */
    private String name;
    
    /**
     * 是否全局(0:否 1:是)
     */
    private Integer isGlobal;
    
    /**
     * 所属功能ID
     */
    private Long functionId;
    
    /**
     * 创建人
     */
    private String creator;
    
    /**
     * 创建时间
     */
    private Date createTime;
    
    /**
     * 修改人
     */
    private String modifier;
    
    /**
     * 修改时间
     */
    private Date modifyTime;
    
    /**
     * 是否删除(0:未删除 1:已删除)
     */
    private Integer isDeleted;
}
```

**SystemElement.java**
```java
package com.example.fema.entity;

import lombok.Data;
import java.util.Date;

/**
 * 系统元素实体类
 */
@Data
public class SystemElement {
    /**
     * 主键ID
     */
    private Long id;
    
    /**
     * 编码
     */
    private String code;
    
    /**
     * 系统元素名称
     */
    private String name;
    
    /**
     * 旧表全路径
     */
    private String oldPath;
    
    /**
     * 数据来源(0:新增 1:迁移)
     */
    private Integer dataSource;
    
    /**
     * 创建人
     */
    private String creator;
    
    /**
     * 创建时间
     */
    private Date createTime;
    
    /**
     * 修改人
     */
    private String modifier;
    
    /**
     * 修改时间
     */
    private Date modifyTime;
    
    /**
     * 是否删除(0:未删除 1:已删除)
     */
    private Integer isDeleted;
}
```

**OperationLog.java**
```java
package com.example.fema.entity;

import lombok.Data;
import java.util.Date;

/**
 * 操作日志实体类
 */
@Data
public class OperationLog {
    /**
     * 主键ID
     */
    private Long id;
    
    /**
     * 操作类型(新增/修改/删除)
     */
    private String operationType;
    
    /**
     * 操作对象(功能域/功能/故障/分类等)
     */
    private String operationObject;
    
    /**
     * 操作对象ID
     */
    private Long operationId;
    
    /**
     * 操作内容
     */
    private String operationContent;
    
    /**
     * 操作人
     */
    private String operator;
    
    /**
     * 操作时间
     */
    private Date operationTime;
}
```

### 2. VO对象

**FunctionTreeVO.java**
```java
package com.example.fema.vo;

import lombok.Data;
import java.util.List;

/**
 * 功能树视图对象
 */
@Data
public class FunctionTreeVO {
    /**
     * 节点ID
     */
    private Long id;
    
    /**
     * 节点编码
     */
    private String code;
    
    /**
     * 节点名称
     */
    private String name;
    
    /**
     * 节点类型(1:功能域 2:功能)
     */
    private Integer type;
    
    /**
     * 层级
     */
    private Integer level;
    
    /**
     * 路径
     */
    private String path;
    
    /**
     * 子节点
     */
    private List<FunctionTreeVO> children;
}
```

**FailureTreeVO.java**
```java
package com.example.fema.vo;

import lombok.Data;
import java.util.List;

/**
 * 故障树视图对象
 */
@Data
public class FailureTreeVO {
    /**
     * 节点ID
     */
    private Long id;
    
    /**
     * 节点编码
     */
    private String code;
    
    /**
     * 节点名称
     */
    private String name;
    
    /**
     * 节点类型(1:分类 2:故障)
     */
    private Integer type;
    
    /**
     * 功能ID
     */
    private Long functionId;
    
    /**
     * 系统元素
     */
    private String systemElement;
    
    /**
     * 子节点
     */
    private List<FailureTreeVO> children;
}
```

**ResultVO.java**
```java
package com.example.fema.vo;

import lombok.Data;

/**
 * 通用返回结果视图对象
 */
@Data
public class ResultVO<T> {
    /**
     * 状态码
     */
    private Integer code;
    
    /**
     * 消息
     */
    private String message;
    
    /**
     * 数据
     */
    private T data;
    
    public static <T> ResultVO<T> success(T data) {
        ResultVO<T> result = new ResultVO<>();
        result.setCode(200);
        result.setMessage("操作成功");
        result.setData(data);
        return result;
    }
    
    public static <T> ResultVO<T> success(String message, T data) {
        ResultVO<T> result = new ResultVO<>();
        result.setCode(200);
        result.setMessage(message);
        result.setData(data);
        return result;
    }
    
    public static <T> ResultVO<T> error(Integer code, String message) {
        ResultVO<T> result = new ResultVO<>();
        result.setCode(code);
        result.setMessage(message);
        return result;
    }
}
```

### 3. 工具类

**HashUtil.java**
```java
package com.example.fema.util;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

/**
 * 哈希工具类
 */
public class HashUtil {
    
    /**
     * 生成MD5哈希
     *
     * @param input 输入字符串
     * @return MD5哈希值
     */
    public static String md5(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] hash = md.digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder sb = new StringBuilder();
            for (byte b : hash) {
                sb.append(String.format("%02x", b));
            }
            return sb.toString();
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("MD5哈希计算失败", e);
        }
    }
    
    /**
     * 生成功能域/功能code
     *
     * @param path 全路径
     * @return code
     */
    public static String generateFunctionCode(String path) {
        return md5(path);
    }
    
    /**
     * 生成故障code
     *
     * @param functionPath 功能全路径
     * @param systemElement 系统元素
     * @param failureName 故障名称
     * @return code
     */
    public static String generateFailureCode(String functionPath, String systemElement, String failureName) {
        return md5(functionPath + "_" + systemElement + "_" + failureName);
    }
}
```

**TreeUtil.java**
```java
package com.example.fema.util;

import com.example.fema.entity.FailureCategory;
import com.example.fema.entity.FailureMode;
import com.example.fema.entity.FunctionDomain;
import com.example.fema.vo.FailureTreeVO;
import com.example.fema.vo.FunctionTreeVO;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * 树形结构工具类
 */
public class TreeUtil {
    
    /**
     * 构建功能树
     *
     * @param functionList 功能列表
     * @param parentId 父节点ID
     * @return 功能树
     */
    public static List<FunctionTreeVO> buildFunctionTree(List<FunctionDomain> functionList, Long parentId) {
        List<FunctionTreeVO> treeList = new ArrayList<>();
        
        // 过滤出当前层级的节点
        List<FunctionDomain> currLevel = functionList.stream()
                .filter(func -> func.getParentId() == null && parentId == null || 
                       func.getParentId() != null && func.getParentId().equals(parentId))
                .collect(Collectors.toList());
                
        // 构建当前层级的树节点
        for (FunctionDomain func : currLevel) {
            FunctionTreeVO treeNode = new FunctionTreeVO();
            treeNode.setId(func.getId());
            treeNode.setCode(func.getCode());
            treeNode.setName(func.getName());
            treeNode.setType(func.getType());
            treeNode.setLevel(func.getLevel());
            treeNode.setPath(func.getPath());
            
            // 递归构建子节点
            treeNode.setChildren(buildFunctionTree(functionList, func.getId()));
            
            treeList.add(treeNode);
        }
        
        return treeList;
    }
    
    /**
     * 构建故障树
     *
     * @param categoryList 故障分类列表
     * @param failureList 故障列表
     * @param functionId 功能ID
     * @return 故障树
     */
    public static List<FailureTreeVO> buildFailureTree(
            List<FailureCategory> categoryList, 
            List<FailureMode> failureList, 
            Long functionId) {
        
        List<FailureTreeVO> treeList = new ArrayList<>();
        
        // 过滤出全局分类和当前功能下的分类
        List<FailureCategory> filteredCategories = categoryList.stream()
                .filter(category -> category.getIsGlobal() == 1 || 
                       (category.getFunctionId() != null && category.getFunctionId().equals(functionId)))
                .collect(Collectors.toList());
        
        // 过滤当前功能下的故障
        List<FailureMode> filteredFailures = failureList.stream()
                .filter(failure -> failure.getFunctionId().equals(functionId))
                .collect(Collectors.toList());
        
        // 按分类ID分组故障
        Map<Long, List<FailureMode>> failureMap = filteredFailures.stream()
                .filter(failure -> failure.getFailureCategoryId() != null)
                .collect(Collectors.groupingBy(FailureMode::getFailureCategoryId));
        
        // 未分类故障
        List<FailureMode> unclassifiedFailures = filteredFailures.stream()
                .filter(failure -> failure.getFailureCategoryId() == null)
                .collect(Collectors.toList());
        
        // 先添加未分类故障节点
        if (!unclassifiedFailures.isEmpty()) {
            FailureTreeVO unclassifiedNode = new FailureTreeVO();
            unclassifiedNode.setId(0L);
            unclassifiedNode.setCode("unclassified");
            unclassifiedNode.setName("未分类故障");
            unclassifiedNode.setType(1); // 类型为分类
            unclassifiedNode.setFunctionId(functionId);
            
            // 添加子节点
            List<FailureTreeVO> children = new ArrayList<>();
            for (FailureMode failure : unclassifiedFailures) {
                FailureTreeVO childNode = new FailureTreeVO();
                childNode.setId(failure.getId());
                childNode.setCode(failure.getCode());
                childNode.setName(failure.getName());
                childNode.setType(2); // 类型为故障
                childNode.setFunctionId(failure.getFunctionId());
                childNode.setSystemElement(failure.getSystemElement());
                children.add(childNode);
            }
            
            unclassifiedNode.setChildren(children);
            treeList.add(unclassifiedNode);
        }
        
        // 构建分类节点
        for (FailureCategory category : filteredCategories) {
            FailureTreeVO categoryNode = new FailureTreeVO();
            categoryNode.setId(category.getId());
            categoryNode.setCode(category.getCode());
            categoryNode.setName(category.getName());
            categoryNode.setType(1); // 类型为分类
            categoryNode.setFunctionId(category.getFunctionId());
            
            // 添加分类下的故障作为子节点
            List<FailureTreeVO> children = new ArrayList<>();
            List<FailureMode> categoryFailures = failureMap.getOrDefault(category.getId(), new ArrayList<>());
            
            for (FailureMode failure : categoryFailures) {
                FailureTreeVO childNode = new FailureTreeVO();
                childNode.setId(failure.getId());
                childNode.setCode(failure.getCode());
                childNode.setName(failure.getName());
                childNode.setType(2); // 类型为故障
                childNode.setFunctionId(failure.getFunctionId());
                childNode.setSystemElement(failure.getSystemElement());
                children.add(childNode);
            }
            
            categoryNode.setChildren(children);
            treeList.add(categoryNode);
        }
        
        return treeList;
    }
}
```

### 4. Mapper 接口和 XML 配置

**FunctionDomainMapper.java**
```java
package com.example.fema.mapper;

import com.example.fema.entity.FunctionDomain;
import org.apache.ibatis.annotations.Param;
import java.util.List;

/**
 * 功能域/功能Mapper接口
 */
public interface FunctionDomainMapper {
    
    /**
     * 查询全部功能域/功能
     *
     * @return 功能域/功能列表
     */
    List<FunctionDomain> selectAll();
    
    /**
     * 根据ID查询功能域/功能
     *
     * @param id ID
     * @return 功能域/功能
     */
    FunctionDomain selectById(@Param("id") Long id);
    
    /**
     * 根据父ID查询功能域/功能
     *
     * @param parentId 父ID
     * @return 功能域/功能列表
     */
    List<FunctionDomain> selectByParentId(@Param("parentId") Long parentId);
    
    /**
     * 根据路径查询功能域/功能
     *
     * @param path 路径
     * @return 功能域/功能列表
     */
    List<FunctionDomain> selectByPath(@Param("path") String path);
    
    /**
     * 根据code查询功能域/功能
     *
     * @param code 编码
     * @return 功能域/功能
     */
    FunctionDomain selectByCode(@Param("code") String code);
    
    /**
     * 新增功能域/功能
     *
     * @param functionDomain 功能域/功能
     * @return 影响行数
     */
    int insert(FunctionDomain functionDomain);
    
    /**
     * 修改功能域/功能
     *
     * @param functionDomain 功能域/功能
     * @return 影响行数
     */
    int update(FunctionDomain functionDomain);
    
    /**
     * 逻辑删除功能域/功能
     *
     * @param id ID
     * @param modifier 修改人
     * @return 影响行数
     */
    int deleteById(@Param("id") Long id, @Param("modifier") String modifier);
    
    /**
     * 根据父ID查询子节点数量
     *
     * @param parentId 父ID
     * @return 子节点数量
     */
    int countByParentId(@Param("parentId") Long parentId);
}
```

**FunctionDomainMapper.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.fema.mapper.FunctionDomainMapper">
    
    <resultMap id="BaseResultMap" type="com.example.fema.entity.FunctionDomain">
        <id column="id" property="id" />
        <result column="code" property="code" />
        <result column="name" property="name" />
        <result column="type" property="type" />
        <result column="parentId" property="parentId" />
        <result column="path" property="path" />
        <result column="level" property="level" />
        <result column="oldPath" property="oldPath" />
        <result column="dataSource" property="dataSource" />
        <result column="creator" property="creator" />
        <result column="createTime" property="createTime" />
        <result column="modifier" property="modifier" />
        <result column="modifyTime" property="modifyTime" />
        <result column="isDeleted" property="isDeleted" />
    </resultMap>
    
    <sql id="Base_Column_List">
        id, code, name, type, parentId, path, level, oldPath, dataSource, 
        creator, createTime, modifier, modifyTime, isDeleted
    </sql>
    
    <select id="selectAll" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM function_domain
        WHERE isDeleted = 0
    </select>
    
    <select id="selectById" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM function_domain
        WHERE id = #{id} AND isDeleted = 0
    </select>
    
    <select id="selectByParentId" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM function_domain
        WHERE parentId = #{parentId} AND isDeleted = 0
    </select>
    
    <select id="selectByPath" parameterType="java.lang.String" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM function_domain
        WHERE path LIKE CONCAT(#{path}, '%') AND isDeleted = 0
    </select>
    
    <select id="selectByCode" parameterType="java.lang.String" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM function_domain
        WHERE code = #{code} AND isDeleted = 0
    </select>
    
    <select id="countByParentId" parameterType="java.lang.Long" resultType="java.lang.Integer">
        SELECT COUNT(1)
        FROM function_domain
        WHERE parentId = #{parentId} AND isDeleted = 0
    </select>
    
    <insert id="insert" parameterType="com.example.fema.entity.FunctionDomain" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO function_domain (
            code, name, type, parentId, path, level, oldPath, dataSource,
            creator, createTime, modifier, modifyTime, isDeleted
        ) VALUES (
            #{code}, #{name}, #{type}, #{parentId}, #{path}, #{level}, #{oldPath}, #{dataSource},
            #{creator}, NOW(), #{modifier}, NOW(), 0
        )
    </insert>
    
    <update id="update" parameterType="com.example.fema.entity.FunctionDomain">
        UPDATE function_domain
        <set>
            <if test="name != null">name = #{name},</if>
            <if test="code != null">code = #{code},</if>
            <if test="path != null">path = #{path},</if>
            modifier = #{modifier},
            modifyTime = NOW()
        </set>
        WHERE id = #{id} AND isDeleted = 0
    </update>
    
    <update id="deleteById">
        UPDATE function_domain
        SET isDeleted = 1,
            modifier = #{modifier},
            modifyTime = NOW()
        WHERE id = #{id} AND isDeleted = 0
    </update>
</mapper>
```

**FailureModeMapper.java**
```java
package com.example.fema.mapper;

import com.example.fema.entity.FailureMode;
import org.apache.ibatis.annotations.Param;
import java.util.List;

/**
 * 故障Mapper接口
 */
public interface FailureModeMapper {
    
    /**
     * 查询全部故障
     *
     * @return 故障列表
     */
    List<FailureMode> selectAll();
    
    /**
     * 根据ID查询故障
     *
     * @param id ID
     * @return 故障
     */
    FailureMode selectById(@Param("id") Long id);
    
    /**
     * 根据功能ID查询故障
     *
     * @param functionId 功能ID
     * @return 故障列表
     */
    List<FailureMode> selectByFunctionId(@Param("functionId") Long functionId);
    
    /**
     * 根据故障分类ID查询故障
     *
     * @param categoryId 故障分类ID
     * @return 故障列表
     */
    List<FailureMode> selectByCategoryId(@Param("categoryId") Long categoryId);
    
    /**
     * 查询功能下未分类故障
     *
     * @param functionId 功能ID
     * @return 未分类故障列表
     */
    List<FailureMode> selectUnclassifiedByFunctionId(@Param("functionId") Long functionId);
    
    /**
     * 根据code查询故障
     *
     * @param code 编码
     * @return 故障
     */
    FailureMode selectByCode(@Param("code") String code);
    
    /**
     * 新增故障
     *
     * @param failureMode 故障
     * @return 影响行数
     */
    int insert(FailureMode failureMode);
    
    /**
     * 修改故障
     *
     * @param failureMode 故障
     * @return 影响行数
     */
    int update(FailureMode failureMode);
    
    /**
     * 逻辑删除故障
     *
     * @param id ID
     * @param modifier 修改人
     * @return 影响行数
     */
    int deleteById(@Param("id") Long id, @Param("modifier") String modifier);
}
```

**FailureModeMapper.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.fema.mapper.FailureModeMapper">
    
    <resultMap id="BaseResultMap" type="com.example.fema.entity.FailureMode">
        <id column="id" property="id" />
        <result column="code" property="code" />
        <result column="name" property="name" />
        <result column="functionId" property="functionId" />
        <result column="systemElement" property="systemElement" />
        <result column="systemElementId" property="systemElementId" />
        <result column="failureCategoryId" property="failureCategoryId" />
        <result column="oldPath" property="oldPath" />
        <result column="dataSource" property="dataSource" />

## Prompt:
继续

## Response:
 继续提供剩余的代码实现：

**FailureModeMapper.xml** (续)
```xml
        <result column="creator" property="creator" />
        <result column="createTime" property="createTime" />
        <result column="modifier" property="modifier" />
        <result column="modifyTime" property="modifyTime" />
        <result column="isDeleted" property="isDeleted" />
    </resultMap>
    
    <sql id="Base_Column_List">
        id, code, name, functionId, systemElement, systemElementId, failureCategoryId, 
        oldPath, dataSource, creator, createTime, modifier, modifyTime, isDeleted
    </sql>
    
    <select id="selectAll" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_mode
        WHERE isDeleted = 0
    </select>
    
    <select id="selectById" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_mode
        WHERE id = #{id} AND isDeleted = 0
    </select>
    
    <select id="selectByFunctionId" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_mode
        WHERE functionId = #{functionId} AND isDeleted = 0
    </select>
    
    <select id="selectByCategoryId" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_mode
        WHERE failureCategoryId = #{categoryId} AND isDeleted = 0
    </select>
    
    <select id="selectUnclassifiedByFunctionId" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_mode
        WHERE functionId = #{functionId} AND failureCategoryId IS NULL AND isDeleted = 0
    </select>
    
    <select id="selectByCode" parameterType="java.lang.String" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_mode
        WHERE code = #{code} AND isDeleted = 0
    </select>
    
    <insert id="insert" parameterType="com.example.fema.entity.FailureMode" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO failure_mode (
            code, name, functionId, systemElement, systemElementId, failureCategoryId,
            oldPath, dataSource, creator, createTime, modifier, modifyTime, isDeleted
        ) VALUES (
            #{code}, #{name}, #{functionId}, #{systemElement}, #{systemElementId}, #{failureCategoryId},
            #{oldPath}, #{dataSource}, #{creator}, NOW(), #{modifier}, NOW(), 0
        )
    </insert>
    
    <update id="update" parameterType="com.example.fema.entity.FailureMode">
        UPDATE failure_mode
        <set>
            <if test="name != null">name = #{name},</if>
            <if test="code != null">code = #{code},</if>
            <if test="systemElement != null">systemElement = #{systemElement},</if>
            <if test="failureCategoryId != null">failureCategoryId = #{failureCategoryId},</if>
            modifier = #{modifier},
            modifyTime = NOW()
        </set>
        WHERE id = #{id} AND isDeleted = 0
    </update>
    
    <update id="deleteById">
        UPDATE failure_mode
        SET isDeleted = 1,
            modifier = #{modifier},
            modifyTime = NOW()
        WHERE id = #{id} AND isDeleted = 0
    </update>
</mapper>
```

**FailureCategoryMapper.java**
```java
package com.example.fema.mapper;

import com.example.fema.entity.FailureCategory;
import org.apache.ibatis.annotations.Param;
import java.util.List;

/**
 * 故障分类Mapper接口
 */
public interface FailureCategoryMapper {
    
    /**
     * 查询全部故障分类
     *
     * @return 故障分类列表
     */
    List<FailureCategory> selectAll();
    
    /**
     * 根据ID查询故障分类
     *
     * @param id ID
     * @return 故障分类
     */
    FailureCategory selectById(@Param("id") Long id);
    
    /**
     * 查询全局故障分类
     *
     * @return 全局故障分类列表
     */
    List<FailureCategory> selectGlobal();
    
    /**
     * 根据功能ID查询故障分类
     *
     * @param functionId 功能ID
     * @return 故障分类列表
     */
    List<FailureCategory> selectByFunctionId(@Param("functionId") Long functionId);
    
    /**
     * 查询可用于指定功能的故障分类（全局分类+功能专属分类）
     *
     * @param functionId 功能ID
     * @return 故障分类列表
     */
    List<FailureCategory> selectAvailableForFunction(@Param("functionId") Long functionId);
    
    /**
     * 根据code查询故障分类
     *
     * @param code 编码
     * @return 故障分类
     */
    FailureCategory selectByCode(@Param("code") String code);
    
    /**
     * 新增故障分类
     *
     * @param failureCategory 故障分类
     * @return 影响行数
     */
    int insert(FailureCategory failureCategory);
    
    /**
     * 修改故障分类
     *
     * @param failureCategory 故障分类
     * @return 影响行数
     */
    int update(FailureCategory failureCategory);
    
    /**
     * 逻辑删除故障分类
     *
     * @param id ID
     * @param modifier 修改人
     * @return 影响行数
     */
    int deleteById(@Param("id") Long id, @Param("modifier") String modifier);
}
```

**FailureCategoryMapper.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.fema.mapper.FailureCategoryMapper">
    
    <resultMap id="BaseResultMap" type="com.example.fema.entity.FailureCategory">
        <id column="id" property="id" />
        <result column="code" property="code" />
        <result column="name" property="name" />
        <result column="isGlobal" property="isGlobal" />
        <result column="functionId" property="functionId" />
        <result column="creator" property="creator" />
        <result column="createTime" property="createTime" />
        <result column="modifier" property="modifier" />
        <result column="modifyTime" property="modifyTime" />
        <result column="isDeleted" property="isDeleted" />
    </resultMap>
    
    <sql id="Base_Column_List">
        id, code, name, isGlobal, functionId, creator, createTime, modifier, modifyTime, isDeleted
    </sql>
    
    <select id="selectAll" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_category
        WHERE isDeleted = 0
    </select>
    
    <select id="selectById" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_category
        WHERE id = #{id} AND isDeleted = 0
    </select>
    
    <select id="selectGlobal" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_category
        WHERE isGlobal = 1 AND isDeleted = 0
    </select>
    
    <select id="selectByFunctionId" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_category
        WHERE functionId = #{functionId} AND isDeleted = 0
    </select>
    
    <select id="selectAvailableForFunction" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_category
        WHERE (isGlobal = 1 OR functionId = #{functionId}) AND isDeleted = 0
    </select>
    
    <select id="selectByCode" parameterType="java.lang.String" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM failure_category
        WHERE code = #{code} AND isDeleted = 0
    </select>
    
    <insert id="insert" parameterType="com.example.fema.entity.FailureCategory" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO failure_category (
            code, name, isGlobal, functionId, creator, createTime, modifier, modifyTime, isDeleted
        ) VALUES (
            #{code}, #{name}, #{isGlobal}, #{functionId}, #{creator}, NOW(), #{modifier}, NOW(), 0
        )
    </insert>
    
    <update id="update" parameterType="com.example.fema.entity.FailureCategory">
        UPDATE failure_category
        <set>
            <if test="name != null">name = #{name},</if>
            <if test="isGlobal != null">isGlobal = #{isGlobal},</if>
            <if test="functionId != null">functionId = #{functionId},</if>
            modifier = #{modifier},
            modifyTime = NOW()
        </set>
        WHERE id = #{id} AND isDeleted = 0
    </update>
    
    <update id="deleteById">
        UPDATE failure_category
        SET isDeleted = 1,
            modifier = #{modifier},
            modifyTime = NOW()
        WHERE id = #{id} AND isDeleted = 0
    </update>
</mapper>
```

**OperationLogMapper.java**
```java
package com.example.fema.mapper;

import com.example.fema.entity.OperationLog;
import org.apache.ibatis.annotations.Param;
import java.util.List;
import java.util.Date;

/**
 * 操作日志Mapper接口
 */
public interface OperationLogMapper {
    
    /**
     * 查询操作日志
     *
     * @param operationType 操作类型
     * @param operationObject 操作对象
     * @param operationId 操作对象ID
     * @param operator 操作人
     * @param startTime 开始时间
     * @param endTime 结束时间
     * @return 操作日志列表
     */
    List<OperationLog> selectLogs(
        @Param("operationType") String operationType,
        @Param("operationObject") String operationObject,
        @Param("operationId") Long operationId,
        @Param("operator") String operator,
        @Param("startTime") Date startTime,
        @Param("endTime") Date endTime
    );
    
    /**
     * 查询指定对象的操作日志
     *
     * @param operationObject 操作对象
     * @param operationId 操作对象ID
     * @return 操作日志列表
     */
    List<OperationLog> selectByObjectAndId(
        @Param("operationObject") String operationObject,
        @Param("operationId") Long operationId
    );
    
    /**
     * 新增操作日志
     *
     * @param operationLog 操作日志
     * @return 影响行数
     */
    int insert(OperationLog operationLog);
}
```

**OperationLogMapper.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.fema.mapper.OperationLogMapper">
    
    <resultMap id="BaseResultMap" type="com.example.fema.entity.OperationLog">
        <id column="id" property="id" />
        <result column="operationType" property="operationType" />
        <result column="operationObject" property="operationObject" />
        <result column="operationId" property="operationId" />
        <result column="operationContent" property="operationContent" />
        <result column="operator" property="operator" />
        <result column="operationTime" property="operationTime" />
    </resultMap>
    
    <sql id="Base_Column_List">
        id, operationType, operationObject, operationId, operationContent, operator, operationTime
    </sql>
    
    <select id="selectLogs" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM operation_log
        <where>
            <if test="operationType != null and operationType != ''">
                AND operationType = #{operationType}
            </if>
            <if test="operationObject != null and operationObject != ''">
                AND operationObject = #{operationObject}
            </if>
            <if test="operationId != null">
                AND operationId = #{operationId}
            </if>
            <if test="operator != null and operator != ''">
                AND operator = #{operator}
            </if>
            <if test="startTime != null">
                AND operationTime &gt;= #{startTime}
            </if>
            <if test="endTime != null">
                AND operationTime &lt;= #{endTime}
            </if>
        </where>
        ORDER BY operationTime DESC
    </select>
    
    <select id="selectByObjectAndId" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM operation_log
        WHERE operationObject = #{operationObject} AND operationId = #{operationId}
        ORDER BY operationTime DESC
    </select>
    
    <insert id="insert" parameterType="com.example.fema.entity.OperationLog" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO operation_log (
            operationType, operationObject, operationId, operationContent, operator, operationTime
        ) VALUES (
            #{operationType}, #{operationObject}, #{operationId}, #{operationContent}, #{operator}, NOW()
        )
    </insert>
</mapper>
```

### 5. 服务层接口和实现

**FunctionDomainService.java**
```java
package com.example.fema.service;

import com.example.fema.entity.FunctionDomain;
import com.example.fema.vo.FunctionTreeVO;
import java.util.List;

/**
 * 功能域/功能服务接口
 */
public interface FunctionDomainService {
    
    /**
     * 查询功能树
     *
     * @return 功能树
     */
    List<FunctionTreeVO> getTree();
    
    /**
     * 查询指定节点的子树
     *
     * @param parentId 父节点ID
     * @return 子树
     */
    List<FunctionTreeVO> getSubTree(Long parentId);
    
    /**
     * 新建功能域
     *
     * @param functionDomain 功能域信息
     * @param operator 操作人
     * @return 新建的功能域
     */
    FunctionDomain createFunctionDomain(FunctionDomain functionDomain, String operator);
    
    /**
     * 新建功能
     *
     * @param functionDomain 功能信息
     * @param operator 操作人
     * @return 新建的功能
     */
    FunctionDomain createFunction(FunctionDomain functionDomain, String operator);
    
    /**
     * 修改功能域/功能名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的功能域/功能
     */
    FunctionDomain updateName(Long id, String name, String operator);
    
    /**
     * 删除功能域/功能
     *
     * @param id ID
     * @param operator 操作人
     * @return 是否删除成功
     */
    boolean delete(Long id, String operator);
    
    /**
     * 根据ID查询功能域/功能
     *
     * @param id ID
     * @return 功能域/功能
     */
    FunctionDomain getById(Long id);
    
    /**
     * 生成功能域/功能完整路径
     *
     * @param functionDomain 功能域/功能
     * @return 完整路径
     */
    String generatePath(FunctionDomain functionDomain);
    
    /**
     * 生成功能域/功能编码
     *
     * @param path 路径
     * @return 编码
     */
    String generateCode(String path);
}
```

**FunctionDomainServiceImpl.java**
```java
package com.example.fema.service.impl;

import com.example.fema.constant.FunctionTypeEnum;
import com.example.fema.entity.FunctionDomain;
import com.example.fema.entity.OperationLog;
import com.example.fema.exception.BusinessException;
import com.example.fema.mapper.FunctionDomainMapper;
import com.example.fema.mapper.OperationLogMapper;
import com.example.fema.service.FunctionDomainService;
import com.example.fema.util.HashUtil;
import com.example.fema.util.TreeUtil;
import com.example.fema.vo.FunctionTreeVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Date;
import java.util.List;
import java.util.Objects;

/**
 * 功能域/功能服务实现
 */
@Service
public class FunctionDomainServiceImpl implements FunctionDomainService {
    
    @Autowired
    private FunctionDomainMapper functionDomainMapper;
    
    @Autowired
    private OperationLogMapper operationLogMapper;
    
    /**
     * 查询功能树
     *
     * @return 功能树
     */
    @Override
    public List<FunctionTreeVO> getTree() {
        List<FunctionDomain> allFunctions = functionDomainMapper.selectAll();
        return TreeUtil.buildFunctionTree(allFunctions, null);
    }
    
    /**
     * 查询指定节点的子树
     *
     * @param parentId 父节点ID
     * @return 子树
     */
    @Override
    public List<FunctionTreeVO> getSubTree(Long parentId) {
        List<FunctionDomain> allFunctions = functionDomainMapper.selectAll();
        return TreeUtil.buildFunctionTree(allFunctions, parentId);
    }
    
    /**
     * 新建功能域
     *
     * @param functionDomain 功能域信息
     * @param operator 操作人
     * @return 新建的功能域
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public FunctionDomain createFunctionDomain(FunctionDomain functionDomain, String operator) {
        // 设置类型为功能域
        functionDomain.setType(FunctionTypeEnum.DOMAIN.getCode());
        
        // 设置操作人信息
        functionDomain.setCreator(operator);
        functionDomain.setModifier(operator);
        functionDomain.setCreateTime(new Date());
        functionDomain.setModifyTime(new Date());
        functionDomain.setDataSource(0); // 新增
        functionDomain.setIsDeleted(0);
        
        // 设置层级
        if (functionDomain.getParentId() == null) {
            // 根节点
            functionDomain.setLevel(1);
        } else {
            // 检查父节点
            FunctionDomain parent = functionDomainMapper.selectById(functionDomain.getParentId());
            if (parent == null) {
                throw new BusinessException("父节点不存在");
            }
            if (!Objects.equals(parent.getType(), FunctionTypeEnum.DOMAIN.getCode())) {
                throw new BusinessException("功能下不能创建功能域");
            }
            functionDomain.setLevel(parent.getLevel() + 1);
        }
        
        // 生成路径和编码
        String path = generatePath(functionDomain);
        functionDomain.setPath(path);
        functionDomain.setCode(generateCode(path));
        
        // 保存功能域
        functionDomainMapper.insert(functionDomain);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("新增");
        log.setOperationObject("功能域");
        log.setOperationId(functionDomain.getId());
        log.setOperationContent("新增功能域：" + functionDomain.getName());
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return functionDomain;
    }
    
    /**
     * 新建功能
     *
     * @param functionDomain 功能信息
     * @param operator 操作人
     * @return 新建的功能
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public FunctionDomain createFunction(FunctionDomain functionDomain, String operator) {
        // 设置类型为功能
        functionDomain.setType(FunctionTypeEnum.FUNCTION.getCode());
        
        // 设置操作人信息
        functionDomain.setCreator(operator);
        functionDomain.setModifier(operator);
        functionDomain.setCreateTime(new Date());
        functionDomain.setModifyTime(new Date());
        functionDomain.setDataSource(0); // 新增
        functionDomain.setIsDeleted(0);
        
        // 检查父节点
        if (functionDomain.getParentId() == null) {
            throw new BusinessException("功能必须有父节点");
        }
        
        FunctionDomain parent = functionDomainMapper.selectById(functionDomain.getParentId());
        if (parent == null) {
            throw new BusinessException("父节点不存在");
        }
        if (!Objects.equals(parent.getType(), FunctionTypeEnum.DOMAIN.getCode())) {
            throw new BusinessException("只能在功能域下创建功能");
        }
        
        // 设置层级
        functionDomain.setLevel(parent.getLevel() + 1);
        
        // 生成路径和编码
        String path = generatePath(functionDomain);
        functionDomain.setPath(path);
        functionDomain.setCode(generateCode(path));
        
        // 保存功能
        functionDomainMapper.insert(functionDomain);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("新增");
        log.setOperationObject("功能");
        log.setOperationId(functionDomain.getId());
        log.setOperationContent("新增功能：" + functionDomain.getName());
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return functionDomain;
    }
    
    /**
     * 修改功能域/功能名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的功能域/功能
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public FunctionDomain updateName(Long id, String name, String operator) {
        // 获取原功能域/功能
        FunctionDomain functionDomain = functionDomainMapper.selectById(id);
        if (functionDomain == null) {
            throw new BusinessException("功能域/功能不存在");
        }
        
        // 记录原名称
        String oldName = functionDomain.getName();
        
        // 更新名称
        functionDomain.setName(name);
        functionDomain.setModifier(operator);
        functionDomain.setModifyTime(new Date());
        
        // 更新路径和编码
        String path = generatePath(functionDomain);
        functionDomain.setPath(path);
        functionDomain.setCode(generateCode(path));
        
        // 保存修改
        functionDomainMapper.update(functionDomain);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("修改");
        log.setOperationObject(functionDomain.getType() == FunctionTypeEnum.DOMAIN.getCode() ? "功能域" : "功能");
        log.setOperationId(id);
        log.setOperationContent("名称修改：" + oldName + " -> " + name);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return functionDomain;
    }
    
    /**
     * 删除功能域/功能
     *
     * @param id ID
     * @param operator 操作人
     * @return 是否删除成功
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean delete(Long id, String operator) {
        // 获取功能域/功能
        FunctionDomain functionDomain = functionDomainMapper.selectById(id);
        if (functionDomain == null) {
            throw new BusinessException("功能域/功能不存在");
        }
        
        // 检查是否有子节点
        int childCount = functionDomainMapper.countByParentId(id);
        if (childCount > 0) {
            throw new BusinessException("该节点下有子节点，不能删除");
        }
        
        // 逻辑删除
        functionDomainMapper.deleteById(id, operator);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("删除");
        log.setOperationObject(functionDomain.getType() == FunctionTypeEnum.DOMAIN.getCode() ? "功能域" : "功能");
        log.setOperationId(id);
        log.setOperationContent("删除：" + functionDomain.getName());
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return true;
    }
    
    /**
     * 根据ID查询功能域/功能
     *
     * @param id ID
     * @return 功能域/功能
     */
    @Override
    public FunctionDomain getById(Long id) {
        return functionDomainMapper.selectById(id);
    }
    
    /**
     * 生成功能域/功能完整路径
     *
     * @param functionDomain 功能域/功能
     * @return 完整路径
     */
    @Override
    public String generatePath(FunctionDomain functionDomain) {
        StringBuilder path = new StringBuilder();
        
        if (functionDomain.getParentId() != null) {
            FunctionDomain parent = functionDomainMapper.selectById(functionDomain.getParentId());
            if (parent != null) {
                path.append(parent.getPath()).append("/");
            }
        }
        
        path.append(functionDomain.getName());
        return path.toString();
    }
    
    /**
     * 生成功能域/功能编码
     *
     * @param path 路径
     * @return 编码
     */
    @Override
    public String generateCode(String path) {
        return HashUtil.generateFunctionCode(path);
    }
}
```

**FailureModeService.java**
```java
package com.example.fema.service;

import com.example.fema.entity.FailureMode;
import com.example.fema.vo.FailureTreeVO;
import java.util.List;

/**
 * 故障服务接口
 */
public interface FailureModeService {
    
    /**
     * 查询故障树
     *
     * @param functionId 功能ID
     * @return 故障树
     */
    List<FailureTreeVO> getFailureTree(Long functionId);
    
    /**
     * 新增故障
     *
     * @param failureMode 故障信息
     * @param operator 操作人
     * @return 新增的故障
     */
    FailureMode createFailureMode(FailureMode failureMode, String operator);
    
    /**
     * 修改故障名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的故障
     */
    FailureMode updateName(Long id, String name, String operator);
    
    /**
     * 修改故障分类
     *
     * @param id ID
     * @param categoryId 分类ID
     * @param operator 操作人
     * @return 修改后的故障
     */
    FailureMode updateCategory(Long id, Long categoryId, String operator);
    
    /**
     * 删除故障
     *
     * @param id ID
     * @param operator 操作人
     * @return 是否删除成功
     */
    boolean delete(Long id, String operator);
    
    /**
     * 根据ID查询故障
     *
     * @param id ID
     * @return 故障
     */
    FailureMode getById(Long id);
    
    /**
     * 根据功能ID查询故障
     *
     * @param functionId 功能ID
     * @return 故障列表
     */
    List<FailureMode> getByFunctionId(Long functionId);
    
    /**
     * 生成故障编码
     *
     * @param functionPath 功能路径
     * @param systemElement 系统元素
     * @param failureName 故障名称
     * @return 编码
     */
    String generateCode(String

## Prompt:
继续

## Response:
 继续提供剩余的代码实现：

**FailureModeService.java** (续)
```java
    /**
     * 生成故障编码
     *
     * @param functionPath 功能路径
     * @param systemElement 系统元素
     * @param failureName 故障名称
     * @return 编码
     */
    String generateCode(String functionPath, String systemElement, String failureName);
}
```

**FailureModeServiceImpl.java**
```java
package com.example.fema.service.impl;

import com.example.fema.entity.FailureCategory;
import com.example.fema.entity.FailureMode;
import com.example.fema.entity.FunctionDomain;
import com.example.fema.entity.OperationLog;
import com.example.fema.exception.BusinessException;
import com.example.fema.mapper.FailureCategoryMapper;
import com.example.fema.mapper.FailureModeMapper;
import com.example.fema.mapper.FunctionDomainMapper;
import com.example.fema.mapper.OperationLogMapper;
import com.example.fema.service.FailureModeService;
import com.example.fema.util.HashUtil;
import com.example.fema.util.TreeUtil;
import com.example.fema.vo.FailureTreeVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Date;
import java.util.List;

/**
 * 故障服务实现
 */
@Service
public class FailureModeServiceImpl implements FailureModeService {
    
    @Autowired
    private FailureModeMapper failureModeMapper;
    
    @Autowired
    private FailureCategoryMapper failureCategoryMapper;
    
    @Autowired
    private FunctionDomainMapper functionDomainMapper;
    
    @Autowired
    private OperationLogMapper operationLogMapper;
    
    /**
     * 查询故障树
     *
     * @param functionId 功能ID
     * @return 故障树
     */
    @Override
    public List<FailureTreeVO> getFailureTree(Long functionId) {
        // 获取功能
        FunctionDomain function = functionDomainMapper.selectById(functionId);
        if (function == null) {
            throw new BusinessException("功能不存在");
        }
        
        // 查询可用于该功能的分类（全局分类 + 功能专属分类）
        List<FailureCategory> categories = failureCategoryMapper.selectAvailableForFunction(functionId);
        
        // 查询该功能下的故障
        List<FailureMode> failures = failureModeMapper.selectByFunctionId(functionId);
        
        // 构建故障树
        return TreeUtil.buildFailureTree(categories, failures, functionId);
    }
    
    /**
     * 新增故障
     *
     * @param failureMode 故障信息
     * @param operator 操作人
     * @return 新增的故障
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public FailureMode createFailureMode(FailureMode failureMode, String operator) {
        // 检查功能是否存在
        FunctionDomain function = functionDomainMapper.selectById(failureMode.getFunctionId());
        if (function == null) {
            throw new BusinessException("功能不存在");
        }
        
        // 检查分类是否存在（如果有）
        if (failureMode.getFailureCategoryId() != null) {
            FailureCategory category = failureCategoryMapper.selectById(failureMode.getFailureCategoryId());
            if (category == null) {
                throw new BusinessException("故障分类不存在");
            }
            
            // 检查分类是否可用于该功能
            if (category.getIsGlobal() != 1 && !category.getFunctionId().equals(failureMode.getFunctionId())) {
                throw new BusinessException("故障分类不可用于该功能");
            }
        }
        
        // 设置操作人信息
        failureMode.setCreator(operator);
        failureMode.setModifier(operator);
        failureMode.setCreateTime(new Date());
        failureMode.setModifyTime(new Date());
        failureMode.setDataSource(0); // 新增
        failureMode.setIsDeleted(0);
        
        // 生成编码
        failureMode.setCode(generateCode(function.getPath(), failureMode.getSystemElement(), failureMode.getName()));
        
        // 保存故障
        failureModeMapper.insert(failureMode);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("新增");
        log.setOperationObject("故障");
        log.setOperationId(failureMode.getId());
        log.setOperationContent("新增故障：" + failureMode.getName());
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return failureMode;
    }
    
    /**
     * 修改故障名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的故障
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public FailureMode updateName(Long id, String name, String operator) {
        // 获取原故障
        FailureMode failureMode = failureModeMapper.selectById(id);
        if (failureMode == null) {
            throw new BusinessException("故障不存在");
        }
        
        // 记录原名称
        String oldName = failureMode.getName();
        
        // 更新名称
        failureMode.setName(name);
        failureMode.setModifier(operator);
        failureMode.setModifyTime(new Date());
        
        // 获取功能路径
        FunctionDomain function = functionDomainMapper.selectById(failureMode.getFunctionId());
        if (function == null) {
            throw new BusinessException("功能不存在");
        }
        
        // 更新编码
        failureMode.setCode(generateCode(function.getPath(), failureMode.getSystemElement(), name));
        
        // 保存修改
        failureModeMapper.update(failureMode);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("修改");
        log.setOperationObject("故障");
        log.setOperationId(id);
        log.setOperationContent("名称修改：" + oldName + " -> " + name);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return failureMode;
    }
    
    /**
     * 修改故障分类
     *
     * @param id ID
     * @param categoryId 分类ID
     * @param operator 操作人
     * @return 修改后的故障
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public FailureMode updateCategory(Long id, Long categoryId, String operator) {
        // 获取故障
        FailureMode failureMode = failureModeMapper.selectById(id);
        if (failureMode == null) {
            throw new BusinessException("故障不存在");
        }
        
        // 记录原分类ID
        Long oldCategoryId = failureMode.getFailureCategoryId();
        
        // 检查分类是否存在（如果有）
        if (categoryId != null) {
            FailureCategory category = failureCategoryMapper.selectById(categoryId);
            if (category == null) {
                throw new BusinessException("故障分类不存在");
            }
            
            // 检查分类是否可用于该功能
            if (category.getIsGlobal() != 1 && !category.getFunctionId().equals(failureMode.getFunctionId())) {
                throw new BusinessException("故障分类不可用于该功能");
            }
        }
        
        // 更新分类
        failureMode.setFailureCategoryId(categoryId);
        failureMode.setModifier(operator);
        failureMode.setModifyTime(new Date());
        
        // 保存修改
        failureModeMapper.update(failureMode);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("修改");
        log.setOperationObject("故障");
        log.setOperationId(id);
        log.setOperationContent("分类修改：" + oldCategoryId + " -> " + categoryId);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return failureMode;
    }
    
    /**
     * 删除故障
     *
     * @param id ID
     * @param operator 操作人
     * @return 是否删除成功
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean delete(Long id, String operator) {
        // 获取故障
        FailureMode failureMode = failureModeMapper.selectById(id);
        if (failureMode == null) {
            throw new BusinessException("故障不存在");
        }
        
        // 逻辑删除
        failureModeMapper.deleteById(id, operator);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("删除");
        log.setOperationObject("故障");
        log.setOperationId(id);
        log.setOperationContent("删除故障：" + failureMode.getName());
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return true;
    }
    
    /**
     * 根据ID查询故障
     *
     * @param id ID
     * @return 故障
     */
    @Override
    public FailureMode getById(Long id) {
        return failureModeMapper.selectById(id);
    }
    
    /**
     * 根据功能ID查询故障
     *
     * @param functionId 功能ID
     * @return 故障列表
     */
    @Override
    public List<FailureMode> getByFunctionId(Long functionId) {
        return failureModeMapper.selectByFunctionId(functionId);
    }
    
    /**
     * 生成故障编码
     *
     * @param functionPath 功能路径
     * @param systemElement 系统元素
     * @param failureName 故障名称
     * @return 编码
     */
    @Override
    public String generateCode(String functionPath, String systemElement, String failureName) {
        return HashUtil.generateFailureCode(functionPath, systemElement, failureName);
    }
}
```

**FailureCategoryService.java**
```java
package com.example.fema.service;

import com.example.fema.entity.FailureCategory;
import java.util.List;

/**
 * 故障分类服务接口
 */
public interface FailureCategoryService {
    
    /**
     * 新增故障分类
     *
     * @param failureCategory 故障分类信息
     * @param operator 操作人
     * @return 新增的故障分类
     */
    FailureCategory createCategory(FailureCategory failureCategory, String operator);
    
    /**
     * 修改故障分类名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的故障分类
     */
    FailureCategory updateName(Long id, String name, String operator);
    
    /**
     * 删除故障分类
     *
     * @param id ID
     * @param operator 操作人
     * @return 是否删除成功
     */
    boolean delete(Long id, String operator);
    
    /**
     * 查询可用于指定功能的故障分类
     *
     * @param functionId 功能ID
     * @return 故障分类列表
     */
    List<FailureCategory> getAvailableForFunction(Long functionId);
    
    /**
     * 根据ID查询故障分类
     *
     * @param id ID
     * @return 故障分类
     */
    FailureCategory getById(Long id);
}
```

**FailureCategoryServiceImpl.java**
```java
package com.example.fema.service.impl;

import com.example.fema.entity.FailureCategory;
import com.example.fema.entity.FailureMode;
import com.example.fema.entity.FunctionDomain;
import com.example.fema.entity.OperationLog;
import com.example.fema.exception.BusinessException;
import com.example.fema.mapper.FailureCategoryMapper;
import com.example.fema.mapper.FailureModeMapper;
import com.example.fema.mapper.FunctionDomainMapper;
import com.example.fema.mapper.OperationLogMapper;
import com.example.fema.service.FailureCategoryService;
import com.example.fema.util.HashUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Date;
import java.util.List;

/**
 * 故障分类服务实现
 */
@Service
public class FailureCategoryServiceImpl implements FailureCategoryService {
    
    @Autowired
    private FailureCategoryMapper failureCategoryMapper;
    
    @Autowired
    private FailureModeMapper failureModeMapper;
    
    @Autowired
    private FunctionDomainMapper functionDomainMapper;
    
    @Autowired
    private OperationLogMapper operationLogMapper;
    
    /**
     * 新增故障分类
     *
     * @param failureCategory 故障分类信息
     * @param operator 操作人
     * @return 新增的故障分类
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public FailureCategory createCategory(FailureCategory failureCategory, String operator) {
        // 检查功能是否存在（如果不是全局分类）
        if (failureCategory.getIsGlobal() != 1 && failureCategory.getFunctionId() != null) {
            FunctionDomain function = functionDomainMapper.selectById(failureCategory.getFunctionId());
            if (function == null) {
                throw new BusinessException("功能不存在");
            }
        }
        
        // 设置操作人信息
        failureCategory.setCreator(operator);
        failureCategory.setModifier(operator);
        failureCategory.setCreateTime(new Date());
        failureCategory.setModifyTime(new Date());
        failureCategory.setIsDeleted(0);
        
        // 生成编码
        failureCategory.setCode(HashUtil.md5(failureCategory.getName() + "_" + (failureCategory.getIsGlobal() == 1 ? "global" : failureCategory.getFunctionId())));
        
        // 保存故障分类
        failureCategoryMapper.insert(failureCategory);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("新增");
        log.setOperationObject("故障分类");
        log.setOperationId(failureCategory.getId());
        log.setOperationContent("新增故障分类：" + failureCategory.getName());
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return failureCategory;
    }
    
    /**
     * 修改故障分类名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的故障分类
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public FailureCategory updateName(Long id, String name, String operator) {
        // 获取原故障分类
        FailureCategory failureCategory = failureCategoryMapper.selectById(id);
        if (failureCategory == null) {
            throw new BusinessException("故障分类不存在");
        }
        
        // 记录原名称
        String oldName = failureCategory.getName();
        
        // 更新名称
        failureCategory.setName(name);
        failureCategory.setModifier(operator);
        failureCategory.setModifyTime(new Date());
        
        // 更新编码
        failureCategory.setCode(HashUtil.md5(name + "_" + (failureCategory.getIsGlobal() == 1 ? "global" : failureCategory.getFunctionId())));
        
        // 保存修改
        failureCategoryMapper.update(failureCategory);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("修改");
        log.setOperationObject("故障分类");
        log.setOperationId(id);
        log.setOperationContent("名称修改：" + oldName + " -> " + name);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return failureCategory;
    }
    
    /**
     * 删除故障分类
     *
     * @param id ID
     * @param operator 操作人
     * @return 是否删除成功
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public boolean delete(Long id, String operator) {
        // 获取故障分类
        FailureCategory failureCategory = failureCategoryMapper.selectById(id);
        if (failureCategory == null) {
            throw new BusinessException("故障分类不存在");
        }
        
        // 检查是否有关联的故障
        List<FailureMode> failures = failureModeMapper.selectByCategoryId(id);
        if (failures != null && !failures.isEmpty()) {
            throw new BusinessException("该分类下有关联的故障，不能删除");
        }
        
        // 逻辑删除
        failureCategoryMapper.deleteById(id, operator);
        
        // 记录操作日志
        OperationLog log = new OperationLog();
        log.setOperationType("删除");
        log.setOperationObject("故障分类");
        log.setOperationId(id);
        log.setOperationContent("删除故障分类：" + failureCategory.getName());
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return true;
    }
    
    /**
     * 查询可用于指定功能的故障分类
     *
     * @param functionId 功能ID
     * @return 故障分类列表
     */
    @Override
    public List<FailureCategory> getAvailableForFunction(Long functionId) {
        return failureCategoryMapper.selectAvailableForFunction(functionId);
    }
    
    /**
     * 根据ID查询故障分类
     *
     * @param id ID
     * @return 故障分类
     */
    @Override
    public FailureCategory getById(Long id) {
        return failureCategoryMapper.selectById(id);
    }
}
```

**OperationLogService.java**
```java
package com.example.fema.service;

import com.example.fema.entity.OperationLog;
import java.util.Date;
import java.util.List;

/**
 * 操作日志服务接口
 */
public interface OperationLogService {
    
    /**
     * 添加操作日志
     *
     * @param operationLog 操作日志
     * @return 操作日志
     */
    OperationLog addLog(OperationLog operationLog);
    
    /**
     * 查询操作日志
     *
     * @param operationType 操作类型
     * @param operationObject 操作对象
     * @param operationId 操作对象ID
     * @param operator 操作人
     * @param startTime 开始时间
     * @param endTime 结束时间
     * @return 操作日志列表
     */
    List<OperationLog> queryLogs(
        String operationType,
        String operationObject,
        Long operationId,
        String operator,
        Date startTime,
        Date endTime
    );
    
    /**
     * 查询指定对象的操作日志
     *
     * @param operationObject 操作对象
     * @param operationId 操作对象ID
     * @return 操作日志列表
     */
    List<OperationLog> queryObjectLogs(String operationObject, Long operationId);
}
```

**OperationLogServiceImpl.java**
```java
package com.example.fema.service.impl;

import com.example.fema.entity.OperationLog;
import com.example.fema.mapper.OperationLogMapper;
import com.example.fema.service.OperationLogService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.List;

/**
 * 操作日志服务实现
 */
@Service
public class OperationLogServiceImpl implements OperationLogService {
    
    @Autowired
    private OperationLogMapper operationLogMapper;
    
    /**
     * 添加操作日志
     *
     * @param operationLog 操作日志
     * @return 操作日志
     */
    @Override
    public OperationLog addLog(OperationLog operationLog) {
        operationLogMapper.insert(operationLog);
        return operationLog;
    }
    
    /**
     * 查询操作日志
     *
     * @param operationType 操作类型
     * @param operationObject 操作对象
     * @param operationId 操作对象ID
     * @param operator 操作人
     * @param startTime 开始时间
     * @param endTime 结束时间
     * @return 操作日志列表
     */
    @Override
    public List<OperationLog> queryLogs(
        String operationType,
        String operationObject,
        Long operationId,
        String operator,
        Date startTime,
        Date endTime
    ) {
        return operationLogMapper.selectLogs(
            operationType,
            operationObject,
            operationId,
            operator,
            startTime,
            endTime
        );
    }
    
    /**
     * 查询指定对象的操作日志
     *
     * @param operationObject 操作对象
     * @param operationId 操作对象ID
     * @return 操作日志列表
     */
    @Override
    public List<OperationLog> queryObjectLogs(String operationObject, Long operationId) {
        return operationLogMapper.selectByObjectAndId(operationObject, operationId);
    }
}
```

**DataMigrationService.java**
```java
package com.example.fema.service;

import com.example.fema.dto.FemaDataDTO;
import java.util.List;

/**
 * 数据迁移服务接口
 */
public interface DataMigrationService {
    
    /**
     * 迁移旧FEMA表数据
     *
     * @param dataList 旧数据列表
     * @param operator 操作人
     * @return 迁移数量
     */
    int migrateData(List<FemaDataDTO> dataList, String operator);
}
```

**DataMigrationServiceImpl.java**
```java
package com.example.fema.service.impl;

import com.example.fema.constant.FunctionTypeEnum;
import com.example.fema.dto.FemaDataDTO;
import com.example.fema.entity.FailureMode;
import com.example.fema.entity.FunctionDomain;
import com.example.fema.entity.OperationLog;
import com.example.fema.entity.SystemElement;
import com.example.fema.mapper.FailureModeMapper;
import com.example.fema.mapper.FunctionDomainMapper;
import com.example.fema.mapper.OperationLogMapper;
import com.example.fema.mapper.SystemElementMapper;
import com.example.fema.service.DataMigrationService;
import com.example.fema.util.HashUtil;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 数据迁移服务实现
 */
@Service
public class DataMigrationServiceImpl implements DataMigrationService {
    
    @Autowired
    private FunctionDomainMapper functionDomainMapper;
    
    @Autowired
    private FailureModeMapper failureModeMapper;
    
    @Autowired
    private SystemElementMapper systemElementMapper;
    
    @Autowired
    private OperationLogMapper operationLogMapper;
    
    /**
     * 迁移旧FEMA表数据
     *
     * @param dataList 旧数据列表
     * @param operator 操作人
     * @return 迁移数量
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public int migrateData(List<FemaDataDTO> dataList, String operator) {
        if (dataList == null || dataList.isEmpty()) {
            return 0;
        }
        
        int count = 0;
        Map<String, FunctionDomain> domainMap = new HashMap<>();
        Map<String, SystemElement> elementMap = new HashMap<>();
        
        for (FemaDataDTO data : dataList) {
            try {
                // 1. 处理功能域和功能
                String domainPath1 = data.getBusinessName();
                String domainPath2 = domainPath1 + "/" + data.getFunctionName();
                String functionPath = domainPath2 + "/" + data.getSubFunctionName();
                
                // 第一层功能域
                FunctionDomain domain1 = domainMap.get(domainPath1);
                if (domain1 == null) {
                    domain1 = createOrGetFunctionDomain(domainPath1, null, data.getBusinessName(), operator);
                    domainMap.put(domainPath1, domain1);
                }
                
                // 第二层功能域
                FunctionDomain domain2 = domainMap.get(domainPath2);
                if (domain2 == null) {
                    domain2 = createOrGetFunctionDomain(domainPath2, domain1.getId(), data.getFunctionName(), operator);
                    domainMap.put(domainPath2, domain2);
                }
                
                // 功能
                FunctionDomain function = domainMap.get(functionPath);
                if (function == null) {
                    function = createOrGetFunction(functionPath, domain2.getId(), data.getSubFunctionName(), operator);
                    domainMap.put(functionPath, function);
                }
                
                // 2. 处理系统元素
                String elementKey = data.getSystemElement();
                SystemElement element = elementMap.get(elementKey);
                if (element == null) {
                    element = createOrGetSystemElement(elementKey, functionPath, operator);
                    elementMap.put(elementKey, element);
                }
                
                // 3. 处理故障
                createOrGetFailureMode(data.getFailureMode(), function.getId(), element.getId(), element.getName(), functionPath, operator);
                
                count++;
            } catch (Exception e) {
                // 记录异常但继续处理
                OperationLog log = new OperationLog();
                log.setOperationType("迁移");
                log.setOperationObject("FEMA数据");
                log.setOperationId(0L);
                log.setOperationContent("迁移数据异常：" + e.getMessage() + "，数据：" + data.toString());
                log.setOperator(operator);
                log.setOperationTime(new Date());
                operationLogMapper.insert(log);
            }
        }
        
        // 记录迁移完成日志
        OperationLog log = new OperationLog();
        log.setOperationType("迁移");
        log.setOperationObject("FEMA数据");
        log.setOperationId(0L);
        log.setOperationContent("完成数据迁移，总数：" + dataList.size() + "，成功：" + count);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return count;
    }
    
    /**
     * 创建或获取功能域
     *
     * @param path 路径
     * @param parentId 父ID
     * @param name 名称
     * @param operator 操作人
     * @return 功能域
     */
    private FunctionDomain createOrGetFunctionDomain(String path, Long parentId, String name, String operator) {
        // 根据路径生成编码
        String code = HashUtil.generateFunctionCode(path);
        
        // 查询是否已存在
        FunctionDomain domain = functionDomainMapper.selectByCode(code);
        if (domain != null) {
            return domain;
        }
        
        // 创建新功能域
        domain = new FunctionDomain();
        domain.setCode(code);
        domain.setName(name);
        domain.setType(FunctionTypeEnum.DOMAIN.getCode());
        domain.setParentId(parentId);
        domain.setPath(path);
        domain.setLevel(parentId == null ? 1 : 2);
        domain.setOldPath(path

## Prompt:
继续

## Response:
 继续提供剩余的代码实现：

**DataMigrationServiceImpl.java** (续)
```java
        domain.setOldPath(path);
        domain.setDataSource(1); // 迁移
        domain.setCreator(operator);
        domain.setModifier(operator);
        domain.setCreateTime(new Date());
        domain.setModifyTime(new Date());
        domain.setIsDeleted(0);
        
        functionDomainMapper.insert(domain);
        
        // 记录日志
        OperationLog log = new OperationLog();
        log.setOperationType("迁移");
        log.setOperationObject("功能域");
        log.setOperationId(domain.getId());
        log.setOperationContent("迁移功能域：" + name);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return domain;
    }
    
    /**
     * 创建或获取功能
     *
     * @param path 路径
     * @param parentId 父ID
     * @param name 名称
     * @param operator 操作人
     * @return 功能
     */
    private FunctionDomain createOrGetFunction(String path, Long parentId, String name, String operator) {
        // 根据路径生成编码
        String code = HashUtil.generateFunctionCode(path);
        
        // 查询是否已存在
        FunctionDomain function = functionDomainMapper.selectByCode(code);
        if (function != null) {
            return function;
        }
        
        // 创建新功能
        function = new FunctionDomain();
        function.setCode(code);
        function.setName(name);
        function.setType(FunctionTypeEnum.FUNCTION.getCode());
        function.setParentId(parentId);
        function.setPath(path);
        function.setLevel(3); // 固定为第三层
        function.setOldPath(path);
        function.setDataSource(1); // 迁移
        function.setCreator(operator);
        function.setModifier(operator);
        function.setCreateTime(new Date());
        function.setModifyTime(new Date());
        function.setIsDeleted(0);
        
        functionDomainMapper.insert(function);
        
        // 记录日志
        OperationLog log = new OperationLog();
        log.setOperationType("迁移");
        log.setOperationObject("功能");
        log.setOperationId(function.getId());
        log.setOperationContent("迁移功能：" + name);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return function;
    }
    
    /**
     * 创建或获取系统元素
     *
     * @param name 名称
     * @param functionPath 功能路径
     * @param operator 操作人
     * @return 系统元素
     */
    private SystemElement createOrGetSystemElement(String name, String functionPath, String operator) {
        if (!StringUtils.hasText(name)) {
            name = "未知元素";
        }
        
        // 生成编码
        String code = HashUtil.md5(name);
        
        // 查询是否已存在
        SystemElement element = systemElementMapper.selectByCode(code);
        if (element != null) {
            return element;
        }
        
        // 创建新系统元素
        element = new SystemElement();
        element.setCode(code);
        element.setName(name);
        element.setOldPath(functionPath + "/" + name);
        element.setDataSource(1); // 迁移
        element.setCreator(operator);
        element.setModifier(operator);
        element.setCreateTime(new Date());
        element.setModifyTime(new Date());
        element.setIsDeleted(0);
        
        systemElementMapper.insert(element);
        
        // 记录日志
        OperationLog log = new OperationLog();
        log.setOperationType("迁移");
        log.setOperationObject("系统元素");
        log.setOperationId(element.getId());
        log.setOperationContent("迁移系统元素：" + name);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return element;
    }
    
    /**
     * 创建或获取故障
     *
     * @param name 名称
     * @param functionId 功能ID
     * @param elementId 系统元素ID
     * @param elementName 系统元素名称
     * @param functionPath 功能路径
     * @param operator 操作人
     * @return 故障
     */
    private FailureMode createOrGetFailureMode(String name, Long functionId, Long elementId, String elementName, String functionPath, String operator) {
        if (!StringUtils.hasText(name)) {
            name = "未知故障";
        }
        
        // 生成编码
        String code = HashUtil.generateFailureCode(functionPath, elementName, name);
        
        // 查询是否已存在
        FailureMode failureMode = failureModeMapper.selectByCode(code);
        if (failureMode != null) {
            return failureMode;
        }
        
        // 创建新故障
        failureMode = new FailureMode();
        failureMode.setCode(code);
        failureMode.setName(name);
        failureMode.setFunctionId(functionId);
        failureMode.setSystemElement(elementName);
        failureMode.setSystemElementId(elementId);
        failureMode.setFailureCategoryId(null); // 初始无分类
        failureMode.setOldPath(functionPath + "/" + elementName + "/" + name);
        failureMode.setDataSource(1); // 迁移
        failureMode.setCreator(operator);
        failureMode.setModifier(operator);
        failureMode.setCreateTime(new Date());
        failureMode.setModifyTime(new Date());
        failureMode.setIsDeleted(0);
        
        failureModeMapper.insert(failureMode);
        
        // 记录日志
        OperationLog log = new OperationLog();
        log.setOperationType("迁移");
        log.setOperationObject("故障");
        log.setOperationId(failureMode.getId());
        log.setOperationContent("迁移故障：" + name);
        log.setOperator(operator);
        log.setOperationTime(new Date());
        operationLogMapper.insert(log);
        
        return failureMode;
    }
}
```

### 6. 控制器实现

**FunctionDomainController.java**
```java
package com.example.fema.controller;

import com.example.fema.entity.FunctionDomain;
import com.example.fema.service.FunctionDomainService;
import com.example.fema.vo.FunctionTreeVO;
import com.example.fema.vo.ResultVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * 功能域/功能控制器
 */
@RestController
@RequestMapping("/api/function")
public class FunctionDomainController {
    
    @Autowired
    private FunctionDomainService functionDomainService;
    
    /**
     * 查询功能树
     *
     * @return 功能树
     */
    @GetMapping("/tree")
    public ResultVO<List<FunctionTreeVO>> getTree() {
        List<FunctionTreeVO> tree = functionDomainService.getTree();
        return ResultVO.success(tree);
    }
    
    /**
     * 查询指定节点的子树
     *
     * @param parentId 父节点ID
     * @return 子树
     */
    @GetMapping("/subtree/{parentId}")
    public ResultVO<List<FunctionTreeVO>> getSubTree(@PathVariable Long parentId) {
        List<FunctionTreeVO> tree = functionDomainService.getSubTree(parentId);
        return ResultVO.success(tree);
    }
    
    /**
     * 新建功能域
     *
     * @param functionDomain 功能域信息
     * @param operator 操作人
     * @return 新建的功能域
     */
    @PostMapping("/domain")
    public ResultVO<FunctionDomain> createFunctionDomain(
            @RequestBody FunctionDomain functionDomain,
            @RequestParam String operator) {
        FunctionDomain result = functionDomainService.createFunctionDomain(functionDomain, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 新建功能
     *
     * @param functionDomain 功能信息
     * @param operator 操作人
     * @return 新建的功能
     */
    @PostMapping("/function")
    public ResultVO<FunctionDomain> createFunction(
            @RequestBody FunctionDomain functionDomain,
            @RequestParam String operator) {
        FunctionDomain result = functionDomainService.createFunction(functionDomain, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 修改功能域/功能名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的功能域/功能
     */
    @PutMapping("/{id}/name")
    public ResultVO<FunctionDomain> updateName(
            @PathVariable Long id,
            @RequestParam String name,
            @RequestParam String operator) {
        FunctionDomain result = functionDomainService.updateName(id, name, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 删除功能域/功能
     *
     * @param id ID
     * @param operator 操作人
     * @return 操作结果
     */
    @DeleteMapping("/{id}")
    public ResultVO<Boolean> delete(
            @PathVariable Long id,
            @RequestParam String operator) {
        boolean result = functionDomainService.delete(id, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 根据ID查询功能域/功能
     *
     * @param id ID
     * @return 功能域/功能
     */
    @GetMapping("/{id}")
    public ResultVO<FunctionDomain> getById(@PathVariable Long id) {
        FunctionDomain result = functionDomainService.getById(id);
        return ResultVO.success(result);
    }
}
```

**FailureModeController.java**
```java
package com.example.fema.controller;

import com.example.fema.entity.FailureMode;
import com.example.fema.service.FailureModeService;
import com.example.fema.vo.FailureTreeVO;
import com.example.fema.vo.ResultVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * 故障控制器
 */
@RestController
@RequestMapping("/api/failure")
public class FailureModeController {
    
    @Autowired
    private FailureModeService failureModeService;
    
    /**
     * 查询故障树
     *
     * @param functionId 功能ID
     * @return 故障树
     */
    @GetMapping("/tree/{functionId}")
    public ResultVO<List<FailureTreeVO>> getFailureTree(@PathVariable Long functionId) {
        List<FailureTreeVO> tree = failureModeService.getFailureTree(functionId);
        return ResultVO.success(tree);
    }
    
    /**
     * 新增故障
     *
     * @param failureMode 故障信息
     * @param operator 操作人
     * @return 新增的故障
     */
    @PostMapping
    public ResultVO<FailureMode> createFailureMode(
            @RequestBody FailureMode failureMode,
            @RequestParam String operator) {
        FailureMode result = failureModeService.createFailureMode(failureMode, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 修改故障名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的故障
     */
    @PutMapping("/{id}/name")
    public ResultVO<FailureMode> updateName(
            @PathVariable Long id,
            @RequestParam String name,
            @RequestParam String operator) {
        FailureMode result = failureModeService.updateName(id, name, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 修改故障分类
     *
     * @param id ID
     * @param categoryId 分类ID
     * @param operator 操作人
     * @return 修改后的故障
     */
    @PutMapping("/{id}/category")
    public ResultVO<FailureMode> updateCategory(
            @PathVariable Long id,
            @RequestParam Long categoryId,
            @RequestParam String operator) {
        FailureMode result = failureModeService.updateCategory(id, categoryId, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 删除故障
     *
     * @param id ID
     * @param operator 操作人
     * @return 操作结果
     */
    @DeleteMapping("/{id}")
    public ResultVO<Boolean> delete(
            @PathVariable Long id,
            @RequestParam String operator) {
        boolean result = failureModeService.delete(id, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 根据ID查询故障
     *
     * @param id ID
     * @return 故障
     */
    @GetMapping("/{id}")
    public ResultVO<FailureMode> getById(@PathVariable Long id) {
        FailureMode result = failureModeService.getById(id);
        return ResultVO.success(result);
    }
    
    /**
     * 根据功能ID查询故障
     *
     * @param functionId 功能ID
     * @return 故障列表
     */
    @GetMapping("/function/{functionId}")
    public ResultVO<List<FailureMode>> getByFunctionId(@PathVariable Long functionId) {
        List<FailureMode> result = failureModeService.getByFunctionId(functionId);
        return ResultVO.success(result);
    }
}
```

**FailureCategoryController.java**
```java
package com.example.fema.controller;

import com.example.fema.entity.FailureCategory;
import com.example.fema.service.FailureCategoryService;
import com.example.fema.vo.ResultVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * 故障分类控制器
 */
@RestController
@RequestMapping("/api/failure/category")
public class FailureCategoryController {
    
    @Autowired
    private FailureCategoryService failureCategoryService;
    
    /**
     * 新增故障分类
     *
     * @param failureCategory 故障分类信息
     * @param operator 操作人
     * @return 新增的故障分类
     */
    @PostMapping
    public ResultVO<FailureCategory> createCategory(
            @RequestBody FailureCategory failureCategory,
            @RequestParam String operator) {
        FailureCategory result = failureCategoryService.createCategory(failureCategory, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 修改故障分类名称
     *
     * @param id ID
     * @param name 新名称
     * @param operator 操作人
     * @return 修改后的故障分类
     */
    @PutMapping("/{id}/name")
    public ResultVO<FailureCategory> updateName(
            @PathVariable Long id,
            @RequestParam String name,
            @RequestParam String operator) {
        FailureCategory result = failureCategoryService.updateName(id, name, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 删除故障分类
     *
     * @param id ID
     * @param operator 操作人
     * @return 操作结果
     */
    @DeleteMapping("/{id}")
    public ResultVO<Boolean> delete(
            @PathVariable Long id,
            @RequestParam String operator) {
        boolean result = failureCategoryService.delete(id, operator);
        return ResultVO.success(result);
    }
    
    /**
     * 查询可用于指定功能的故障分类
     *
     * @param functionId 功能ID
     * @return 故障分类列表
     */
    @GetMapping("/function/{functionId}")
    public ResultVO<List<FailureCategory>> getAvailableForFunction(@PathVariable Long functionId) {
        List<FailureCategory> result = failureCategoryService.getAvailableForFunction(functionId);
        return ResultVO.success(result);
    }
    
    /**
     * 根据ID查询故障分类
     *
     * @param id ID
     * @return 故障分类
     */
    @GetMapping("/{id}")
    public ResultVO<FailureCategory> getById(@PathVariable Long id) {
        FailureCategory result = failureCategoryService.getById(id);
        return ResultVO.success(result);
    }
}
```

**OperationLogController.java**
```java
package com.example.fema.controller;

import com.example.fema.entity.OperationLog;
import com.example.fema.service.OperationLogService;
import com.example.fema.vo.ResultVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.web.bind.annotation.*;

import java.util.Date;
import java.util.List;

/**
 * 操作日志控制器
 */
@RestController
@RequestMapping("/api/log")
public class OperationLogController {
    
    @Autowired
    private OperationLogService operationLogService;
    
    /**
     * 查询操作日志
     *
     * @param operationType 操作类型
     * @param operationObject 操作对象
     * @param operationId 操作对象ID
     * @param operator 操作人
     * @param startTime 开始时间
     * @param endTime 结束时间
     * @return 操作日志列表
     */
    @GetMapping
    public ResultVO<List<OperationLog>> queryLogs(
            @RequestParam(required = false) String operationType,
            @RequestParam(required = false) String operationObject,
            @RequestParam(required = false) Long operationId,
            @RequestParam(required = false) String operator,
            @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss") Date startTime,
            @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss") Date endTime) {
        List<OperationLog> logs = operationLogService.queryLogs(
            operationType,
            operationObject,
            operationId,
            operator,
            startTime,
            endTime
        );
        return ResultVO.success(logs);
    }
    
    /**
     * 查询指定对象的操作日志
     *
     * @param operationObject 操作对象
     * @param operationId 操作对象ID
     * @return 操作日志列表
     */
    @GetMapping("/object")
    public ResultVO<List<OperationLog>> queryObjectLogs(
            @RequestParam String operationObject,
            @RequestParam Long operationId) {
        List<OperationLog> logs = operationLogService.queryObjectLogs(operationObject, operationId);
        return ResultVO.success(logs);
    }
}
```

**DataMigrationController.java**
```java
package com.example.fema.controller;

import com.example.fema.dto.FemaDataDTO;
import com.example.fema.service.DataMigrationService;
import com.example.fema.vo.ResultVO;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * 数据迁移控制器
 */
@RestController
@RequestMapping("/api/migration")
public class DataMigrationController {
    
    @Autowired
    private DataMigrationService dataMigrationService;
    
    /**
     * 迁移旧FEMA表数据
     *
     * @param dataList 旧数据列表
     * @param operator 操作人
     * @return 迁移数量
     */
    @PostMapping
    public ResultVO<Integer> migrateData(
            @RequestBody List<FemaDataDTO> dataList,
            @RequestParam String operator) {
        int count = dataMigrationService.migrateData(dataList, operator);
        return ResultVO.success("迁移成功", count);
    }
}
```

### 7. 常量和枚举

**FunctionTypeEnum.java**
```java
package com.example.fema.constant;

/**
 * 功能类型枚举
 */
public enum FunctionTypeEnum {
    
    /**
     * 功能域
     */
    DOMAIN(1, "功能域"),
    
    /**
     * 功能
     */
    FUNCTION(2, "功能");
    
    private final Integer code;
    private final String desc;
    
    FunctionTypeEnum(Integer code, String desc) {
        this.code = code;
        this.desc = desc;
    }
    
    public Integer getCode() {
        return code;
    }
    
    public String getDesc() {
        return desc;
    }
}
```

### 8. DTO 对象

**FemaDataDTO.java**
```java
package com.example.fema.dto;

import lombok.Data;

/**
 * 旧FEMA表数据传输对象
 */
@Data
public class FemaDataDTO {
    /**
     * 业务名称（一级功能域）
     */
    private String businessName;
    
    /**
     * 功能名称（二级功能域）
     */
    private String functionName;
    
    /**
     * 子功能名称（功能）
     */
    private String subFunctionName;
    
    /**
     * 系统元素
     */
    private String systemElement;
    
    /**
     * 故障模式
     */
    private String failureMode;
}
```

### 9. 异常处理

**BusinessException.java**
```java
package com.example.fema.exception;

/**
 * 业务异常
 */
public class BusinessException extends RuntimeException {
    
    public BusinessException(String message) {
        super(message);
    }
    
    public BusinessException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

**GlobalExceptionHandler.java**
```java
package com.example.fema.exception;

import com.example.fema.vo.ResultVO;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

/**
 * 全局异常处理器
 */
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    /**
     * 处理业务异常
     *
     * @param e 业务异常
     * @return 错误响应
     */
    @ExceptionHandler(BusinessException.class)
    public ResultVO<Object> handleBusinessException(BusinessException e) {
        return ResultVO.error(400, e.getMessage());
    }
    
    /**
     * 处理其他异常
     *
     * @param e 异常
     * @return 错误响应
     */
    @ExceptionHandler(Exception.class)
    public ResultVO<Object> handleException(Exception e) {
        return ResultVO.error(500, "系统内部错误：" + e.getMessage());
    }
}
```

### 10. SystemElementMapper.java

```java
package com.example.fema.mapper;

import com.example.fema.entity.SystemElement;
import org.apache.ibatis.annotations.Param;
import java.util.List;

/**
 * 系统元素Mapper接口
 */
public interface SystemElementMapper {
    
    /**
     * 查询全部系统元素
     *
     * @return 系统元素列表
     */
    List<SystemElement> selectAll();
    
    /**
     * 根据ID查询系统元素
     *
     * @param id ID
     * @return 系统元素
     */
    SystemElement selectById(@Param("id") Long id);
    
    /**
     * 根据code查询系统元素
     *
     * @param code 编码
     * @return 系统元素
     */
    SystemElement selectByCode(@Param("code") String code);
    
    /**
     * 新增系统元素
     *
     * @param systemElement 系统元素
     * @return 影响行数
     */
    int insert(SystemElement systemElement);
    
    /**
     * 修改系统元素
     *
     * @param systemElement 系统元素
     * @return 影响行数
     */
    int update(SystemElement systemElement);
    
    /**
     * 逻辑删除系统元素
     *
     * @param id ID
     * @param modifier 修改人
     * @return 影响行数
     */
    int deleteById(@Param("id") Long id, @Param("modifier") String modifier);
}
```

**SystemElementMapper.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.fema.mapper.SystemElementMapper">
    
    <resultMap id="BaseResultMap" type="com.example.fema.entity.SystemElement">
        <id column="id" property="id" />
        <result column="code" property="code" />
        <result column="name" property="name" />
        <result column="oldPath" property="oldPath" />
        <result column="dataSource" property="dataSource" />
        <result column="creator" property="creator" />
        <result column="createTime" property="createTime" />
        <result column="modifier" property="modifier" />
        <result column="modifyTime" property="modifyTime" />
        <result column="isDeleted" property="isDeleted" />
    </resultMap>
    
    <sql id="Base_Column_List">
        id, code, name, oldPath, dataSource, creator, createTime, modifier, modifyTime, isDeleted
    </sql>
    
    <select id="selectAll" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM system_element
        WHERE isDeleted = 0
    </select>
    
    <select id="selectById" parameterType="java.lang.Long" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM system_element
        WHERE id = #{id} AND isDeleted = 0
    </select>
    
    <select id="selectByCode" parameterType="java.lang.String" resultMap="BaseResultMap">
        SELECT <include refid="Base_Column_List" />
        FROM system_element
        WHERE code = #{code} AND isDeleted = 0
    </select>
    
    <insert id="insert" parameterType="com.example.fema.entity.SystemElement" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO system_element (
            code, name, oldPath, dataSource, creator, createTime, modifier, modifyTime, isDeleted
        ) VALUES (
            #{code}, #{name}, #{oldPath}, #{dataSource}, #{creator}, NOW(), #{modifier}, NOW(), 0
        )
    </insert>
    
    <update id="update" parameterType="com.example.fema.entity.SystemElement">
        UPDATE system_element
        <set>
            <if test="name != null">name = #{name},</if>
            modifier = #{modifier},
            modifyTime = NOW()
        </set>
        WHERE id = #{id} AND isDeleted = 0
    </update>
    
    <update id="deleteById">
        UPDATE system_element
        SET isDeleted = 1,
            modifier = #{modifier},
            modifyTime = NOW()
        WHERE id = #{id} AND isDeleted = 0
    </update>
</mapper>
```

## 缺少的接口清单

基于以上实现，我们已经满足了您的需求中的所有接口。概括如下：

1. **功能树相关**
   - 功能树查询 ✓
   - 新建功能域 ✓
   - 新建功能 ✓
   - 修改功能/功能域名称 ✓
   - 删除功能/功能域 ✓

2. **故障相关**
   - 故障树查询（包含未分类故障） ✓
   - 新增故障分类 ✓
   - 新建故障 ✓
   - 修改故障名称/故障分类 ✓
   - 删除故障名称/故障分类 ✓

3. **数据迁移**
   - 从旧FEMA表导入数据 ✓

4. **操作记录**
   - 操作记录查询 ✓

5. **其他辅助功能**
   - 系统元素管理 ✓
   - Code生成规则 ✓

系统已经实现了基于您的需求的完整增删改查功能，包括故障树查询（以分类为第一层，故障为第二层）和操作记录查询功能。所有操作都会记录操作日志，所有实体都支持逻辑删除，并且数据迁移功能能够正确处理旧表到新表的数据转换。

这个系统设计遵循了您的要求，使用Spring Boot 2.7.18 + MySQL + MyBatis + XML实现，并提供了详细的代码实现和注释。



---
Powered by [Claude Exporter](https://www.claudexporter.com)