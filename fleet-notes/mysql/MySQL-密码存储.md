---
title: MySQL-密码存储
tags:
  - fleet-note
  - middleware/database/mysql
date: 2024-12-03
time: 22:37
aliases:
---
#### 账户密码存储设计

切记，在数据库表结构设计时，千万不要直接在数据库表中直接存储密码，一旦有恶意用户进入到系统，则面临用户数据泄露的极大风险。比如金融行业，从合规性角度看，所有用户隐私字段都需要加密，甚至业务自己都无法知道用户存储的信息（隐私数据如登录密码、手机、信用卡信息等）。

相信不少开发开发同学会通过函数 MD5 加密存储隐私数据，这没有错，因为 MD5 算法并不可逆。然而，MD5 加密后的值是固定的，如密码 12345678，它对应的 MD5 固定值即为 25d55ad283aa400af464c76d713c07ad。

因此，可以对 MD5 进行暴力破解，计算出所有可能的字符串对应的 MD5 值。若无法枚举所有的字符串组合，那可以计算一些常见的密码，如111111、12345678 等。我放在文稿中的这个[网站](https://www.md5online.org/?fileGuid=xxQTRXtVcqtHK6j8)，可用于在线解密 MD5 加密后的字符串。

所以，在设计密码存储使用，还需要加盐（salt），每个公司的盐值都是不同的，因此计算出的值也是不同的。若盐值为 psalt，则密码 12345678 在数据库中的值为：

```ini
password = MD5（‘psalt12345678’）
```

这样的密码存储设计是一种固定盐值的加密算法，其中存在三个主要问题：

- 若 salt 值被（离职）员工泄漏，则外部黑客依然存在暴利破解的可能性；
- 对于相同密码，其密码存储值相同，一旦一个用户密码泄漏，其他相同密码的用户的密码也将被泄漏；
- 固定使用 MD5 加密算法，一旦 MD5 算法被破解，则影响很大。

所以一个真正好的密码存储设计，应该是：**动态盐 + 非固定加密算法**。

我比较推荐这么设计密码，列 password 存储的格式如下：

```swift
$salt$cryption_algorithm$value
```

其中：

- $salt：表示动态盐，每次用户注册时业务产生不同的盐值，并存储在数据库中。若做得再精细一点，可以动态盐值 + 用户注册日期合并为一个更为动态的盐值。
- $cryption_algorithm：表示加密的算法，如 v1 表示 MD5 加密算法，v2 表示 AES256 加密算法，v3 表示 AES512 加密算法等。
- $value：表示加密后的字符串。

这时表 User 的结构设计如下所示：

```markdown
CREATE TABLE User (

    id BIGINT NOT NULL AUTO_INCREMENT,

    name VARCHAR(255) NOT NULL,

    sex CHAR(1) NOT NULL,

    password VARCHAR(1024) NOT NULL,

    regDate DATETIME NOT NULL,

    CHECK (sex = 'M' OR sex = 'F'),

    PRIMARY KEY(id)

);

SELECT * FROM User\G

*************************** 1. row ***************************

      id: 1

    name: David

     sex: M

password: $fgfaef$v1$2198687f6db06c9d1b31a030ba1ef074

 regDate: 2020-09-07 15:30:00

*************************** 2. row ***************************

      id: 2

    name: Amy

     sex: F

password: $zpelf$v2$0x860E4E3B2AA4005D8EE9B7653409C4B133AF77AEF53B815D31426EC6EF78D882

 regDate: 2020-09-07 17:28:00
```

在上面的例子中，用户 David 和 Amy 密码都是 12345678，然而由于使用了动态盐和动态加密算法，两者存储的内容完全不同。

即便别有用心的用户拿到当前密码加密算法，则通过加密算法 $cryption_algorithm 版本，可以对用户存储的密码进行升级，进一步做好对于恶意数据攻击的防范。


# Reference