
# RESTful API in Sync & Async Rust

_11 May 2021 · #rust · #diesel · #rocket · #sqlx · #actix-web_

**Table of Contents**

- [Intro](#intro)
- [General](#general)
    - [Project Setup](#project-setup)
    - [Loading Environment Variables w/dotenv](#loading-environment-variables-wdotenv)
    - [Handling Dates & Times w/chrono](#handling-dates--times-wchrono)
    - [Logging w/fern](#logging-wfern)
    - [JSON Serialization w/serde](#json-serialization-wserde)
    - [Domain Modeling](#domain-modeling)
- [Sync Implementation](#sync-implementation)
    - [SQL Schema Migrations w/diesel-cli](#sql-schema-migrations-wdiesel-cli)
    - [Executing SQL Queries w/Diesel](#executing-sql-queries-wdiesel)
        - [Mapping DB Enums to Rust Enums](#mapping-db-enums-to-rust-enums)
        - [Fetching Data](#fetching-data)
        - [Inserting Data](#inserting-data)
        - [Updating Data](#updating-data)
        - [Deleting Data](#deleting-data)
        - [Using a Connection Pool w/r2d2](#using-a-connection-pool-wr2d2)
        - [Refactoring DB Operations Into a Module](#refactoring-db-operations-into-a-module)
    - [HTTP Routing w/Rocket](#http-routing-wrocket)
        - [Routing Basics](#routing-basics)
        - [GET Requests](#get-requests)
        - [POST & PATCH Requests](#post--patch-requests)
        - [DELETE Requests](#delete-requests)
        - [Refactoring API Routes Into a Module](#refactoring-api-routes-into-a-module)
        - [Authentication](#authentication)
- [Async Implementation](#async-implementation)
    - [SQL Schema Migrations w/sqlx-cli](#sql-schema-migrations-wsqlx-cli)
    - [Executing SQL Queries w/sqlx](#executing-sql-queries-wsqlx)
        - [Fetching Data](#fetching-data-1)
        - [Inserting Data](#inserting-data-1)
        - [Updating Data](#updating-data-1)
        - [Deleting Data](#deleting-data-1)
        - [Compile-Time Verification of SQL Queries](#compile-time-verification-of-sql-queries)
        - [Using a Connection Pool w/sqlx](#using-a-connection-pool-wsqlx)
        - [Refactoring DB Operations Into a Module](#refactoring-db-operations-into-a-module-1)
    - [HTTP Routing w/actix-web](#http-routing-wactix-web)
        - [Routing Basics](#routing-basics-1)
        - [GET Requests](#get-requests-1)
        - [POST & PATCH Requests](#post--patch-requests-1)
        - [DELETE Requests](#delete-requests-1)
        - [Refactoring API Routes Into a Module](#refactoring-api-routes-into-a-module-1)
        - [Authentication](#authentication-1)
- [Benchmarks](#benchmarks)
    - [Servers](#servers)
    - [Methodology](#methodology)
    - [Measuring Resource Usage](#measuring-resource-usage)
    - [Results](#results)
        - [Read-Only Workload](#read-only-workload)
        - [Reads + Writes Workload](#reads--writes-workload)
- [Concluding Thoughts](#concluding-thoughts)
    - [Diesel vs sqlx](#diesel-vs-sqlx)
    - [Rocket vs actix-web](#rocket-vs-actix-web)
    - [Sync Rust vs Async Rust](#sync-rust-vs-async-rust)
    - [Rust vs JS](#rust-vs-js)
    - [In Summary](#in-summary)
- [Discuss](#discuss)
- [Notifications](#notifications)
- [Further Reading](#further-reading)



## 介绍

让我们在Rust中为一个假想的看板式项目管理应用实现一个RESTful API服务器。这种应用在现实世界中的一个流行例子是Trello。

![trello board](../../../assets/trello-board.png)

表面上看，看板很简单：有一个看板和卡片。看板代表一个项目。卡片代表任务。卡片在板上的位置代表任务的状态和进度。最简单的看板有三列任务，分别是：排队（做），进行中（做），和完成（做）。

尽管表面上很简单，但看板，以及所有类型的项目管理软件，都是一个字面上的复杂的无底洞。我们可以实现一百万件事情，而在我们完成第一百万件事情之后，还会有一百万件事情。然而，由于我只是想写一篇文章，而不是写一整本书，所以让我们把功能范围保持得很小。

服务器应该支持以下能力：
- 创建看板
- 看板有名称

- 获取所有看板的列表
- 删除看板
- 创建卡片
  - 可以将卡片与板块联系起来
  - 卡片有描述和状态
- 获取一个看板上所有卡片的列表
- 获取看板摘要：看板上所有卡片的数量，按其状态分组
- 更新卡片
- 删除卡片

就这样了! 为了使这个项目更加有趣，我们还为服务器的所有端点加入了基于令牌的认证，但我们要保持简单：只要一个请求包含一个有效的令牌，它就可以访问所有的看板和卡片。

此外，为了满足我自己的好奇心，并使本文更有教育意义，我们将一起写两个实现：一个使用同步Rust，另一个使用async Rust。第一个实现将使用r2d2、Diesel和Rocket。第二个实现将使用sqlx，和actix-web。下面是我们将在这个项目中使用的crates：

通用crates
- dotenv （加载环境变量）
- log + fern （日志）
- chrono （日期和时间处理）
- serde + serde_json （JSON 序列化/反序列化）

同步crates
- diesel-cli （DB schema 迁移）
- diesel + diesel-derive-enum （ORM / 构建SQL queries）
- r2d2 （数据库连接池）
- rocket + rocket_contrib （HTTP路由）

异步crates
- sqlx-cli (数据库模式迁移)
- sqlx (执行SQL查询，数据库连接池)
- actix-web (HTTP 路由)
- futures (future通用模块)

在完成了同步和异步的实现后，我们将运行一些基准测试，看看哪个性能更好，因为每个人都喜欢基准测试。



## 概



### 项目设置

所有关于设置这个项目的无聊说明，比如安装Docker和本地运行，都在[配套代码库](https://github.com/pretzelhammer/kanban)中。在这篇文章中，让我们把注意力完全集中在有趣的部分：Rust!

在初始设置之后，我们有了这个空的`Cargo.toml`文件：

```toml
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"
```

还有这个空的main文件：

```rust
// src/main.rs

fn main() {
    println!("Hello, world!");
}
```


### 用w/dotenv加载环境变量

crates
- dotenv

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

+[dependencies]
+dotenv = "0.15"
```

这个crate做了一个简单的小工作：它从当前工作目录中的`.env`中加载变量，并将它们添加到程序的环境变量中。下面是我们将要使用的通用`.env`文件：
```bash
# .env

LOG_LEVEL=INFO
LOG_FILE=server.log
DATABASE_URL=postgres://postgres@localhost:5432/postgres 
```

加了`dotenv`的main文件：

```rust
// src/main.rs

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    // 从.env加载环境变量
    dotenv::dotenv()?;

    // 样例
    assert_eq!("INFO", std::env::var("LOG_LEVEL").unwrap());

    Ok(())
}
```



### 用w/chrono处理日期和时间

crates
- chrono

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
+chrono = "0.4"
```

Rust处理日期和时间的常用库是Chrono。我们还没有在项目中使用这个依赖，但在增加一些依赖之后很快就会用到。



### 用w/fern实现日志

crates
- log
- fern

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = "0.4"
+log = "0.4"
+fern = "0.6"
```

Log是Rust的logging外观(facade)库。它提供了高层次的日志API，但我们仍然需要选择一个实现，而我们要用的实现是fern crate。我们可以用Fern轻松地定制日志格式，并且还可以链式调用多个输出，这样我们就可以把日志记录到stderr和某个文件。在添加了log和fern之后，让我们把所有的日志配置和初始化都封装到它自己的模块中。

```rust
// src/logger.rs

use std::env;
use std::fs;
use log::{debug, error, info, trace, warn};

pub fn init() -> Result<(), fern::InitError> {
    // 从env拉取日志等级
    let log_level = env::var("LOG_LEVEL").unwrap_or("INFO".into());
    let log_level = log_level
        .parse::<log::LevelFilter>()
        .unwrap_or(log::LevelFilter::Info);

    let mut builder = fern::Dispatch::new()
        .format(|out, message, record| {
            out.finish(format_args!(
                "[{}][{}][{}] {}",
                chrono::Local::now().format("%H:%M:%S"),
                record.target(),
                record.level(),
                message
            ))
        })
        .level(log_level)
        // 向stderr写入日志
        .chain(std::io::stderr());

    // 如果env里有文件，也向文件写入日志
    if let Ok(log_file) = env::var("LOG_FILE") {
        let log_file = fs::File::create(log_file)?;
        builder = builder.chain(log_file);
    }

    // 全局应用logger
    builder.apply()?;

    trace!("TRACE output enabled");
    debug!("DEBUG output enabled");
    info!("INFO output enabled");
    warn!("WARN output enabled");
    error!("ERROR output enabled");

    Ok(())
}
```

然后将该模块添加到我们的main文件中：

```diff
// src/main.rs

+mod logger;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
+   logger::init()?;

    Ok(())
}
```

如果现在运行这个程序，由于`INFO`是默认的日志级别，我们会看到以下输出：

```bash
$ cargo run
[08:36:30][kanban::logger][INFO] INFO output enabled
[08:36:30][kanban::logger][WARN] WARN output enabled
[08:36:30][kanban::logger][ERROR] ERROR output enabled
```


### 用w/serde序列化JSON

crates
- serde
- serde_json

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
- chrono = "0.4"
+ chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
+ serde = { version = "1.0", features = ["derive"] }
+ serde_json = "1.0"
```

建议：在项目中添加新的依赖时，最好检查一下现有的依赖，看看它们是否将新的依赖当作特性标志。在这个例子中，chrono的serde是一个特性标志，如果启用它，会在chrono的所有类型中增加`serde::Serialize`和`serde::Deserialize` 这两个impls。这将允许我们以后在自己的结构中使用chrono类型，我们也将用它们实现`serde::Serialize`和`serde::Deserialize` impls的派生。



### 业务对象建模

好吧，让我们开始为业务对象建模。我们知道会有看板，所以：

```rust
#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Board {
    pub id: i64,
    pub name: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

揭晓这里出现的新语法：
- `#[derive(serde::Serialize)]`为`Board`派生了一个`serde::Serialize`内联，这将允许我们使用`serde_json`将它序列化为JSON。
- `#[serde(rename_all = "camelCase")]`在序列化时，将所有snake_case成员标识符重命名为camelCase（在反序列化时，反之亦然）。这是因为Rust的惯例是用snake_case（下划线命名）命名，但是JSON经常是由JS代码产生和消费的，而JS的惯例是成员标识符用camelCase（驼峰命名）命名。
- `id`是`i64`而不是`u64`似乎很奇怪，但由于用PostgreSQL作数据库，我们只能这样做，因为PostgreSQL只支持有符号的整数类型。
- `created_at`成员总是很有用的，如果没有其他原因的话，可以在没有更好的排序方式时按时间顺序对实体进行排序。

好了，让我们添加卡片和状态：
```rust
#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Card {
    pub id: i64,
    pub board_id: i64,
    pub description: String,
    pub status: Status,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub enum Status {
    Todo,
    Doing,
    Done,
}
```

由于我们也想支持返回 "看板摘要"，其中包含了看板上所有卡片的数量，按其状态分组，这里的模型是这样的：

```rust
#[derive(serde::Serialize)]
pub struct BoardSummary {
    pub todo: i64,
    pub doing: i64,
    pub done: i64,
}
```

当用API创建一个新的看板时，用户可以提供看板名称，但不能提供它的ID，因为它由数据库来设置，所以我们也需要一个模型：

```rust
#[derive(serde::Deserialize)]
pub struct CreateBoard {
    pub name: String,
}
```

同样地，用户也可以创建卡片。当创建卡片时，我们假设只希望用户提供新卡片的描述和它应该与哪个看板相关联。新的卡片将获得默认的todo状态，并获得由数据库设置的ID。

```rust
#[derive(serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CreateCard {
    pub board_id: i64,
    pub description: String,
}
```

当更新一个卡片时，我们假设用户只能更新描述或状态。如果我们允许他们在看板之间移动卡片，那就很奇怪了，这在大多数项目管理软件中是一个很少见的功能：

```rust
#[derive(serde::Deserialize)]
pub struct UpdateCard {
    pub description: String,
    pub status: Status,
}
```

把上面这些都扔进它们自己的模块，我们就得到：

```rust
// src/models.rs

// 处理GET请求

#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Board {
    pub id: i64,
    pub name: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Card {
    pub id: i64,
    pub board_id: i64,
    pub description: String,
    pub status: Status,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub enum Status {
    Todo,
    Doing,
    Done,
}

#[derive(serde::Serialize)]
pub struct BoardSummary {
    pub todo: i64,
    pub doing: i64,
    pub done: i64,
}

// 处理POST请求

#[derive(serde::Deserialize)]
pub struct CreateBoard {
    pub name: String,
}

#[derive(serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CreateCard {
    pub board_id: i64,
    pub description: String,
}

// 处理PATCH请求

#[derive(serde::Deserialize)]
pub struct UpdateCard {
    pub description: String,
    pub status: Status,
}
```

还有更新后的main文件：

```diff
// src/main.rs

mod logger;
+mod models;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```


## 实现Sync



### 用w/diesel-cli实现SQL schema迁移

crates
- diesel-cli

```bash
cargo install diesel_cli
```

如果上面的命令一开始不起作用，很可能是因为我们没有diesel-cli支持的数据库开发库。由于我们只用PostgreSQL，我们可以用这些命令安装开发库：

```bash
# macOS
brew install postgresql

# ubuntu
apt-get install postgresql libpq-dev
```

然后我们可以告知cargo只安装支持PostgreSQL的diesel-cli：

```bash
cargo install diesel_cli --no-default-features --features postgres
```

diesel-cli通过检查`DATABASE_URL`环境变量来确定要连接的数据库，如果当前工作目录中有`.env`文件，它也会从该文件中加载。

假如数据库目前正在运行，并且`DATABASE_URL`环境变量存在，下面是我们要运行的第一条启动项目的diesel-cli命令：

```bash
diesel setup
```

有了这个，diesel-cli就创建了一个`migrations`目录，我们可以在这里生成和编写我们的数据库 schema 迁移。让我们来生成第一个迁移：

```bash
diesel migration generate create_boards
```

这将创建一个新的目录，例如 `migrations/<year>-<month>-<day>-<time>_create_boards`，其中有`up.sql`和`down.sql`，这是我们要写SQL查询语句的地方。查询语句“up”用于创建或修改我们的数据库模式，在本例中是创建一个看板table：

```sql
-- create_boards up.sql
CREATE TABLE IF NOT EXISTS boards (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'utc')
);

-- 给本地开发环境写点测试数据
INSERT INTO boards
(name)
VALUES
('Test board 1'),
('Test board 2'),
('Test board 3');
```

而 查询语句"down "是为了恢复在查询语句“up”对schema的改变，在这种情况下，放弃(drop)创建的看板table：

```sql
-- create_boards down.sql
DROP TABLE IF EXISTS boards;
```

我们还要保存一些卡片：

```bash
diesel migration generate create_cards
```

卡片的up查询语句：

```sql
-- create_cards up.sql
CREATE TYPE STATUS_ENUM AS ENUM ('todo', 'doing', 'done');

CREATE TABLE IF NOT EXISTS cards (
    id BIGSERIAL PRIMARY KEY,
    board_id BIGINT NOT NULL,
    description TEXT NOT NULL,
    status STATUS_ENUM NOT NULL DEFAULT 'todo',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'utc'),
    CONSTRAINT board_fk
        FOREIGN KEY (board_id)
        REFERENCES boards(id)
        ON DELETE CASCADE
);

-- 给本地开发环境写点测试数据
INSERT INTO cards
(board_id, description, status)
VALUES
(1, 'Test card 1', 'todo'),
(1, 'Test card 2', 'doing'),
(1, 'Test card 3', 'done'),
(2, 'Test card 4', 'todo'),
(2, 'Test card 5', 'todo'),
(3, 'Test card 6', 'done'),
(3, 'Test card 7', 'done');
```

还有卡片的down查询语句：

```sql
-- create_cards down.sql
DROP TABLE IF EXISTS cards;
```

在写完我们的迁移后，我们可以用这个命令来运行：

```bash
diesel migration run
```

程序将按照时间顺序执行迁移，并写出一个Diesel schema文件，此时该文件应该是这样的：

```rust
// src/schema.rs

table! {
    boards (id) {
        id -> Int8,
        name -> Text,
        created_at -> Timestamptz,
    }
}

table! {
    cards (id) {
        id -> Int8,
        board_id -> Int8,
        description -> Text,
        status -> Status_enum,
        created_at -> Timestamptz,
    }
}

joinable!(cards -> boards (board_id));

allow_tables_to_appear_in_same_query!(
    boards,
    cards,
);
```

上述文件由diesel-cli生成，我们不应该手动编辑，但是前面的`diesel setup`命令也会生成`diesel.toml`配置文件，如果我们需要配置或修改Diesel模式的生成方式，我们可以编辑`diesel.toml`。

```toml
# diesel.toml

[print_schema]
file = "src/schema.rs"
```

这一点在后面会很有用。



### 用w/Diesel执行SQL查询语句

crates
- diesel

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
+diesel = { version = "1.4", features = ["postgres", "chrono"] }
```

我们启用postgres和chrono功能标志，因为我们将连接到PostgreSQL并将PostgreSQL的时间戳类型反序列化为chrono类型。更新后的main文件：

```diff
// src/main.rs

+#[macro_use]
+extern crate diesel;

mod logger;
mod models;
+mod schema;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```

在用Diesel的派生宏装饰我们的模型之前，我们必须将Diesel模式文件中生成的类型带入作用域：

```diff
// src/models.rs

+use crate::schema::*;

// etc
```

然而，如果我们现在运行`cargo check`，会得到这个结果：

```none
error[E0412]: cannot find type `Status_enum` in this scope
  --> src/schema.rs:14:19
   |
14 |         status -> Status_enum,
   |                   ^^^^^^^^^^^ not found in this scope
```

啊哦，现在怎么办？



#### 映射数据库枚举(Enum)到Rust枚举(Enums)

你可能还记得在上一节中，我们在PostgreSQL中这样定义一个枚举类型：

```sql
CREATE TYPE STATUS_ENUM AS ENUM ('todo', 'doing', 'done');
```

不幸的是，Diesel并不支持直接将数据库枚举映射到Rust枚举，所以我们不得不用一个非官方的第三方库来做这件事：

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
diesel = { version = "1.4", features = ["postgres", "chrono"] }
+diesel-derive-enum = { version = "1.1", features = ["postgres"] }
```

然后我们用下面的派生宏和属性来装饰这个枚举类型：

```diff
-#[derive(serde::Serialize, serde::Deserialize)]
+#[derive(serde::Serialize, serde::Deserialize, diesel_derive_enum::DbEnum)]
#[serde(rename_all = "camelCase")]
+#[DieselType = "Status_enum"]
pub enum Status {
    Todo,
    Doing,
    Done,
}
```

Derive宏在同一作用域内生成一个新的类型`Status_enum`，Diesel可以用这个类型将`STATUS_ENUM`数据库枚举映射到`Status`Rust枚举。

然后我们用它更新`diesel.toml`：

```diff
[print_schema]
file = "src/schema.rs"
+import_types = ["diesel::sql_types::*", "crate::models::Status_enum"]
```

然后我们用这个命令重新生成Diesel schema文件：

```bash
diesel print-schema > ./src/schema.rs
```

Diesel schema文件就会更新成这样：:

```diff
// src/schema.rs

table! {
+    use diesel::sql_types::*;
+    use crate::models::Status_enum;

    boards (id) {
        id -> Int8,
        name -> Text,
        created_at -> Timestamptz,
    }
}

table! {
+    use diesel::sql_types::*;
+    use crate::models::Status_enum;

    cards (id) {
        id -> Int8,
        board_id -> Int8,
        description -> Text,
        status -> Status_enum,
        created_at -> Timestamptz,
    }
}

joinable!(cards -> boards (board_id));

allow_tables_to_appear_in_same_query!(
    boards,
    cards,
);
```

现在当我们运行 "cargo check"时，所有类型的检查都很正常。哇，这个问题相当恼人，不过还好它已经解决了。


#### 获取数据

好了，现在让我们用方便的派生宏，为我们的模型impl一些Diesel特征：

```diff
-#[derive(serde::Serialize)]
+#[derive(serde::Serialize, diesel::Queryable)]
#[serde(rename_all = "camelCase")]
pub struct Board {
    pub id: i64,
    pub name: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

为结构体(struct)派生"diesel::Queryable"，使其可以作为Diesel查询的结果返回。该结构成员的顺序必须与Diesel查询返回的行中的列的顺序一致。

由Diesel生成的模式文件输出了一个DSL，我们可以用它来制作数据库查询语句。使用DSL的主要好处之一是，它可以在编译时验证我们所有的SQL查询在语法和语义上都是正确的。这里有一些加了注释的例子：
```rust
use diesel::prelude::*;
use diesel::PgConnection;

// diesel-cli从数据库schema生成的
use crate::schema::boards;

// 我们手写的
use crate::models::Board;

// 连接到PostgreSQL的样例
fn get_connection() -> PgConnection {
    dotenv::dotenv().unwrap();
    let db_url = env::var("DATABASE_URL").unwrap();
    PgConnection::establish(&db_url).unwrap()
}

// 从Board表中获取所有的Board，
// diesel可以将DB的行映射到Board模型上
// 因为我们为它派生了Queryable，并且
// 数据库的行数据与Board的成员相匹配。
fn all_boards(conn: &PgConnection) -> Vec<Board> {
    boards::table
        .load(conn)
        .unwrap()
}

// 按时间顺序获取所有看板
fn all_boards_chronological(conn: &PgConnection) -> Vec<Board> {
    boards::table
        .order_by(boards::created_at.asc())
        .load(conn)
        .unwrap()
}

// 按id获取一个看板
fn board_by_id(conn: &PgConnection, board_id: i64) -> Board {
    boards::table
        .filter(boards::id.eq(board_id))
        .first(conn)
        .unwrap()
}

// 按名称获取多个看板
fn board_by_name(conn: &PgConnection, board_name: &str) -> Vec<Board> {
    boards::table
        .filter(boards::name.eq(board_name))
        .load(conn)
        .unwrap()
}

// 如果看板名称中包含输入的字符串，则获取看板
fn board_name_contains(conn: &PgConnection, contains: &str) -> Vec<Board> {
    // 在LIKE查询中，"%"表示 "匹配零或更多的任何字符"
    let contains = format!("%{}%", contains);
    boards::table
        .filter(boards::name.ilike(contains))
        .load(conn)
        .unwrap()
}

// 获取过去24小时内创建的看板
fn recent_boards(conn: &PgConnection) -> Vec<Board> {
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    boards::table
        .filter(boards::created_at.ge(past_day))
        .load(conn)
        .unwrap()
}

// 通过链式过滤方法
// 获取过去24小时内创建、
// 且名称包含输入字符串的看板
fn recent_boards_and_name_contains(conn: &PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    boards::table
        .filter(boards::name.ilike(contains))
        .filter(boards::created_at.ge(past_day))
        .load(conn)
        .unwrap()
}

// 通过组合操作符
// 获取过去24小时内创建、
// 且名称中包含某个字符串的看板
fn recent_boards_and_name_contains_2(conn: &PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    let predicate = boards::name.ilike(contains).and(boards::created_at.ge(past_day));
    boards::table
        .filter(predicate)
        .load(conn)
        .unwrap()
}

// 通过链式过滤方法
// 获取过去24小时内创建、
// 或名称包含输入字符串的看板
fn recent_boards_or_name_contains(conn: &PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    boards::table
        .filter(boards::name.ilike(contains))
        .or_filter(boards::created_at.ge(past_day))
        .load(conn)
        .unwrap()
}

// 通过组合操作符
// 获取过去24小时内创建、
// 或名称中包含某个字符串的看板
fn recent_boards_or_name_contains_2(conn: &PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    let predicate = boards::name.ilike(contains).or(boards::created_at.ge(past_day));
    boards::table
        .filter(predicate)
        .load(conn)
        .unwrap()
}
```

OK，让我们也为卡片派生`diesel::Queryable`。

```diff
-#[derive(serde::Serialize)]
+#[derive(serde::Serialize, diesel::Queryable)]
#[serde(rename_all = "camelCase")]
pub struct Card {
    pub id: i64,
    pub board_id: i64,
    pub description: String,
    pub status: Status,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

我们在上面讲了很多查询的例子，但这里还有一些：

```rust
use diesel::prelude::*;
use diesel::PgConnection;

// diesel-cli从数据库schema生成的
use crate::schema::cards;

// 我们手写的
use crate::models::{Card, Status};

// 获取所有卡片
fn all_cards(conn: &PgConnection) -> Vec<Card> {
    cards::table
        .load(conn)
        .unwrap()
}

// 获取看板内的多个卡牌
fn cards_by_board(conn: &PgConnection, board_id: i64) -> Vec<Card> {
    cards::table
        .filter(cards::board_id.eq(board_id))
        .load(conn)
        .unwrap()
}

// 获取属于某一状态的卡牌
fn cards_by_status(conn: &PgConnection, status: Status) -> Vec<Card> {
    cards::table
        .filter(cards::status.eq(status))
        .load(conn)
        .unwrap()
}
```

因此，我们似乎可以用Diesel的DSL来生成服务器需要支持的几乎所有查询，几乎是这样。我们想支持的查询要返回一个 "看板摘要"，也就是按状态分组的所有卡牌的分组计数。这就是我们要写的SQL查询：

```sql
SELECT count(*), status
FROM cards
WHERE cards.board_id = $1
GROUP BY status;
```

不幸的是，我们不能用Diesel生成的DSL来生成这个查询。像所有的ORM一样，Diesel提供了一个直接运行SQL的方法，这个方法就是`diesel::sql_query`，所以让我们讨论一下如何使用这个方法来获得看板摘要：

首先，我们需要创建一个新的模型，它是我们查询的结果，并派生`diesel::QueryableByName`，以及用diesel类型来装饰它的成员。

```rust
#[derive(Default, serde::Serialize)]
pub struct BoardSummary {
    pub todo: i64,
    pub doing: i64,
    pub done: i64,
}

// diesel::sql_query 查询的结果
#[derive(diesel::QueryableByName)]
pub struct StatusCount {
    #[sql_type = "diesel::sql_types::BigInt"]
    pub count: i64,
    #[sql_type = "Status_enum"]
    pub status: Status,
}

// 从StatusCounts转换成BoardSummary
impl From<Vec<StatusCount>> for BoardSummary {
    fn from(counts: Vec<StatusCount>) -> BoardSummary {
        let mut summary = BoardSummary::default();
        for StatusCount { count, status } in counts {
            match status {
                Status::Todo => summary.todo += count,
                Status::Doing => summary.doing += count,
                Status::Done => summary.done += count,
            }
        }
        summary
    }
}
```

为什么作为”常规“Diesel查询结果的结构体必须派生`diesel::Queryable`，而作为`diesel::sql_query`查询结果的结构体必须派生`diesel::QueryableByName`？根据Diesel的文档，前者按索引反序列化DB行列，后者按名称反序列化DB行列，前者性能更好，但后者更不容易出错，因此它被用于任意的SQL查询。

总之，在跳过上面所有的圈套之后，我们终于可以实现函数获取看板摘要了：

```rust
// 通过id获取看板的卡片摘要
fn board_summary(conn: &PgConnection, board_id: i64) -> BoardSummary {
    diesel::sql_query(format!(
        "SELECT count(*), status FROM cards WHERE cards.board_id = {} GROUP BY status",
        board_id
    ))
    .load::<StatusCount>(conn)
    .unwrap()
    .into()
}
```



#### 插入数据

我们不只是想获取看板，还想创建看板！这里提醒一下，`CreateBoard`模型只有一个`name`成员，因为`id`和`created_at`成员由数据设置。这就是我们更新`CreateBoard`模型，将其用于Diesel的DSL：

```diff
// 看板必须在作用域内才能在table_name属性中使用
use crate::schema::boards;

-#[derive(serde::Deserialize)]
+#[derive(serde::Deserialize, diesel::Insertable)]
+#[table_name = "boards"]
pub struct CreateBoard {
    pub name: String,
}
```

为 `CreateBoard`派生了`diesel::Insertable`impl之后，我们可以在Diesel插入查询中直接使用它：

```rust
use diesel::prelude::*;
use diesel::PgConnection;

use crate::schema::boards;
use crate::models::{Board, CreateBoard};

// 从CreateBoard模型中创建并返回新看板
fn create_board(conn: &PgConnection, create_board: CreateBoard) -> Board {
    diesel::insert_into(boards::table)
        .values(&create_board)
        .get_result(conn) // 返回插入看板的结果
        .unwrap()
}

// 同上，但不用CreateBoard模型
fn create_board_2(conn: &PgConnection, board_name: String) -> Board {
    diesel::insert_into(boards::table)
        .values(boards::name.eq(board_name))
        .get_result(conn) // 返回插入看板的结果
        .unwrap()
}
```

我们对 `CreateCard`结构体采取同样的步骤：

```diff
// 看板必须在作用域内才能在table_name属性中使用
use crate::schema::cards;

- #[derive(serde::Deserialize)]
+ #[derive(serde::Deserialize, diesel::Insertable)]
#[serde(rename_all = "camelCase")]
+ #[table_name = "cards"]
pub struct CreateCard {
    pub board_id: i64,
    pub description: String,
}
```

create函数样例：

```rust
use diesel::prelude::*;
use diesel::PgConnection;

use crate::schema::cards;
use crate::models::{Card, CreateCard};

// 使用CreateCard模型创建并返回新卡片
fn create_card(conn: &PgConnection, create_card: CreateCard) -> Card {
    diesel::insert_into(cards::table)
        .values(&create_card)
        .get_result(conn) // 返回插入卡片的结果
        .unwrap()
}

// 同上，但不用CreateCard模型
fn create_card_2(conn: &PgConnection, board_id: i64, description: String) -> Card {
    diesel::insert_into(cards::table)
        .values((cards::board_id.eq(board_id), cards::description.eq(description)))
        .get_result(conn) // 返回插入卡片的结果
        .unwrap()
}
```



#### 更新数据

If we derive `diesel::AsChangeSet` for the `UpdateCard` model:

```diff
// cards has to be in scope to be used in table_name attribute
use crate::schema::cards;

-#[derive(serde::Deserialize)]
+#[derive(serde::Deserialize, diesel::AsChangeset)]
+#[table_name = "cards"]
pub struct UpdateCard {
    pub description: String,
    pub status: Status,
}
```

We can now pass `UpdateCard` directly to the `set` method of Diesel update queries:

```rust
use diesel::prelude::*;
use diesel::PgConnection;

use crate::schema::cards;
use crate::models::{Card, UpdateCard};

// update card in DB using UpdateCard model
fn update_card(conn: &PgConnection, card_id: i64, update_card: UpdateCard) -> Card {
    diesel::update(cards::table.filter(cards::id.eq(card_id)))
        .set(update_card)
        .get_result(conn) // return updated card result
        .unwrap()
}

// same as above except without using model
fn update_card_2(conn: &PgConnection, card_id: i64, description: String, status: Status) -> Card {
    diesel::update(cards::table.filter(cards::id.eq(card_id)))
        .set((cards::description.eq(description), cards::status.eq(status)))
        .get_result(conn) // return updated card result
        .unwrap()
}
```



#### Deleting Data

Some delete query examples:

```rust
use diesel::prelude::*;
use diesel::PgConnection;
use crate::schema::{boards, cards};

// delete all boards
fn delete_all_boards(conn: &PgConnection) {
    diesel::delete(boards::table)
        .execute(conn)
        .unwrap();
}

// delete a board by its id
fn delete_board_by_id(conn: &PgConnection, board_id: i64) {
    diesel::delete(boards::table.filter(boards::id.eq(board_id)))
        .execute(conn)
        .unwrap();
}

// delete all cards
fn delete_all_cards(conn: &PgConnection) {
    diesel::delete(cards::table)
        .execute(conn)
        .unwrap();
}

// delete a card by its id
fn delete_card_by_id(conn: &PgConnection, card_id: i64) {
    diesel::delete(cards::table.filter(cards::id.eq(card_id)))
        .execute(conn)
        .unwrap();
}

// delete all the cards on a board
fn delete_cards_by_board(conn: &PgConnection, board_id: i64) {
    diesel::delete(cards::table.filter(cards::board_id.eq(board_id)))
        .execute(conn)
        .unwrap();
}

// delete all done cards on a board
fn delete_done_cards_by_board(conn: &PgConnection, board_id: i64) {
    diesel::delete(
        cards::table
            .filter(cards::board_id.eq(board_id))
            .filter(cards::status.eq(Status::Done)),
    )
    .execute(conn)
    .unwrap();
}
```



#### Using a Connection Pool w/r2d2

crates
- r2d2

Creating DB connections is expensive, so let's use a connection pool library to handle the hard work of managing and reusing DB connections for us. The library we're going to use is r2d2, which comes as a standalone crate but can also can come packaged within Diesel if we enable the r2d2 feature flag, so let's do that:

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
-diesel = { version = "1.4", features = ["postgres", "chrono"] }
+diesel = { version = "1.4", features = ["postgres", "chrono", "r2d2"] }
diesel-derive-enum = { version = "1.1", features = ["postgres"] }
```

Here's how we'd create a pool with the default configuration options:

```rust
use diesel::prelude::*;
use diesel::PgConnection;
use diesel::r2d2;

type PgPool = r2d2::Pool<r2d2::ConnectionManager<PgConnection>>;

fn get_pool() -> PgPool {
    dotenv::dotenv().unwrap();
    let db_url = env::var("DATABASE_URL").unwrap();
    let manager = r2d2::ConnectionManager::new(db_url);
    let pool = r2d2::Pool::new(manager).unwrap();
    pool
}

fn use_pool(pool: &PgPool) {
    let connection = pool.get().unwrap();
    // use connection
}
```



#### Refactoring DB Operations Into a Module

The rest of our application doesn't need to know or care that we're using Diesel and r2d2, so let's abstract all of those implementation details away within a `Db` struct, which we'll put into a `db` module:

```rust
// src/db.rs

use diesel::prelude::*;
use diesel::PgConnection;
use diesel::r2d2;

use crate::StdErr;
use crate::models::*;
use crate::schema::*;

type PgPool = r2d2::Pool<r2d2::ConnectionManager<PgConnection>>;

pub struct Db {
    pool: PgPool,
}

impl Db {
    pub fn connect() -> Result<Self, StdErr> {
        let db_url = env::var("DATABASE_URL")?;
        let manager = r2d2::ConnectionManager::new(db_url);
        let pool = r2d2::Pool::new(manager)?;
        Ok(Db { pool })
    }

    pub fn boards(&self) -> Result<Vec<Board>, StdErr> {
        let conn = self.pool.get()?;
        Ok(boards::table.load(&conn)?)
    }

    pub fn board_summary(&self, board_id: i64) -> Result<BoardSummary, StdErr> {
        let conn = self.pool.get()?;
        let counts: Vec<StatusCount> = diesel::sql_query(format!(
            "select count(*), status from cards where cards.board_id = {} group by status",
            board_id
        ))
        .load(&conn)?;
        Ok(counts.into())
    }

    pub fn create_board(&self, create_board: CreateBoard) -> Result<Board, StdErr> {
        let conn = self.pool.get()?;
        let board = diesel::insert_into(boards::table)
            .values(&create_board)
            .get_result(&conn)?;
        Ok(board)
    }

    pub fn delete_board(&self, board_id: i64) -> Result<(), StdErr> {
        let conn = self.pool.get()?;
        diesel::delete(boards::table.filter(boards::id.eq(board_id))).execute(&conn)?;
        Ok(())
    }

    pub fn cards(&self, board_id: i64) -> Result<Vec<Card>, StdErr> {
        let conn = self.pool.get()?;
        let cards = cards::table
            .filter(cards::board_id.eq(board_id))
            .load(&conn)?;
        Ok(cards)
    }

    pub fn create_card(&self, create_card: CreateCard) -> Result<Card, StdErr> {
        let conn = self.pool.get()?;
        let card = diesel::insert_into(cards::table)
            .values(create_card)
            .get_result(&conn)?;
        Ok(card)
    }

    pub fn update_card(&self, card_id: i64, update_card: UpdateCard) -> Result<Card, StdErr> {
        let conn = self.pool.get()?;
        let card = diesel::update(cards::table.filter(cards::id.eq(card_id)))
            .set(update_card)
            .get_result(&conn)?;
        Ok(card)
    }

    pub fn delete_card(&self, card_id: i64) -> Result<(), StdErr> {
        let conn = self.pool.get()?;
        diesel::delete(
            cards::table.filter(cards::id.eq(card_id)),
        )
        .execute(&conn)?;
        Ok(())
    }
}
```

Updated main file:

```diff
// src/main.rs

#[macro_use]
extern crate diesel;

+mod db;
mod logger;
mod models;
mod schema;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```


### HTTP Routing w/Rocket

crates
- rocket
- rocket_contrib

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
diesel = { version = "1.4", features = ["postgres", "chrono", "r2d2"] }
diesel-derive-enum = { version = "1.1", features = ["postgres"] }
+rocket = "0.4"
+rocket_contrib = "0.4"
```

Rocket v0.4 depends on some nightly features, some of which are only available on compiler version 1.53.0 since they're later removed, so we'll need to install a specific nightly compiler and then switch to using these commands

```bash
# install rustc 1.53.0-nightly (07e0e2ec2 2021-03-24)
rustup install nightly-2021-03-24

# set it as default
rustup default nightly-2021-03-24
```

We also have to add some compiler feature flags to the top of our main:

```diff
// src/main.rs

+// required for rocket macros to work
+#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use]
extern crate diesel;

mod db;
mod logger;
mod models;
mod schema;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```

While we're here let's throw together a quick Rocket hello world:

```diff
// required for rocket macros to work
#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use]
extern crate diesel;

mod db;
mod logger;
mod models;
mod routes;
mod schema;

type StdErr = Box<dyn std::error::Error>;

+#[rocket::get("/")]
+fn hello_world() -> &'static str {
+   "Hello, world!"
+}

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

+   rocket::ignite()
+       .mount("/", rocket::routes![hello_world])
+       .launch();

    Ok(())
}
```

The code above does exactly what you think it does. Also, if we `cargo run` it:

```bash
$ cargo run
[13:07:21][launch][INFO] Configured for development.
[13:07:21][launch_][INFO] address: localhost
[13:07:21][launch_][INFO] port: 8000
[13:07:21][launch_][INFO] log: normal
[13:07:21][launch_][INFO] workers: 32
[13:07:21][launch_][INFO] secret key: generated
[13:07:21][launch_][INFO] limits: forms = 32KiB
[13:07:21][launch_][INFO] keep-alive: 5s
[13:07:21][launch_][INFO] read timeout: 5s
[13:07:21][launch_][INFO] write timeout: 5s
[13:07:21][launch_][INFO] tls: disabled
[13:07:21][rocket::rocket][INFO] Mounting /:
[13:07:21][_][INFO] GET / (hello_world)
[13:07:21][launch][INFO] Rocket has launched from http://localhost:8000
```

Look at all those beautiful logs! I bet you were wondering if, and when, all the logging setup work we did forever ago was going to become useful. Well that time is now! Of course, being good developers who care about being able to debug issues when they arise, we should be logging way more in our application, but I've been intentionally omitting the logging statements because they're noisy and have little educational value. Please imagine that they are there.



#### Routing Basics

We can decorate functions with Rocket's procedural macros to turn them into request handlers, example:

```rust
#[rocket::get("/")]
fn index() -> &'static str {
    "I'm the index route!"
}

#[rocket::get("/nested/route")]
fn index() -> &'static str {
    "I'm some nested route!"
}
```

Data can be extracted from the path using `<param>` placeholders:

```rust
#[rocket::get("/hello/<name>")]
fn greet(name: String) -> String {
    format!("Hello, {}!", name)
}
```

Any type which impls `rocket::request::FromParam` can be extracted from the path:

```rust
use rocket::request::FromParam;

// Rocket provides FromParam impls for
// - all stdlib number types
// - bool
// - String
// - &str
// - some other misc types
#[rocket::get("/echo/<string>/<num>/<maybe>/etc")]
fn echo_path(string: String, num: usize, maybe: bool) -> String {
    format!("got string {}, num {}, and maybe {}", string, num, maybe)
}

// Rocket also provides FromParam impls for
// - Option<T> where T: FromParam
//     - returns Some(T) on success, None otherwise
// - Result<T> where T: FromParam
//     - returns Ok(T) on success, Err(<T as FromParam>::Error) otherwise

// example custom type
struct EvenNumber(i32);

// example FromParam impl
impl<'a> FromParam<'a> for EvenNumber {
    type Error = &'static str;

    fn from_param(param: &'a RawStr) -> Result<Self, Self::Error> {
        let result = i32::from_param(param);
        if let Ok(num) = result {
            if num % 2 == 0 {
                Ok(EvenNumber(num))
            } else {
                Err("param not an even number")
            }
        } else {
            Err("param not a number")
        }
    }
}

// extracting our own custom defined type
#[rocket::get("/<even>")]
fn even_param(even: EvenNumber) -> String {
    format!("got even number {}", even.0)
}
```

A request handler's return type can be anything that impls `rocket::response::Responder`:

```rust
use std::io;
use std::fs::File;
use rocket::{http, request, response};
use rocket::response::{Response, Responder};

// Rocket provides Responder impls for
// - &str
// - String
// - &[u8]
// - Vec<u8>
// - File
// - ()
// - some other misc types
#[rocket::get("/cargo")]
fn returns_cargo() -> File {
    File::open("Cargo.toml").unwrap()
}

// Rocket also provides Responder impls for
// - Option<T> where T: Responder
//     - returns T if Some(T), 404 Not Found if None
// - Result<T, E> where T: Responder, E: Debug
//     - returns T if Ok(T), 500 Internal Server Error if Err(E)

// example custom type
struct EvenNumber(i32);

// example Responder impl
impl<'r> Responder<'r> for EvenNumber {
    fn respond_to(self, _req: &request::Request<'_>) -> response::Result<'r> {
        Response::build()
            .status(http::Status::Ok)
            .raw_header("X-Number-Parity", "Even")
            .sized_body(io::Cursor::new(format!("returning even number {}", self.0)))
            .ok()
    }
}

// returning custom defined type
#[rocket::get("/even")]
fn returns_even() -> EvenNumber {
    EvenNumber(2)
}
```



#### GET Requests

The first problem we need to solve is how to pass our `Db` to our Rocket request handlers. We can achieve this by first passing the `Db` instance to the Rocket application builder via the `manage` method which tells Rocket this is some application state that it should manage for us:

```diff
// src/main.rs

// required for rocket macros to work
#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use]
extern crate diesel;

mod db;
mod logger;
mod models;
mod schema;
mod routes;

type StdErr = Box<dyn std::error::Error>;

#[rocket::get("/")]
fn hello_world() -> &'static str {
    "Hello, world!"
}

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

+   let db = db::Db::connect()?;

    rocket::ignite()
+       .manage(db)
        .mount("/", rocket::routes![hello_world])
        .launch();

    Ok(())
}
```

Then we can retrieve the managed application state from Rocket in our request handlers by wrapping the data type we wish to receive with `rocket::State` as one of our function parameters:

```rust
use rocket::State;
use crate::db::Db;

#[rocket::get("/use/db")]
fn use_db(db: State<Db>) {
    // use db
}
```

We can automatically return JSON by wrapping the return type with the `rocket_contrib::json::Json` type which impls `Responder` and will serialize the wrapped type to JSON using serde and set the response header `Content-Type: application/json`, example:

```rust
use rocket_contrib::json::Json;
use crate::models::Status;

#[rocket::get("/example/json")]
fn return_json() -> Json<Status> {
    Json(Status::Todo)
}
```

Okay, we now have enough context to write all of our GET request handlers:

```rust
use rocket::State;
use rocket_contrib::json::Json;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, Card, BoardSummary};

// GET requests

#[rocket::get("/boards")]
fn boards(db: State<Db>) -> Result<Json<Vec<Board>>, StdErr> {
    db.boards().map(Json)
}

#[rocket::get("/boards/<board_id>/summary")]
fn board_summary(db: State<Db>, board_id: i64) -> Result<Json<BoardSummary>, StdErr> {
    db.board_summary(board_id).map(Json)
}

#[rocket::get("/boards/<board_id>/cards")]
fn cards(db: State<Db>, board_id: i64) -> Result<Json<Vec<Card>>, StdErr> {
    db.cards(board_id).map(Json)
}
```



#### POST & PATCH Requests

`rocket_contrib::json::Json` not only impls `Responder` but also impls `rocket::data::FromData` which means we can use it as a function parameter to a request handler and Rocket will attempt deserialize the request's JSON body into the type we specify, so here's how we'd write the POST & PATCH request handlers:

```rust
use rocket::State;
use rocket_contrib::json::Json;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, CreateBoard, Card, CreateCard, UpdateCard};

// POST requests

#[rocket::post("/boards", data = "<create_board>")]
fn create_board(db: State<Db>, create_board: Json<CreateBoard>) -> Result<Json<Board>, StdErr> {
    db.create_board(create_board.0).map(Json)
}

#[rocket::post("/cards", data = "<create_card>")]
fn create_card(db: State<Db>, create_card: Json<CreateCard>) -> Result<Json<Card>, StdErr> {
    db.create_card(create_card.0).map(Json)
}

// PATCH requests

#[rocket::patch("/cards/<card_id>", data = "<update_card>")]
fn update_card(
    db: State<Db>,
    card_id: i64,
    update_card: Json<UpdateCard>,
) -> Result<Json<Card>, StdErr> {
    db.update_card(card_id, update_card.0).map(Json)
}
```



#### DELETE Requests

The DELETE request handlers:

```rust
use rocket::State;
use rocket_contrib::json::Json;

use crate::StdErr;
use crate::db::Db;

#[rocket::delete("/boards/<board_id>")]
fn delete_board(db: State<Db>, board_id: i64) -> Result<(), StdErr> {
    db.delete_board(board_id)
}

#[rocket::delete("/cards/<card_id>")]
fn delete_card(db: State<Db>, card_id: i64) -> Result<(), StdErr> {
    db.delete_card(card_id)
}
```



#### Refactoring API Routes Into A Module

Let's clean up our code and put all of the API routes into their own module:

```rust
// src/routes.rs

use rocket::http;
use rocket::response;
use rocket::request
use rocket::State;
use rocket_contrib::json::Json;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, CreateBoard, BoardSummary, Card, CreateCard, UpdateCard};

// board routes

#[rocket::get("/boards")]
fn boards(db: State<Db>) -> Result<Json<Vec<Board>>, StdErr> {
    db.boards().map(Json)
}

#[rocket::post("/boards", data = "<create_board>")]
fn create_board(db: State<Db>, create_board: Json<CreateBoard>) -> Result<Json<Board>, StdErr> {
    db.create_board(create_board.0).map(Json)
}

#[rocket::get("/boards/<board_id>/summary")]
fn board_summary(db: State<Db>, board_id: i64) -> Result<Json<BoardSummary>, StdErr> {
    db.board_summary(board_id).map(Json)
}

#[rocket::delete("/boards/<board_id>")]
fn delete_board(db: State<Db>, board_id: i64) -> Result<(), StdErr> {
    db.delete_board(board_id)
}

// card routes

#[rocket::get("/boards/<board_id>/cards")]
fn cards(db: State<Db>, board_id: i64) -> Result<Json<Vec<Card>>, StdErr> {
    db.cards(board_id).map(Json)
}

#[rocket::post("/cards", data = "<create_card>")]
fn create_card(db: State<Db>, create_card: Json<CreateCard>) -> Result<Json<Card>, StdErr> {
    db.create_card(create_card.0).map(Json)
}

#[rocket::patch("/cards/<card_id>", data = "<update_card>")]
fn update_card(
    db: State<Db>,
    card_id: i64,
    update_card: Json<UpdateCard>,
) -> Result<Json<Card>, StdErr> {
    db.update_card(card_id, update_card.0).map(Json)
}

#[rocket::delete("/cards/<card_id>")]
fn delete_card(db: State<Db>, card_id: i64) -> Result<(), StdErr> {
    db.delete_card(card_id)
}

// single public function which returns all API routes

pub fn api() -> Vec<rocket::Route> {
    rocket::routes![
        boards,
        create_board,
        board_summary,
        delete_board,
        cards,
        create_card,
        update_card,
        delete_card,
    ]
}
```

Updated main file:

```diff
// src/main.rs

// required for rocket macros to work
#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use]
extern crate diesel;

mod db;
mod logger;
mod models;
+mod routes;
mod schema;

type StdErr = Box<dyn std::error::Error>;

#[rocket::get("/")]
fn hello_world() -> &'static str {
    "Hello, world!"
}

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    let db = db::Db::connect()?;

    rocket::ignite()
        .manage(db)
        .mount("/", rocket::routes![hello_world])
+       .mount("/api", routes::api())
        .launch();

    Ok(())
}
```

We're almost there! Only one small piece of the puzzle is missing: authentication.



#### Authentication

Let's implement token-based authentication for our RESTful API server. When a request comes in, we'll first check if it has a `Authorization: Bearer <token>` header, and if not reject it immediately. Otherwise, we validate the `<token>` by checking that it exists in a `tokens` DB table, and if it's valid we respond to the request.

Since this is a toy project we're not going to implement different access permissions per token, all tokens will have access to all boards and all cards. Also, let's skip the implementing the handler which generates and returns tokens because it's mostly boilerplate and not relevant to us implementing authentication.

Anyway, we're gonna need to create a `tokens` table first:

```bash
diesel migration generate create_tokens
```

SQL query to create the `tokens` table:

```sql
-- create_tokens up.sql
CREATE TABLE IF NOT EXISTS tokens (
    id TEXT PRIMARY KEY,
    expired_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- seed db with some test data for local dev
INSERT INTO tokens
(id, expired_at)
VALUES
('LET_ME_IN', (CURRENT_TIMESTAMP + INTERVAL '15 minutes') AT TIME ZONE 'utc');
```

SQL query to drop the `tokens` table:

```sql
-- create_tokens down.sql
DROP TABLE IF EXISTS tokens;
```

Then we run the new migration with:

```bash
diesel migration run
```

The above command also has the side-effect of updating the generated Diesel schema file with the new tokens table metadata:

```diff
// src/schema.rs

table! {
    use diesel::sql_types::*;
    use crate::models::Status_enum;

    boards (id) {
        id -> Int8,
        name -> Text,
        created_at -> Timestamptz,
    }
}

table! {
    use diesel::sql_types::*;
    use crate::models::Status_enum;

    cards (id) {
        id -> Int8,
        board_id -> Int8,
        description -> Text,
        status -> Status_enum,
        created_at -> Timestamptz,
    }
}

+table! {
+   use diesel::sql_types::*;
+   use crate::models::Status_enum;
+
+   tokens (id) {
+       id -> Text,
+       expired_at -> Timestamptz,
+   }
+}

joinable!(cards -> boards (board_id));

allow_tables_to_appear_in_same_query!(
    boards,
    cards,
+   tokens,
);
```

Then we derive a `diesel::Queryable` impl for the `Token` model:

```diff
// src/models.rs

use crate::schema::*;

+// for authentication
+
+#[derive(diesel::Queryable)]
+pub struct Token {
+    pub id: String,
+    pub expired_at: chrono::DateTime<chrono::Utc>,
+}

// other models
```

And then add the following method to our `Db` struct to validate tokens:

```diff
use std::env;

use diesel::prelude::*;
use diesel::r2d2;
use diesel::PgConnection;

use crate::models::*;
use crate::schema::*;
use crate::StdErr;

type PgPool = r2d2::Pool<r2d2::ConnectionManager<PgConnection>>;

pub struct Db {
    pool: PgPool,
}

impl Db {
    pub fn connect() -> Result<Self, StdErr> {
        let db_url = env::var("DATABASE_URL")?;
        let manager = r2d2::ConnectionManager::new(db_url);
        let pool = r2d2::Pool::new(manager)?;
        Ok(Db { pool })
    }

+   // token methods
+
+   pub fn validate_token(&self, token_id: &str) -> Result<Token, StdErr> {
+       let conn = self.pool.get()?;
+       let token = tokens::table
+           .filter(tokens::id.eq(token_id))
+           .filter(tokens::expired_at.ge(diesel::dsl::now))
+           .first(&conn)?;
+       Ok(token)
+   }

    // other methods
}
```

We can implement authentication in Rocket in a couple ways, by using either Rocket middleware or Rocket request guards. The official Rocket docs recommend the latter so let's go with that.

A request guard is any type within a handler's parameters which impls `rocket::request::FromRequest`. So all we have to do is just impl that for `Token` and then add `Token` parameters to all our request handlers and we're golden. Here's the `FromRequest` impl on `Token`:

```rust
use rocket::request::{FromRequest, Request, Outcome};
use crate::models::Token;

impl<'a, 'r> FromRequest<'a, 'r> for Token {
    type Error = &'static str;
    fn from_request(request: &'a Request<'r>) -> Outcome<Self, Self::Error> {
        // get request headers
        let headers = request.headers();

        // check that Authorization header exists
        let maybe_auth_header = headers.get_one("Authorization");
        if maybe_auth_header.is_none() {
            return Outcome::Failure((
                http::Status::Unauthorized,
                "missing Authorization header",
            ));
        }

        // and is well-formed
        let auth_header = maybe_auth_header.unwrap();
        let mut auth_header_parts = auth_header.split_ascii_whitespace();
        let maybe_auth_type = auth_header_parts.next();
        if maybe_auth_type.is_none() {
            return Outcome::Failure((
                http::Status::Unauthorized,
                "malformed Authorization header",
            ));
        }

        // and uses the Bearer token authorization method
        let auth_type = maybe_auth_type.unwrap();
        if auth_type != "Bearer" {
            return Outcome::Failure((
                http::Status::BadRequest,
                "invalid Authorization type",
            ));
        }

        // and the Bearer token is present
        let maybe_token_id = auth_header_parts.next();
        if maybe_token_id.is_none() {
            return Outcome::Failure((http::Status::Unauthorized, "missing Bearer token"));
        }
        let token_id = maybe_token_id.unwrap();

        // we can use request.guard::<T>() to get a T from a request
        // which includes managed application state like our Db
        let outcome_db = request.guard::<State<Db>>();
        let db: State<Db> = match outcome_db {
            Outcome::Success(db) => db,
            _ => return Outcome::Failure((http::Status::InternalServerError, "internal error")),
        };

        // validate token
        let token_result = db.validate_token(token_id);
        match token_result {
            Ok(token) => Outcome::Success(token),
            Err(_) => Outcome::Failure((
                http::Status::Unauthorized,
                "invalid or expired Bearer token",
            )),
        }
    }
}
```

Now we add a `Token` parameter to every request handler we want to guard, which is effectively how we implement authentication:

```diff
// src/routes.rs

use rocket::http;
use rocket::State;
use rocket::request::{FromRequest, Request, Outcome};
use rocket_contrib::json::Json;

use crate::db::Db;
use crate::models::*;
use crate::StdErr;

impl<'a, 'r> FromRequest<'a, 'r> for Token {
    type Error = &'static str;
    fn from_request(request: &'a Request<'r>) -> Outcome<Self, Self::Error> {
        // impl
    }
}

// board routes

#[rocket::get("/boards")]
-fn boards(db: State<Db>) -> Result<Json<Vec<Board>>, StdErr> {
+fn boards(db: State<Db>, _t: Token) -> Result<Json<Vec<Board>>, StdErr> {
    db.boards().map(Json)
}

#[rocket::post("/boards", data = "<create_board>")]
-fn create_board(db: State<Db>, create_board: Json<CreateBoard>) -> Result<Json<Board>, StdErr> {
+fn create_board(db: State<Db>, create_board: Json<CreateBoard>, _t: Token) -> Result<Json<Board>, StdErr> {
    db.create_board(create_board.0).map(Json)
}

#[rocket::get("/boards/<board_id>/summary")]
-fn board_summary(db: State<Db>, board_id: i64) -> Result<Json<BoardSummary>, StdErr> {
+fn board_summary(db: State<Db>, board_id: i64, _t: Token) -> Result<Json<BoardSummary>, StdErr> {
    db.board_summary(board_id).map(Json)
}

#[rocket::delete("/boards/<board_id>")]
- fn delete_board(db: State<Db>, board_id: i64) -> Result<(), StdErr> {
+ fn delete_board(db: State<Db>, board_id: i64, _t: Token) -> Result<(), StdErr> {
    db.delete_board(board_id)
}

// card routes

#[rocket::get("/boards/<board_id>/cards")]
-fn cards(db: State<Db>, board_id: i64) -> Result<Json<Vec<Card>>, StdErr> {
+fn cards(db: State<Db>, board_id: i64, _t: Token) -> Result<Json<Vec<Card>>, StdErr> {
    db.cards(board_id).map(Json)
}

#[rocket::post("/cards", data = "<create_card>")]
-fn create_card(db: State<Db>, create_card: Json<CreateCard>) -> Result<Json<Card>, StdErr> {
+fn create_card(db: State<Db>, create_card: Json<CreateCard>, _t: Token) -> Result<Json<Card>, StdErr> {
    db.create_card(create_card.0).map(Json)
}

#[rocket::patch("/cards/<card_id>", data = "<update_card>")]
fn update_card(
    db: State<Db>,
    card_id: i64,
    update_card: Json<UpdateCard>,
+   _t: Token,
) -> Result<Json<Card>, StdErr> {
    db.update_card(card_id, update_card.0).map(Json)
}

#[rocket::delete("/cards/<card_id>")]
-fn delete_card(db: State<Db>, card_id: i64) -> Result<(), StdErr> {
+fn delete_card(db: State<Db>, card_id: i64, _t: Token) -> Result<(), StdErr> {
    db.delete_card(card_id)
}

pub fn api() -> Vec<rocket::Route> {
    rocket::routes![
        boards,
        create_board,
        board_summary,
        delete_board,
        cards,
        create_card,
        update_card,
        delete_card,
    ]
}
```

Amazing. We did it. The full source code for the Diesel + Rocket implementation can be found in the [companion code repository](https://github.com/pretzelhammer/kanban/tree/main/diesel-rocket) for this article. Okay, now let's do it again, except this time we'll implement it in async Rust using sqlx and actix-web.



## Async Implementation

This section picks up from the same place as the **Sync Implementation** section does: which is right after we added the `serde` & `serde_json` crates to our project's dependencies and created a `models` module.



### SQL Schema Migrations w/sqlx-cli

crates
- sqlx-cli

```bash
cargo install sqlx-cli
```

Again if this fails it's likely because we're missing some development libraries on our system, we can solve this issue with the following commands:

```bash
# macOS
brew install postgresql

# ubuntu
apt-get install pkg-config libssl-dev postgresql libpq-dev
```

And then we can install sqlx-cli with only support for PostgreSQL:

```bash
cargo install sqlx-cli --no-default-features --features postgres
```

As before, let's create a boards and cards tables:

```bash
sqlx migrate add create_boards
sqlx migrate add create_cards
```

This creates a `migrations` directory with a couple migration files. Here's the migration file to create the boards table:

```sql
-- create_boards.sql
CREATE TABLE IF NOT EXISTS boards (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'utc')
);

-- seed db with some test data for local dev
INSERT INTO boards
(name)
VALUES
('Test board 1'),
('Test board 2'),
('Test board 3');
```

And here's the migration file to create the cards table:

```sql
-- create_cards.sql
CREATE TYPE STATUS AS ENUM ('todo', 'doing', 'done');

CREATE TABLE IF NOT EXISTS cards (
    id BIGSERIAL PRIMARY KEY,
    board_id BIGINT NOT NULL,
    description TEXT NOT NULL,
    status STATUS NOT NULL DEFAULT 'todo',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'utc'),
    CONSTRAINT board_fk
        FOREIGN KEY (board_id)
        REFERENCES boards(id)
        ON DELETE CASCADE
);

-- seed db with some test data for local dev
INSERT INTO cards
(board_id, description, status)
VALUES
(1, 'Test card 1', 'todo'),
(1, 'Test card 2', 'doing'),
(1, 'Test card 3', 'done'),
(2, 'Test card 4', 'todo'),
(2, 'Test card 5', 'todo'),
(3, 'Test card 6', 'done'),
(3, 'Test card 7', 'done');
```

We run the migrations with:

```bash
sqlx migrate run
```



### Executing SQL Queries w/sqlx

crates
- sqlx

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
+sqlx = { version = "0.4", features = ["runtime-actix-rustls", "chrono", "postgres"] }
```

Since sqlx is an async library that produces futures those futures have to be executed by a runtime, and actix-web provides a runtime which we're gonna add later, but this is the reason why we added the `runtime-actix-rustls` feature flag.



#### Fetching Data

We can impl the `sqlx::FromRow` trait for our `Board` model using a derive macro:

```diff
// src/models.rs

-#[derive(serde::Serialize)]
+#[derive(serde::Serialize, sqlx::FromRow)]
#[serde(rename_all = "camelCase")]
pub struct Board {
    pub id: i64,
    pub name: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

This impl allows us to use `Board` as the returned result of SQL queries. Let's look at some examples:

```rust
use sqlx::{Connection, PgConnection};
use crate::models::Board;

// example of connecting to PostgreSQL
async fn get_connection() -> PgConnection {
    dotenv::dotenv().unwrap();
    let db_url = std::env::var("DATABASE_URL").unwrap();
    PgConnection::connect(&db_url).await.unwrap()
}

// fetch all boards
async fn all_boards(conn: &mut PgConnection) -> Vec<Board> {
    sqlx::query_as("SELECT * FROM boards")
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all boards in order
async fn all_boards_chronological(conn: &mut PgConnection) -> Vec<Board> {
    sqlx::query_as("SELECT * FROM boards ORDER BY created_at ASC")
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch board by primary key
async fn board_by_id(conn: &mut PgConnection, board_id: i64) -> Board {
    sqlx::query_as("SELECT * FROM boards WHERE id = $1")
        .bind(board_id)
        .fetch_one(conn)
        .await
        .unwrap()
}

// fetch all boards with a specific exact name
async fn board_by_name(conn: &mut PgConnection, board_name: &str) -> Vec<Board> {
    sqlx::query_as("SELECT * FROM boards WHERE name = $1")
        .bind(board_name)
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all board whose name contains some string
async fn board_name_contains(conn: &mut PgConnection, contains: &str) -> Vec<Board> {
    // in LIKE queries "%" means "match zero or more of any character"
    let contains = format!("%{}%", contains);
    sqlx::query_as("SELECT * FROM boards WHERE name ILIKE $1")
        .bind(contains)
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all boards created in the past 24 hours
async fn recent_boards(conn: &mut PgConnection) -> Vec<Board> {
    sqlx::query_as("SELECT * FROM boards WHERE created_at >= CURRENT_TIMESTAMP - INTERVAL '1 day'")
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all boards created in the past 24 hours and whose name also contains some string
async fn recent_boards_and_name_contains(conn: &mut PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    sqlx::query_as("SELECT * FROM boards WHERE created_at >= CURRENT_TIMESTAMP - INTERVAL '1 day' AND name ILIKE $1")
        .bind(contains)
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all boards created in the past 24 hours or whose name contains some string
async fn recent_boards_or_name_contains(conn: &mut PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    sqlx::query_as("SELECT * FROM boards WHERE created_at >= CURRENT_TIMESTAMP - INTERVAL '1 day' OR name ILIKE $1")
        .bind(contains)
        .fetch_all(conn)
        .await
        .unwrap()
}
```

Okay, let's look at fetching cards now:

```diff
// src/models.rs

-#[derive(serde::Serialize)]
+#[derive(serde::Serialize, sqlx::FromRow)]
#[serde(rename_all = "camelCase")]
pub struct Card {
    pub id: i64,
    pub board_id: i64,
    pub description: String,
    pub status: Status,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

-#[derive(serde::Deserialize, serde::Serialize)]
+#[derive(serde::Deserialize, serde::Serialize, sqlx::Type)]
#[serde(rename_all = "camelCase")]
+#[sqlx(rename_all = "camelCase")]
pub enum Status {
    Todo,
    Doing,
    Done,
}
```

We can derive `sqlx::Type` for a Rust enum to make it map to a DB enum of the same name. In this case we're mapping the `Status` Rust enum to the `STATUS` DB enum type we defined earlier:

```sql
CREATE TYPE STATUS AS ENUM ('todo', 'doing', 'done');
```

Now let's look at some card query examples:

```rust
use sqlx::{Connection, PgConnection};
use crate::models::{Card, Status};

// fetch all cards
async fn all_cards(conn: &mut PgConnection) -> Vec<Card> {
    sqlx::query_as("SELECT * FROM cards")
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch cards by board
async fn cards_by_board(conn: &mut PgConnection, board_id: i64) -> Vec<Card> {
    sqlx::query_as("SELECT * FROM cards WHERE board_id = $1")
        .bind(board_id)
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch cards by status
async fn cards_by_status(conn: &mut PgConnection, status: Status) -> Vec<Card> {
    sqlx::query_as("SELECT * FROM cards WHERE status = $1")
        .bind(status)
        .fetch_all(conn)
        .await
        .unwrap()
}
```

Okay, the final select request we need to support is getting the board summary. We first need to add a `From<Vec<(i64, Status)>>` impl for `BoardSummary`:

```rust
// src/models.rs

#[derive(Default, serde::Serialize)]
pub struct BoardSummary {
    pub todo: i64,
    pub doing: i64,
    pub done: i64,
}

// convert list of status counts into a board summary
impl From<Vec<(i64, Status)>> for BoardSummary {
    fn from(counts: Vec<(i64, Status)>) -> BoardSummary {
        let mut summary = BoardSummary::default();
        for (count, status) in counts {
            match status {
                Status::Todo => summary.todo += count,
                Status::Doing => summary.doing += count,
                Status::Done => summary.done += count,
            }
        }
        summary
    }
}
```

And then the function to fetch the board summary is as simple as:

```rust
// fetch board summary
async fn board_summary(conn: &mut PgConnection, board_id: i64) -> BoardSummary {
    sqlx::query_as(
        "SELECT COUNT(*), status FROM cards WHERE board_id = $1 GROUP BY status",
    )
    .bind(board_id)
    .fetch_all(conn)
    .await
    .unwrap()
    .into()
}
```



#### Inserting Data

Examples of creating boards and cards:

```rust
// create board from CreateBoard model
async fn create_board(conn: &mut PgConnection, create_board: CreateBoard) -> Board {
    sqlx::query_as("INSERT INTO boards (name) VALUES ($1) RETURNING *")
        .bind(create_board.name)
        .fetch_one(conn)
        .await
        .unwrap()
}

// create card from CreateCard model
async fn create_card(conn: &mut PgConnection, create_card: CreateCard) -> Card {
    sqlx::query_as("INSERT INTO cards (board_id, description) VALUES ($1, $2) RETURNING *")
        .bind(create_card.board_id)
        .bind(create_card.description)
        .fetch_one(conn)
        .await
        .unwrap()
}
```



#### Updating Data

Example of updating cards:

```rust
// update card from UpdateCard model
async fn update_card(conn: &mut PgConnection, card_id: i64, update_card: UpdateCard) -> Card {
    sqlx::query_as("UPDATE cards SET description = $1, status = $2 WHERE id = $3 RETURNING *")
        .bind(update_card.description)
        .bind(update_card.status)
        .bind(card_id)
        .fetch_one(conn)
        .await
        .unwrap()
}
```



#### Deleting Data

Examples of deleting boards and cards:

```rust
// delete all boards
fn delete_all_boards(conn: &mut PgConnection) {
    sqlx::query("DELETE FROM boards")
        .execute(conn)
        .await
        .unwrap();
}

// delete a board by its id
fn delete_board_by_id(conn: &mut PgConnection, board_id: i64) {
    sqlx::query("DELETE FROM boards WHERE id = $1")
        .bind(board_id)
        .execute(conn)
        .await
        .unwrap();
}

// delete all cards
fn delete_all_cards(conn: &mut PgConnection) {
    sqlx::query("DELETE FROM cards")
    .execute(conn)
    .await
    .unwrap();
}

// delete a card by its id
fn delete_card_by_id(conn: &mut PgConnection, card_id: i64) {
    sqlx::query("DELETE FROM cards WHERE id = $1")
        .bind(card_id)
        .execute(conn)
        .await
        .unwrap();
}

// delete all of the cards on a board
fn delete_cards_by_board(conn: &mut PgConnection, board_id: i64) {
    sqlx::query("DELETE FROM cards WHERE board_id = $1")
        .bind(board_id)
        .execute(conn)
        .await
        .unwrap();
}

// delete all of the done cards on a board
fn delete_done_cards_by_board(conn: &mut PgConnection, board_id: i64) {
    sqlx::query("DELETE FROM cards WHERE board_id = $1 AND status = 'done'")
        .bind(board_id)
        .execute(conn)
        .await
        .unwrap();
}
```

You may have noticed we switched from using `sqlx::query_as` to `sqlx::query`. The difference between the two is that the former attempts to map the result rows into some type which impls `sqlx::FromRow` whereas the latter doesn't, so the latter is more appropriate for `DELETE` queries that don't return any rows.


#### Compile-Time Verification of SQL Queries

One interesting feature of sqlx is that it can verify all of our queries are syntactically and semantically valid at compile-time, but this is behavior we have to opt into by using the `sqlx::query!` and `sqlx::query_as!` macros over the `sqlx::query` and `sqlx::query_as` functions. The downsides of the macros are that they're slightly less ergonomic to use than the regular functions and they will increase compiles but the upsides can greatly outweigh the downsides if we're working in a project with lots of complex queries or many tables and entities. I haven't been using this for this project but they're worth knowing about.



#### Using a Connection Pool w/sqlx

sqlx comes built-in with a connection pool so we don't have to install any additional dependencies!

Here's how we'd create a pool with the default configuration options:

```rust
use sqlx::{Connection, PgConnection, Pool, Postgres};
use sqlx::postgres::PgPoolOptions;

// example of connecting to Pg Pool
async fn get_pool() -> Pool<Postgres> {
    dotenv::dotenv().unwrap();
    let db_url = std::env::var("DATABASE_URL").unwrap();
    PgPoolOptions::new().connect(&db_url).await.unwrap()
}

async fn use_pool(pool: &Pool<Postgres>) {
    // don't need to fetch connection from pool
    // can pass pool directly to queries, e.g.
    sqlx::query_as::<_,(String,)>("SELECT version()")
        .fetch_one(pool) // passing pool directly
        .await
        .unwrap();
}
```




#### Refactoring DB Operations Into a Module

Let's clean up our code and neatly put everything into a single module:

```rust
// src/db.rs

use sqlx::{Connection, PgConnection, Pool, Postgres, postgres::PgPoolOptions};
use crate::models::*;
use crate::StdErr;

pub struct Db {
    pool: Pool<Postgres>,
}

impl Db {
    pub async fn connect() -> Result<Self, StdErr> {
        let db_url = std::env::var("DATABASE_URL")?;
        let pool = PgPoolOptions::new().connect(&db_url).await?;
        Ok(Db { pool })
    }

    pub async fn boards(&self) -> Result<Vec<Board>, StdErr> {
        let boards = sqlx::query_as("SELECT * FROM boards")
            .fetch_all(&self.pool)
            .await?;
        Ok(boards)
    }

    pub async fn board_summary(&self, board_id: i64) -> Result<BoardSummary, StdErr> {
        let counts: Vec<(i64, Status)> = sqlx::query_as(
            "SELECT count(*), status FROM cards WHERE board_id = $1 GROUP BY status",
        )
        .bind(board_id)
        .fetch_all(&self.pool)
        .await?;
        Ok(counts.into())
    }

    pub async fn create_board(&self, create_board: CreateBoard) -> Result<Board, StdErr> {
        let board = sqlx::query_as("INSERT INTO boards (name) VALUES ($1) RETURNING *")
            .bind(&create_board.name)
            .fetch_one(&self.pool)
            .await?;
        Ok(board)
    }

    pub async fn delete_board(&self, board_id: i64) -> Result<(), StdErr> {
        sqlx::query("DELETE FROM boards WHERE id = $1")
            .bind(board_id)
            .execute(&self.pool)
            .await?;
        Ok(())
    }

    pub async fn cards(&self, board_id: i64) -> Result<Vec<Card>, StdErr> {
        let cards = sqlx::query_as("SELECT * FROM cards WHERE board_id = $1")
            .bind(board_id)
            .fetch_all(&self.pool)
            .await?;
        Ok(cards)
    }

    pub async fn create_card(&self, create_card: CreateCard) -> Result<Card, StdErr> {
        let card =
            sqlx::query_as("INSERT INTO cards (board_id, description) VALUES ($1, $2) RETURNING *")
                .bind(&create_card.board_id)
                .bind(&create_card.description)
                .fetch_one(&self.pool)
                .await?;
        Ok(card)
    }

    pub async fn update_card(&self, card_id: i64, update_card: UpdateCard) -> Result<Card, StdErr> {
        let card = sqlx::query_as(
            "UPDATE cards SET description = $1, status = $2 WHERE id = $3 RETURNING *",
        )
        .bind(&update_card.description)
        .bind(&update_card.status)
        .bind(card_id)
        .fetch_one(&self.pool)
        .await?;
        Ok(card)
    }

    pub async fn delete_card(&self, card_id: i64) -> Result<(), StdErr> {
        sqlx::query("DELETE FROM cards WHERE id = $1")
            .bind(card_id)
            .execute(&self.pool)
            .await?;
        Ok(())
    }
}
```

Updated main file:

```diff
// src/main.rs

+mod db;
mod logger;
mod models;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```




### HTTP Routing w/actix-web

crates
- actix-web

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
sqlx = { version = "0.4", features = ["runtime-actix-rustls", "chrono", "postgres"] }
+ actix-web = "3.3"
```

Let's throw together a quick actix-web hello world example into our main:

```diff
mod db;
mod logger;
mod models;

type StdErr = Box<dyn std::error::Error>;

+#[actix_web::get("/")]
+async fn hello_world() -> &'static str {
+    "Hello, world!"
+}

+#[actix_web::main]
-fn main() -> Result<(), StdErr> {
+async fn main() -> Result<(), StdErr> {
    dotenv::dotenv().ok();
    logger::init()?;

+   actix_web::HttpServer::new(move || actix_web::App::new().service(hello_world))
+       .bind(("127.0.0.1", 8000))?
+       .run()
+       .await?;

    Ok(())
}
```

We have to make our main function async and decorate it with the `actix_web::main` procedural macro to tell actix-web to start a runtime and execute our main function as the first task.



#### Routing Basics

We can decorate functions with actix-web's procedural macros to route incoming HTTP requests:

```rust
#[actix_web::get("/")]
async fn index() -> &'static str {
    "I'm the index route!"
}

#[actix_web::get("/nested/route")]
async fn nested_route() -> &'static str {
    "I'm a nested route!"
}
```

Data can be extracted from the path using `{param}` placeholders:

```rust
use actix_web::web::Path;

#[actix_web::get("/hello/{name}")]
async fn greet(Path(name): Path<String>) -> String {
    format!("Hello, {}!", name)
}
```

Any type which impls `serde::Deserialize` can be extracted from within `actix_web::web::Path`:

```rust
// actix_web::web::Path can extract anything out of the
// URL path as long as it impls serde::Deserialize
#[actix_web::get("/echo/{string}/{num}/{maybe}/etc")]
async fn echo_path(Path((string, num, maybe)): Path<(String, usize, bool)>) -> String {
    format!("got string {}, num {}, and maybe {}", string, num, maybe)
}

// custom type example
struct EvenNumber(i32);

// hand-written deserialize impl, mostly deferring to i32::deserialize
impl<'de> Deserialize<'de> for EvenNumber {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let value = i32::deserialize(deserializer)?;
        if value % 2 == 0 {
            Ok(EvenNumber(value))
        } else {
            Err(D::Error::custom("not even"))
        }
    }
}

// but now we can extract EvenNumbers directly from the Path:
#[actix_web::get("/even/{even_num}")]
async fn echo_even(Path(even_num): Path<EvenNumber>) -> String {
    format!("got even number {}", even_num.0)
}
```

The response type can be anything that impls `actix_web::Responder`:

```rust
use std::future::{ready, Ready};
use actix_web::{HttpResponse, HttpRequest};
use actix_web::error::InternalError;

// actix_web provides Responder impls for
// - &'static str
// - &'static [u8]
// - String
// - &String
// - some other misc types
// have to use actix_files to get Responder impls for
// - Files
#[actix_web::get("/cargo")]
async fn returns_cargo() -> actix_files::NamedFile {
    actix_files::NamedFile::open("Cargo.toml").unwrap()
}

type ActixError = actix_web::error::Error;

// actix_web also provides Responder impls for
// - Option<T> where T: Responder
//     - returns T if Some(T), 404 Not Found if None
// - Result<T, E> where T: Responder, E: Into<ActixError>
//     - returns T if Ok(T), otherwise ActixError::from(e) if Err(e)

// example custom type
struct EvenNumber(i32);

// example Responder impl
impl Responder for EvenNumber {
    type Error = InternalError<&'static str>;
    type Future = Ready<Result<HttpResponse, Self::Error>>;

    fn respond_to(self, _req: &HttpRequest) -> Self::Future {
        let res = HttpResponse::Ok()
            .set_header("X-Number-Parity", "Even")
            .body(format!("returning even number {}", self.0));
        ready(Ok(res))
    }
}

// returning custom defined type
#[actix_web::get("/even")]
async fn returns_even() -> EvenNumber {
    EvenNumber(2)
}
```



#### GET Requests

We have to pass our `Db` to our actix-web request handlers. We can do this is by passing an instance of the `Db` to the `data` method of the actix-web application factory function, and then we can receive it in our request handlers using the `actix_web::web::Data` extractor.

Since the application factory function creates an application per system thread, the data referenced inside the factory has to be cloneable, so we first have to impl `Clone` for `Db` which is easy since `Pool<Postgres>` already impls `Clone` so we can just derive it:

```diff
// src/db.rs

use sqlx::{Pool, Postgres};

+#[derive(Clone)]
pub struct Db {
    pool: Pool<Postgres>,
}

// etc
```

And then this is how we'd update our main file:

```diff
// src/main.rs

mod db;
mod logger;
mod models;
mod routes;

type StdErr = Box<dyn std::error::Error>;

#[actix_web::get("/")]
async fn hello_world() -> &'static str {
    "Hello, world!"
}

#[actix_web::main]
async fn main() -> Result<(), StdErr> {
    dotenv::dotenv().ok();
    logger::init()?;

+   let db = db::Db::connect().await?;

    actix_web::HttpServer::new(move || {
        actix_web::App::new()
+           .data(db.clone())
            .service(hello_world)
    })
    .bind(("127.0.0.1", 8000))?
    .run()
    .await?;

    Ok(())
}
```

And here's how we'd use the `Data` extractor to receive the `Db` instance in our request handlers:

```rust
use actix_web::web::Data;
use crate::db::Db;

#[actix_web::get("/use/db")]
fn use_db(db: Data<Db>) {
    // use db
}
```

We can automatically return JSON by wrapping the return type in `actix_web::web::Json` which impls `actix_web::Responder` and will serialize the wrapped type to JSON using serde and set the response header `Content-Type: application/json`. Example:

```rust
use actix_web::web::Json;
use crate::models::Status;

#[actix_web::get("/example/json")]
fn return_json() -> Json<Status> {
    Json(Status::Todo)
}
```

Furthermore, because actix-web does not impl `Responder` for `Box<dyn std::error::Error>` we have to wrap it with some type which does, and we can use the generic `actix_web::error::InternalError` type for this purpose:

```rust
use actix_web::error::InternalError;
use actix_web::http::StatusCode;
use crate::StdErr;

fn some_fallible_function() -> Result<&'static str, StdErr> {
    todo!()
}

// map StdErr to an error that impls Responder
fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

#[actix_web::get("/error")]
async fn return_error() -> Result<&'static str, InternalError<StdErr>> {
    some_falliable_function().map_err(to_internal_error)
}
```

Okay, we now have enough context to write all of our GET request handlers:

```rust
use actix_web::web::{Data, Json};
use actix_web::error::InternalError;
use actix_web::http::StatusCode;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, Card, BoardSummary};

// convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

// GET requests

#[actix_web::get("/boards")]
async fn boards(db: Data<Db>) -> Result<Json<Vec<Board>>, InternalError<StdErr>> {
    db.boards()
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::get("/boards/{board_id}/summary")]
async fn board_summary(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<Json<BoardSummary>, InternalError<StdErr>> {
    db.board_summary(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::get("/boards/{board_id}/cards")]
async fn cards(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<Json<Vec<Card>>, InternalError<StdErr>> {
    db.cards(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}
```



#### POST & PATCH Requests

`actix_web::web::Json` not only impls `actix_web::Responder` but also impls `actix_web::FromRequest` which means we can add it as a function parameter to any request handler and actix-web will attempt deserialize the request's JSON body into the type we specify, so here's how we'd write the POST & PATCH request handlers:

```rust
use actix_web::web::{Data, Json};
use actix_web::error::InternalError;
use actix_web::http::StatusCode;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, Card, CreateBoard, CreateCard};

// convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

// POST requests

#[actix_web::post("/boards")]
async fn create_board(
    db: Data<Db>,
    create_board: Json<CreateBoard>,
) -> Result<Json<Board>, InternalError<StdErr>> {
    db.create_board(create_board.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/cards")]
async fn create_card(
    db: Data<Db>,
    create_card: Json<CreateCard>,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.create_card(create_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

// PATCH requests

#[actix_web::patch("/cards/{card_id}")]
async fn update_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
    update_card: Json<UpdateCard>,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.update_card(card_id, update_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}
```



#### DELETE Requests

The DELETE request handlers:

```rust
use actix_web::web::{Data, Json};
use actix_web::error::InternalError;
use actix_web::http::StatusCode;

use crate::StdErr;
use crate::db::Db;

// some convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

fn to_ok(_: ()) -> HttpResponse {
    HttpResponse::new(StatusCode::OK)
}

// DELETE requests

#[actix_web::delete("/boards/{board_id}")]
async fn delete_board(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_board(board_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

#[actix_web::delete("/cards/{card_id}")]
async fn delete_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_card(card_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}
```



#### Refactoring API Routes Into a Module

Let's clean up the code and put all of the API routes into their own module:

```rust
// src/routes.rs

use actix_web::web::{Data, Json, Path};
use actix_web::http::StatusCode;
use actix_web::error::InternalError;
use actix_web::dev::HttpServiceFactory;
use actix_web::{HttpResponse, Responder};

use crate::StdErr;
use crate::db::Db;
use crate::models::*;

// some convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

fn to_ok(_: ()) -> HttpResponse {
    HttpResponse::new(StatusCode::OK)
}

// board routes

#[actix_web::get("/boards")]
async fn boards(db: Data<Db>) -> Result<Json<Vec<Board>>, InternalError<StdErr>> {
    db.boards()
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/boards")]
async fn create_board(
    db: Data<Db>,
    create_board: Json<CreateBoard>,
) -> Result<Json<Board>, InternalError<StdErr>> {
    db.create_board(create_board.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::get("/boards/{board_id}/summary")]
async fn board_summary(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<Json<BoardSummary>, InternalError<StdErr>> {
    db.board_summary(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::delete("/boards/{board_id}")]
async fn delete_board(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_board(board_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

// card routes

#[actix_web::get("/boards/{board_id}/cards")]
async fn cards(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<Json<Vec<Card>>, InternalError<StdErr>> {
    db.cards(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/cards")]
async fn create_card(
    db: Data<Db>,
    create_card: Json<CreateCard>,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.create_card(create_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::patch("/cards/{card_id}")]
async fn update_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
    update_card: Json<UpdateCard>,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.update_card(card_id, update_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::delete("/cards/{card_id}")]
async fn delete_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_card(card_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

// single public function which returns all of the API request handlers

pub fn api() -> impl HttpServiceFactory + 'static {
    actix_web::web::scope("/api")
        .service(boards)
        .service(board_summary)
        .service(create_board)
        .service(delete_board)
        .service(cards)
        .service(create_card)
        .service(update_card)
        .service(delete_card)
}
```

Updated main file:

```diff
// src/main.rs

mod db;
mod logger;
mod models;
+mod routes;

type StdErr = Box<dyn std::error::Error>;

#[actix_web::get("/")]
async fn hello_world() -> &'static str {
    "Hello, world!"
}

#[actix_web::main]
async fn main() -> Result<(), StdErr> {
    dotenv::dotenv().ok();
    logger::init()?;

    let db = db::Db::connect().await?;

    actix_web::HttpServer::new(move || {
        actix_web::App::new()
            .data(db.clone())
            .service(hello_world)
+           .service(routes::api())
    })
    .bind(("127.0.0.1", 8000))?
    .run()
    .await?;

    Ok(())
}
```



#### Authentication

Let's now implement token-based authentication by checking the `Authorization` HTTP header in requests.

Generating a new migration:

```bash
sqlx migrate add create_tokens
```

The create `tokens` table query:

```sql
-- create_tokens.sql
CREATE TABLE IF NOT EXISTS tokens (
    id TEXT PRIMARY KEY,
    expired_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- seed db with some local test data
INSERT INTO tokens
(id, expired_at)
VALUES
('LET_ME_IN', (CURRENT_TIMESTAMP + INTERVAL '15 minutes') AT TIME ZONE 'utc');
```

Creating our `Token` model:

```diff
// src/models.rs

+// for authentication
+
+#[derive(sqlx::FromRow)]
+pub struct Token {
+    pub id: String,
+    pub expired_at: chrono::DateTime<chrono::Utc>,
+}

// other models
```

Validating tokens using our `Db`:

```diff
use sqlx::{postgres::PgPoolOptions, Connection, PgConnection, Pool, Postgres};
use crate::models::*;
use crate::StdErr;

#[derive(Clone)]
pub struct Db {
    pool: Pool<Postgres>,
}

impl Db {
    pub async fn connect() -> Result<Self, StdErr> {
        let db_url = std::env::var("DATABASE_URL")?;
        let pool = PgPoolOptions::new().connect(&db_url).await?;
        Ok(Db { pool })
    }

+   // token methods
+
+   pub async fn validate_token<T: AsRef<str>>(&self, token_id: T) -> Result<Token, StdErr> {
+       let token_id = token_id.as_ref();
+       let token = sqlx::query_as("SELECT * FROM tokens WHERE id = $1 AND expired_at > current_timestamp")
+           .bind(token_id)
+           .fetch_one(&self.pool)
+           .await?;
+       Ok(token)
+   }

    // other methods
}
```

There's a couple ways we can implement authentication in actix-web. We can implement it as middleware or as an extractor. Extractors are the actix-web equivalent to Rocket's request guards, and since we implemented authentication using a request guard in Rocket, let's use an extractor for actix-web.

Since everything in actix-web is async, including the `actix_web::FromRequest` method signature, let's add the `futures` crate to our project for some handy utility future traits and functions:

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
sqlx = { version = "0.4", features = ["runtime-actix-rustls", "chrono", "postgres"] }
actix-web = "3.3"
+futures = "0.3.14"
```

Okay, so here's the `actix_web::FromRequest` impl for `Token`:

```rust
use std::future::Ready;
use std::pin::Pin;

use actix_web::{FromRequest, HttpRequest};
use actix_web::http::StatusCode;
use actix_web::error::InternalError;
use actix_web::dev::Payload;
use futures::{future, Future, FutureExt};

use crate::StdErr;
use crate::db::Db;
use crate::models::Token;

impl FromRequest for Token {
    type Error = InternalError<&'static str>;
    type Config = ();

    // we return a Future that is either
    // - immediately ready (on a bad request with a missing or malformed Authorization header)
    // - ready later (pending on a SQL query that validates the request's Bearer token)
    type Future = future::Either<
        future::Ready<Result<Self, Self::Error>>,
        Pin<Box<dyn Future<Output = Result<Self, Self::Error>> + 'static>>,
    >;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
        // get request headers
        let headers = req.headers();

        // check that Authorization header exists
        let maybe_auth = headers.get("Authorization");
        if maybe_auth.is_none() {
            return future::err(InternalError::new(
                "missing Authorization header",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // check Authorization header is valid utf-8
        let auth_config = maybe_auth.unwrap().to_str();
        if auth_config.is_err() {
            return future::err(InternalError::new(
                "malformed Authorization header",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // check Authorization header specifies some authorization strategy
        let mut auth_config_parts = auth_config.unwrap().split_ascii_whitespace();
        let maybe_auth_type = auth_config_parts.next();
        if maybe_auth_type.is_none() {
            return future::err(InternalError::new(
                "missing Authorization type",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // check that authorization strategy is using a bearer token
        let auth_type = maybe_auth_type.unwrap();
        if auth_type != "Bearer" {
            return future::err(InternalError::new(
                "unsupported Authorization type",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // check that bearer token is present
        let maybe_token_id = auth_config_parts.next();
        if maybe_token_id.is_none() {
            return future::err(InternalError::new(
                "missing Bearer token",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // we can fetch managed application data using HttpRequest.app_data::<T>()
        let db = req.app_data::<Data<Db>>();
        if db.is_none() {
            return future::err(InternalError::new(
                "internal error",
                StatusCode::INTERNAL_SERVER_ERROR,
            ))
            .left_future();
        }

        // clone these so that we can return an impl Future + 'static
        let db = db.unwrap().clone();
        let token_id = maybe_token_id.unwrap().to_owned();

        async move {
            db.validate_token(token_id)
                .await
                .map_err(|_| InternalError::new("invalid Bearer token", StatusCode::UNAUTHORIZED))
        }
        .boxed_local()
        .right_future()
    }
}
```

Updated routes file:

```diff
use std::pin::Pin;

use actix_web::{FromRequest, HttpRequest, HttpResponse};
use actix_web::web::{Data, Json, Path};
use actix_web::http::StatusCode;
use actix_web::error::InternalError;
use actix_web::dev::{HttpServiceFactory, Payload};
use futures::{Future, FutureExt, future};

use crate::StdErr;
use crate::db::Db;
use crate::models::*;

impl FromRequest for Token {
    type Error = InternalError<&'static str>;
    type Config = ();
    type Future = future::Either<
        future::Ready<Result<Self, Self::Error>>,
        Pin<Box<dyn Future<Output = Result<Self, Self::Error>> + 'static>>,
    >;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
       // impl
    }
}

// some convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

fn to_ok(_: ()) -> HttpResponse {
    HttpResponse::new(StatusCode::OK)
}

// board routes

#[actix_web::get("/boards")]
async fn boards(
    db: Data<Db>,
+   _t: Token
) -> Result<Json<Vec<Board>>, InternalError<StdErr>> {
    db.boards()
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/boards")]
async fn create_board(
    db: Data<Db>,
    create_board: Json<CreateBoard>,
+   _t: Token,
) -> Result<Json<Board>, InternalError<StdErr>> {
    db.create_board(create_board.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::get("/boards/{board_id}/summary")]
async fn board_summary(
    db: Data<Db>,
    Path(board_id): Path<i64>,
+   _t: Token,
) -> Result<Json<BoardSummary>, InternalError<StdErr>> {
    db.board_summary(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::delete("/boards/{board_id}")]
async fn delete_board(
    db: Data<Db>,
    Path(board_id): Path<i64>,
+   _t: Token,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_board(board_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

// card routes

#[actix_web::get("/boards/{board_id}/cards")]
async fn cards(
    db: Data<Db>,
    Path(board_id): Path<i64>,
+   _t: Token,
) -> Result<Json<Vec<Card>>, InternalError<StdErr>> {
    db.cards(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/cards")]
async fn create_card(
    db: Data<Db>,
    create_card: Json<CreateCard>,
+   _t: Token,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.create_card(create_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::patch("/cards/{card_id}")]
async fn update_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
    update_card: Json<UpdateCard>,
+   _t: Token,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.update_card(card_id, update_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::delete("/cards/{card_id}")]
async fn delete_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
+   _t: Token,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_card(card_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

pub fn api() -> impl HttpServiceFactory + 'static {
    actix_web::web::scope("/api")
        .service(boards)
        .service(board_summary)
        .service(create_board)
        .service(delete_board)
        .service(cards)
        .service(create_card)
        .service(update_card)
        .service(delete_card)
}
```

We did it! Again! Look at us go, we're implementation machines! The full source code for the sqlx + actix-web implementation can be found in the [companion code repository](https://github.com/pretzelhammer/kanban/tree/main/sqlx-actix-web) for this article.

Since we have two identical RESTful API servers but with two different implementations let's benchmark them and see which is faster :)



## Benchmarks



### Servers

I don't normally benchmark RESTful API servers so I don't know what "good performance" is and what "bad performance" is. Aside from comparing the Diesel + Rocket server to the sqlx + actix-web server I've decided to throw in a couple node.js servers to make things more interesting. The full source code for the benchmarks as well as all the servers can be found in the [companion code repository](https://github.com/pretzelhammer/kanban) for this article. Here's a list of the servers we will be benchmarking:

Server #1: Diesel + Rocket
- Nickname: DR
- Connection pool: r2d2
- SQL executor: Diesel
- HTTP routing: Rocket
- Compiled with: Rust v1.53 (Nightly)

Server #2: sqlx + actix-web
- Nickname: SA
- Connection pool: sqlx
- SQL executor: sqlx
- HTTP Routing: actix-web
- Compiled with: Rust v1.53 (Nightly)

Server #3: pg-promise + express.js (single process)
- Nickname: PES
- Connection pool: pg-promise
- SQL executor: pg-promise
- HTTP Routing: express.js
- Interpreted with: node.js v16.0.0
- Mode: single process

Server #4: pg-promise + express.js (multi process)
- Nickname: PEM
- Connection pool: pg-promise
- SQL executor: pg-promise
- HTTP Routing: express.js
- Interpreted with: node.js v16.0.0
- Mode: multi process 

Test Machine: DigitalOcean VPS
- OS: Ubuntu 20.04 (LTS)
- CPU: 2.3 GHz 4-core Intel
- Memory: 8 GB

Since almost everyone builds and ships web servers to the cloud nowadays I decided to run all the benchmarks on a DigitalOcean VPS.



### Methodology

The tools we're going to use for benchmarking and profiling are vegeta and psutil. Vegeta is an HTTP load testing command line tool written in Go. We can give it a list of targets, a duration, and a number of workers and it will pummel the targets for the given duration using the given number of workers while recording statistics like the number of requests successfully processed, their response times, their status codes, and so on. Psutil is a Python library that we can use to easily write a script that queries the system every second to check how much CPU and memory a process is using.

Let's run all of the HTTP load tests for 60 seconds and use up to a max of 40 workers. Vegeta will naturally scale the workers if the HTTP server can handle the load. Also, let's use two different sets of targets, the first set is going to represent a read-only (RO) workload and the second set is going to represent a more realistic reads + writes (RW) workload.

Here's the RO workload target list:

```none
GET http://localhost:8000/api/boards
Authorization: Bearer LET_ME_IN

GET http://localhost:8000/api/boards/1/summary
Authorization: Bearer LET_ME_IN

GET http://localhost:8000/api/boards/1/cards
Authorization: Bearer LET_ME_IN
```

Here's the RW workload target list:

```none
GET http://localhost:8000/api/boards
Authorization: Bearer LET_ME_IN

GET http://localhost:8000/api/boards/1/summary
Authorization: Bearer LET_ME_IN

POST http://localhost:8000/api/boards
Authorization: Bearer LET_ME_IN
Content-Type: application/json
@post-board.json

DELETE http://localhost:8000/api/boards/10000
Authorization: Bearer LET_ME_IN

GET http://localhost:8000/api/boards/1/cards
Authorization: Bearer LET_ME_IN

POST http://localhost:8000/api/cards
Authorization: Bearer LET_ME_IN
Content-Type: application/json
@post-card.json

PATCH http://localhost:8000/api/cards/1
Authorization: Bearer LET_ME_IN
Content-Type: application/json
@patch-card.json

DELETE http://localhost:8000/api/cards/10000
Authorization: Bearer LET_ME_IN
```

And here are the test JSON payloads:

```js
// post-board.json
{"name": "Vegeta Stress Test Board"}

// post-card.json
{"boardId": 3, "description": "Vegeta Stress Test Card"}

// patch-card.json
{"description": "Vegeta Stress Update Card", "status": "doing"}
```

Of course we're going to compile the Rust servers with this command:

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

And with this release profile:

```toml
[profile.release]
debug = 0
lto = true
codegen-units = 1
panic = "abort"
```

Also, let's disable the logging in the Rust servers for the benchmarks because I didn't bother adding any logging to the node.js servers. And finally, I ran the benchmarks a few times a day over the course of a couple days and took the best results for every individual server and benchmark.



### Measuring Resource Usage

I'm going to measure CPU usage in CPU seconds and memory usage in megabytes. Everyone knows what megabytes are so I'm not going to explain those, but not everyone is familiar with CPU seconds and they're kinda weird so let's discuss those now.

When people say "CPU second" what they really mean is "CPU logical core second." For example, a CPU with 16 logical cores can perform 16 CPU seconds of processing for every second of wall clock time. This is why when you open the process manager tool in your operating system of choice you'll occasionally see it report some processes as using over 100% CPU. This makes no sense, as it's not possible to use more than 100% of anything, but it's reported this way because the percent is calculated based on the processing power of a single logical core of the CPU and not the total processing power of the entire CPU. For example, a process running on a CPU with 4 logical cores can use up to "400%" of the CPU. Yup, it's dumb, but whatever, now you know.



### Results

As a quick reminder, the servers we're benchmarking:
- DR: Diesel + Rocket
- SA: sqlx + actix-web
- PES: pg-promise + express.js (single process)
- PEM: pg-promise + express.js (multi process)

The workloads we're using for the benchmarks:
- Read-only (RO) workload for 60 seconds with up to 40 workers
- Reads + Writes (RW) workload for 60 seconds with up to 40 workers

The stats we're tracking:
- Total requests processed
- How many of the requests were successful (return 200 status code)
- CPU usage (in CPU seconds)
- Memory usage (in megabytes)

And we're running these benchmarks on a DigitalOcean VPS with:
- OS: Ubuntu 20.04 (LTS)
- CPU: 2.3 GHz 4-core Intel
- Memory: 8 GB

Disclaimer: Like I said before, I don't normally benchmark anything so it's likely I may have gotten some things wrong or biased the tests toward one server or another, so don't take the results below as proof of anything, they're more for entertainment than anything else.



#### Read-Only Workload

**Request Throughput**

Absolute measurements

| Server | Total Requests | Successful Requests | Success Rate | Successful Requests per Second |
|-|-|-|-|-|
| **DR** | 171431 req | 171431 req | 100% 🥇 | 2857 req/sec |
| **SA** | 275803 req 🥇 | 275803 req 🥇 | 100% 🥇 | 4567 req/sec 🥇 |
| **PES** | 115708 req | 115708 req | 100% 🥇 | 1928 req/sec |
| **PEM** | 190624 req | 190624 req | 100% 🥇 | 3177 req/sec |

Relative measurements

| Server | Total Requests | Successful Requests | Success Rate | Successful Requests per Second |
|-|-|-|-|-|
| **DR** | 0.62x | 0.62x | 1.00x 🥇 | 0.62x |
| **SA** | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 |
| **PES** | 0.42x | 0.42x | 1.00x 🥇 | 0.42x |
| **PEM** | 0.69x | 0.69x | 1.00x 🥇 | 0.69x |

**Request Latencies**

Absolute measurements

| Server | Min | Avg | 50th percentile | 90th percentile | 95th percentile | 99th percentile | Max |
|-|-|-|-|-|-|-|-|
| **DR** | 856 µs | 11.5 ms | 10.3 ms | 19.9 ms | 23.4 ms | 32.0 ms | 109.6 ms 🥇 |
| **SA** | 830 µs 🥇 | 8.6 ms 🥇 | 7.4 ms 🥇 | 14.5 ms 🥇 | 17.7 ms 🥇 | 26.2 ms 🥇 | 191.1 ms |
| **PES** | 7.5 ms | 20.7 ms | 19.8 ms | 28.0 ms | 31.4 ms | 39.6 ms | 182.0 ms |
| **PEM** | 889 µs | 12.6 ms | 10.7 ms | 23.4 ms | 28.8 ms | 41.7 ms | 214.2 ms |

Relative measurements

| Server | Min | Avg | 50th percentile | 90th percentile | 95th percentile | 99th percentile | Max |
|-|-|-|-|-|-|-|-|
| **DR** | 1.03x | 1.34x | 1.39x | 1.37x | 1.32x | 1.22x | 1.00x 🥇 |
| **SA** | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.74x |
| **PES** | 9.04x | 2.41x | 2.68x | 1.93x | 1.77x | 1.51x | 1.66x |
| **PEM** | 1.07x | 1.47x | 1.45x | 1.61x | 1.63x | 1.59x | 1.95x |

**Resource Usage**

Absolute measurements

| Server | Total CPU Seconds | Avg CPU Seconds per Second | Avg CPU Seconds per Successful Request | Max Memory Used |
|-|-|-|-|-|
| **DR** | 44.49 cpu 🥇 | 0.74 cpu/sec 🥇 | 0.00026 cpu/req | 14.2 MB |
| **SA** | 68.77 cpu | 1.15 cpu/sec | 0.00025 cpu/req 🥇 | 10.6 MB 🥇 |
| **PES** | 52.64 cpu | 0.88 cpu/sec | 0.00045 cpu/req | 117.5 MB |
| **PEM** | 123.60 cpu | 2.06 cpu/sec | 0.00065 cpu/req | 476.8 MB |

Relative measurements

| Server | Total CPU Seconds | Avg CPU Seconds per Second | Avg CPU Seconds per Successful Request | Max Memory Used |
|-|-|-|-|-|
| **DR** | 1.00x 🥇 | 1.00x 🥇 | 1.04x | 1.34x |
| **SA** | 1.55x | 1.55x | 1.00x 🥇 | 1.00x 🥇 |
| **PES** | 1.18x | 1.18x | 1.80x | 11.08x |
| **PEM** | 2.78x | 2.78x | 2.60x | 44.98x |


The DR server had a request throughput of 2857 req/sec with an average response latency of 11.5 ms, an average CPU utilization of 0.74 cpu/sec, and used 14.2 MB of memory. All of that sounds pretty solid to me!

The SA server had a request throughput of 4567 req/sec, which is 160% the performance of the DR server! The SA server had an average response latency of 7.4 ms, an average CPU utilization of 1.15 cpu/sec, and used 10.6 MB of memory. So it used a bit more CPU than the DR server but a bit less memory.

The PES server had a request throughput of 1928 req/sec. Not as performant as the Rust servers, but I suppose that's to be expected because node.js is a single-threaded process so it can't take advantage of all 4 cores on the test machine to process multiple requests in parallel. PES had an average response latency of 20.7 ms which is pretty good but it's double that of the Rust servers. PES had an average CPU utilization of 0.88 cpu/sec which is similar to the Rust servers but because its request throughput was so much lower it actually took 0.00045 cpu/req which is almost double the Rust servers. PES also used 117.5 MB which is a lot but it's expected since it's node.js. In short: PES took roughly ~2x as much CPU and ~10x as much memory to only get ~0.5x of the performance of the Rust servers.

The PEM server had a request throughput of 3177 req/sec with an average response latency of 12.6 ms which is competitive with the Rust servers. Where it stops being competitive is in its resource usage, consuming an average of 2.06 cpu/sec and 476.8 MB of memory, which is signcantly higher than all the other servers.



#### Reads + Writes Workload

**Request Throughput**

Absolute measurements

| Server | Total Requests | Successful Requests | Success Rate | Successful Requests per Second |
|-|-|-|-|-|
| **DR** | 88778 req 🥇 | 74021 req 🥇 | 83% | 1234 req/sec 🥇 |
| **SA** | 62362 req | 62362 req | 100% 🥇 | 1039 req/sec |
| **PES** | 31683 req | 31683 req | 100% 🥇 | 528 req/sec |
| **PEM** | 56380 req | 56380 req | 100% 🥇 | 940 req/sec |

Relative measurements

| Server | Total Requests | Successful Requests | Success Rate | Successful Requests per Second |
|-|-|-|-|-|
| **DR** | 1.00x 🥇 | 1.00x 🥇 | 0.83x | 1.00x 🥇 |
| **SA** | 0.70x | 0.84x | 1.00x 🥇 | 0.84x |
| **PES** | 0.36x | 0.43x | 1.00x 🥇 | 0.43x |
| **PEM** | 0.64x | 0.76x | 1.00x 🥇 | 0.76x |

**Request Latencies**

Absolute measurements

| Server | Min | Avg | 50th percentile | 90th percentile | 95th percentile | 99th percentile | Max |
|-|-|-|-|-|-|-|-|
| **DR** | 31 µs 🥇 | 19.4 ms 🥇 | 15.2 ms 🥇 | 40.4 ms 🥇 | 51.4 ms 🥇 | 79.3 ms 🥇 | 277.5 ms |
| **SA** | 829 µs | 38.3 ms | 33.4 ms | 80.7 ms | 94.7 ms | 125.6 ms | 236.9 ms 🥇 |
| **PES** | 10.4 ms | 75.9 ms | 55.3 ms | 106.5 ms | 274.3 ms | 448.7 ms | 573.4 ms |
| **PEM** | 2.0 ms | 42.6 ms | 25.4 ms | 87.9 ms | 151.2 ms | 325.1 ms | 668.3 ms |

Relative measurements

| Server | Min | Avg | 50th percentile | 90th percentile | 95th percentile | 99th percentile | Max |
|-|-|-|-|-|-|-|-|
| **DR** | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.00x 🥇 | 1.17x |
| **SA** | 26.74x | 1.97x | 2.20x | 2.00x | 1.84x | 1.584x | 1.00x 🥇 |
| **PES** | 335.48x | 3.91x | 3.64x | 2.64x | 5.34x | 5.66x | 2.42x |
| **PEM** | 64.52x | 2.20x | 1.67x | 2.18x | 2.94x | 4.10x | 2.82x |

**Resource Usage**

Absolute measurements

| Server | Total CPU Seconds | Avg CPU Seconds per Second | Avg CPU Seconds per Successful Request | Max Memory Used |
|-|-|-|-|-|
| **DR** | 67.17 cpu | 1.12 cpu/sec | 0.00091 cpu/req 🥇 | 97.8 MB |
| **SA** | 149.11 cpu | 2.49 cpu/sec | 0.00239 cpu/req | 45.3 MB 🥇 |
| **PES** | 59.11 cpu 🥇 | 0.99 cpu/sec 🥇 | 0.00187 cpu/req | 154.5 MB |
| **PEM** | 163.38 cpu | 2.72 cpu/sec | 0.00290 cpu/req | 571.4 MB |

Relative measurements

| Server | Total CPU Seconds | Avg CPU Seconds per Second | Avg CPU Seconds per Successful Request | Max Memory Used |
|-|-|-|-|-|
| **DR** | 1.14x | 1.14x | 1.00x 🥇 | 2.16x |
| **SA** | 2.52x | 2.52x | 2.63x | 1.00x 🥇 |
| **PES** | 1.00x 🥇 | 1.00x 🥇 | 2.05x | 3.41x |
| **PEM** | 2.76x | 2.76x | 3.19x | 12.61x |

I was expecting these results to be a repeat of the previous results but slower but to my surprise they came out very differently! Request throughputs went down across the board. CPU utilization, memory utilization, and average response latencies went up across the board.

For starters, the DR server is the only server which failed to successfully process all requests, but despite that it still had the highest request throughput at 1234 req/sec! Technically it has the best performance but if I was working with a server that dropped 1 out of 5 requests I would be very annoyed.

The SA server had a request throughput of 1039 req/sec which is 84% the performance of the DR server, but it at least processed all request successfully, so I'm not sure which is better in practice. I was sure after the first benchmark that actix-web was faster than Rocket, but now I'm not so sure. It's possible the difference in these results might be caused by Diesel having better performance for write queries than sqlx. I'm just speculating here, don't take anything I say too seriously. SA used ~2.6x as much CPU but ~0.5x as much memory as DR.

Nothing too interesting to say about the node.js servers in this benchmark, similarly to the previous benchmark they used a lot more CPU and memory to get worse performance relative to the Rust servers.



## Concluding Thoughts



### Diesel vs sqlx

I'm gonna be upfront about my biases here: I love SQL and I hate ORMs.

In my opinion, SQL is already an amazing abstraction. It's high-level, elegant, declarative, flexible, composable, intuitive, and powerful. ORMs which attempt to abstract over SQL rarely capture all of these qualities, and usually the result is an underpowered, inflexible, leaky API that gives users the ability to perform only a small fraction of the queries that they could more easily and concisely express in SQL.

What I like about Diesel v1.4:
- diesel-cli is really nice, especially for authoring, running, and reverting migrations.
- The derive macros `diesel::Queryable`, `diesel::QueryableByName`, `diesel::Insertable`, and `diesel::AsChangeSet` are pretty nice.

Where I think Diesel v1.4 could improve:
- More guides would be nice, I think it's a bit strange that associations are not covered in any guide and the only way to learn about that feature is to stumble across it in the API docs.
- More logging would be nice, if I set the log level to `TRACE` I see nothing from Diesel.
- Something like a `diesel_contrib` crate, similar to how Rocket has `rocket` for core stuff and `rocket_contrib` for commonly requested nice-to-haves, would be great. It was not fun having to find and use an unofficial 3rd-party crate just to map DB enums to Rust enums.
- Almost all popular Rust libraries use macros, but Diesel especially seems to use a ton. It's still unclear to me why there needs to be a `diesel::Queryable` and a `diesel::QueryableByName` macro, as those seem like they can be consolidated. Also, if the generated Diesel schema file is not suppose to be edited by hand, why does it use macros at all? Why not just generate the code the macros would generate in the first place?
- Diesel overall felt underwhelming compared to feature-rich and batteries-included ORMs available in other languages. I think calling Diesel an "ORM" probably falsely expectations for a lot of users, or at least it did for me. I've seen similar libraries in other languages call themselves "micro ORMs" or "query builders" which I think would be much more appropriate descriptions for Diesel.
- No support for async/await and because the implementation seems to be blocked by some Rust compiler bug that nobody is interested in fixing it seems like async/await is not going to come to Diesel any time soon.

What I like about sqlx v0.4:
- Lets me just write and run SQL, which is what I want to do in the first place anyway.
- The derive macros `sqlx::FromRow` and `sqlx::Type` are really nice.
- Logs every executed query, how many rows it returned, and how long the query took at an `INFO` level. Very nice!

Where I think sqlx v0.4 can improve:
- More documentation please! The docs on sqlx-cli especially are almost nonexistent.
- Please add all of diesel-cli's migration-related functionality to sqlx-cli, including the ability to revert migrations.
- Derive macros similar to `diesel::Insertable` and `diesel::AsChangeSet` which I could use to decorate structs and then pass those structs as-is to the `bind` method of parameterized queries would be pretty nice.

The benchmarks we performed above were probably skewed a lot by Rocket and actix-web, so it's not really fair to use them to compare Diesel and sqlx. If you would like to see the results of detailed benchmarks run specifically to profile Diesel and sqlx then you can find [the results for those here](https://github.com/diesel-rs/metrics/) and [the source code for them here](https://github.com/diesel-rs/diesel/tree/master/diesel_bench). Disclaimer: this benchmark suite is maintained by the Diesel team.



### Rocket vs actix-web

What I like about Rocket v0.4:
- Super good DX (developer experience)!
- Amazing documentation!
- Amazing logs!
- Procedural macros check that all path and data parameters are used in the request handler function!
- All of the guard-related traits are great: `FromRequest`, `FromParam`, `FromData`, and so on.

Where I think Rocket v0.4 can improve:
- Please get off nightly Rust and use stable Rust. (Note: this is coming in Rocket v0.5)
- Please support async/await. (Note: this is coming in Rocket v0.5)

What I like about actix-web v3.3:
- Good documentation.
- Using procedural macros to decorate request handlers is optional and there's a non-macro API.

Where I think actix-web v3.3 can improve:
- Providing a `Responder` impl for `()` that returns `204 No Content` would be really nice.
- Providing a `ResponseError` impl for `E where E: std::error::Error` that returns `500 Server Error` and prints the error message on debug builds and prints a generic error message on release builds would be really nice.
- Please log more, like way more! Even when I set the logging level to `TRACE` the logs were still almost useless when it came to helping me debug issues.
- On one hand, the `FromRequest` impl on `Path<T> where T: DeserializeOwned` is lowkey brilliant, but on the other hand if the type is not trivially deserializable (e.g. any situation where a member has to be validated) then it's hostile to users who have never written a `serde::Deserialize` impl by hand before, which is most users.



### Sync Rust vs Async Rust

We didn't really get into "advanced" async programming in this article, and in a way that's a good thing! One of the big selling points of introducing the `async` and `await` keywords to Rust is that it would make async programming as simple and straight-forward as sync programming, and I believe they delivered on that promise in this project given how similar the async implementation was to the sync implementation.

Although with that said, while async Rust _can be_ as simple as sync Rust, it also can be way more complicated! I wasn't completely forthcoming or transparent about all of my struggles in this article, but writing the `actix_web::FromRequest` impl for `Token` was really, really hard. There were also some things I tried to do in the async implementation that never made it into the article because I just couldn't get them to compile. While I believe this is _partially_ due to my own inexperience with async programming Rust, I also think the async stuff is just inherently harder and more complex. I'd like to tackle all of these things head-on so I'll probably write an _"Advanced Async Patterns in Rust"_ article or something like that in the future.



### Rust vs JS

I noticed I _feel_ very productive in Rust because I'm making a lot of decisions, but most of the time most of the decisions are unrelated to solving the problem at hand, so my _actual_ progress and productivity is kinda low. I'm well past the newbie stage so I rarely have issues with lifetimes or borrowing anymore, but like I mentioned above: working with futures and async Rust can still be really challenging.

The one big thing I missed from Rust when re-writing the RESTful API servers in JS was the static type checking, but that's not really an argument for Rust, it's more of an argument for TypeScript. One of my personal takeaways from this project is that I should probably learn TypeScript. 



### In Summary

In the future, if I find myself writing another RESTful API server in Rust, I'm definitely going to use sqlx over Diesel, but this is to satisfy my own personal preferences, I don't think Diesel is a bad choice at all for those would prefer using an ORM over SQL.

If I had to pick a web framework right now I'd probably pick Rocket, because although actix-web seems to have better performance Rocket wins in almost every other possible metric: documentation, logging, easy-to-use APIs, and overall developer friendliness. I'm eagerly awaiting the release of Rocket v0.5, which should fix all the issues I have with Rocket v0.4, including improving the performance which hopefully will put it on par with actix-web.



## Discuss

Discuss this article on
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/nanar9/restful_api_in_sync_async_rust/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-restful-api-in-sync-async-rust/59713)
- [Twitter](https://twitter.com/pretzelhammer/status/1392825891428909062)
- [rust subreddit](https://www.reddit.com/r/rust/comments/nnp6j4/restful_api_in_sync_async_rust/)



## Notifications

Get notified when the next article get published by
- [Following pretzelhammer on Twitter](https://twitter.com/pretzelhammer) or
- [Subscribing to this repo's release RSS feed](https://github.com/pretzelhammer/rust-blog/releases.atom) or
- Watching this repo's releases (click `Watch` -> click `Custom` -> select `Releases` -> click `Apply`)



## Further Reading

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Sizedness in Rust](./sizedness-in-rust.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Learning Rust in 2020](./learning-rust-in-2020.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)
