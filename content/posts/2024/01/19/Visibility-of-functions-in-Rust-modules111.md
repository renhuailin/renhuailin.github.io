---
layout: article
title: 搞懂Rust模块中函数的可见性
date: 2024-01-19 09:09:09
author: 任怀林
tags:
- Rust
thumb: linux.png
comment: true
isCJKLanguage: true
---
今天在看Sea ORM的例子时，看到一段代码：
```rust
use sea_orm::*;

 const DATABASE_URL: &str = "mysql://root:root@localhost:3306";
 const DB_NAME: &str = "bakeries_db";

 pub(super) async fn set_up_db() -> Result<DatabaseConnection, DbErr> {
     let db = Database::connect(DATABASE_URL).await?;

     let db = match db.get_database_backend() {
         DbBackend::MySql => {
             db.execute(Statement::from_string(
                 db.get_database_backend(),
                 format!("CREATE DATABASE IF NOT EXISTS `{}`;", DB_NAME),
             ))
             .await?;

             let url = format!("{}/{}", DATABASE_URL, DB_NAME);
             Database::connect(&url).await?
         }
         DbBackend::Postgres => {
             db.execute(Statement::from_string(
                 db.get_database_backend(),
                 format!("DROP DATABASE IF EXISTS \"{}\";", DB_NAME),
             ))
             .await?;
             db.execute(Statement::from_string(
                 db.get_database_backend(),
                 format!("CREATE DATABASE \"{}\";", DB_NAME),
             ))
             .await?;

             let url = format!("{}/{}", DATABASE_URL, DB_NAME);
             Database::connect(&url).await?
         }
         DbBackend::Sqlite => db,
     };

     Ok(db)
 }
```

我对函数前面的` pub(super) `有点不明白，猜想可能跟java的`public`差不多，于是去查Rust Book，结果里面没有。

在`Rust语言圣经`里找到了相关的章节说明，[受限可见性](https://course.rs/basic/crate-module/use.html#%E5%8F%97%E9%99%90%E7%9A%84%E5%8F%AF%E8%A7%81%E6%80%A7) 

**限制可见性语法**

`pub(crate)` 或 `pub(in crate::a)` 就是限制可见性语法，前者是限制在整个包内可见，后者是通过绝对路径，限制在包内的某个模块内可见，总结一下：

- `pub` 意味着可见性无任何限制
- `pub(crate)` 表示在当前包可见
- `pub(self)` 在当前模块可见
- `pub(super)` 在父模块可见
- `pub(in <path>)` 表示在某个路径代表的模块中可见，其中 `path` 必须是父模块或者祖先模块

为了理解` pub(super) `,我写了下面的测试代码。

```rust
mod parent_module {
        use self::child_module::say_hi;

        // 这是子模块
        pub mod child_module {
            // 这是在子模块中的函数
            pub(super) fn say_hi() {
                println!("Hello world!")
            }

            #[test]
            fn test3() {
                say_hi();
            }
        }

        mod child2_module {
            use super::child_module::say_hi;

            #[test]
            fn test2() {
                say_hi();
            }
        }

        #[test]
        fn test1() {
            say_hi();
        }
    }

    mod parent_module2 {
        // use super::parent_module::child_module::say_hi;
        // #[test]
        // fn test1() {
        //     say_hi();
        // }
    }
}
```