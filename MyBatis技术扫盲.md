------

### 一、ResultMap

MyBatis 中的 `<resultMap>` 元素是其最强大和灵活的功能之一，它的主要作用是**定义数据库查询结果如何映射到 Java 对象（POJO）**。

简单来说，当 MyBatis 执行一个 SQL 查询并从数据库获取到结果集（ResultSet）后，它需要知道如何把 ResultSet 中的每一列数据，准确地填充到你 Java 对象（比如 `SysDictData`）的对应属性中去。`<resultMap>` 就是用来告诉 MyBatis 这个映射规则的。

示例：

```xml
<resultMap type="SysDictData" id="SysDictDataResult">
    <id     property="dictCode"   column="dict_code"   />
    <result property="dictSort"   column="dict_sort"   />
    <result property="dictLabel"  column="dict_label"  />
    <result property="dictValue"  column="dict_value"  />
    <result property="dictType"   column="dict_type"   />
    <result property="cssClass"   column="css_class"   />
    <result property="listClass"  column="list_class"  />
    <result property="isDefault"  column="is_default"  />
    <result property="status"     column="status"      />
    <result property="createBy"   column="create_by"   />
    <result property="createTime" column="create_time" />
    <result property="updateBy"   column="update_by"   />
    <result property="updateTime" column="update_time" />
</resultMap>
```

- **`type="SysDictData"`**: 指定了这个 `resultMap` 会将查询结果映射到 `SysDictData` 这个 Java 类。`SysDictData` 必须是完整的类名（如果不在同一个包下，需要写全限定名，但通常会通过 `<configuration>` 中的 `<typeAliases>` 配置简化）。
- **`id="SysDictDataResult"`**: 为这个 `resultMap` 定义了一个唯一的标识符。在 `<select>` 标签中，你可以通过 `resultMap="SysDictDataResult"` 来引用它。
- **`<id property="dictCode" column="dict_code" />`**:
  - `id` 标签表示这是一个主键列的映射。
  - `property="dictCode"`: 对应 `SysDictData` 类中的 `dictCode` 属性。
  - `column="dict_code"`: 对应数据库查询结果中的 `dict_code` 列。
  - 这意味着，数据库返回的 `dict_code` 列的值，会被赋值给 `SysDictData` 对象的 `dictCode` 属性。
- **`<result property="dictSort" column="dict_sort" />`**:
  - `result` 标签表示这是一个非主键列的映射。
  - `property="dictSort"`: 对应 `SysDictData` 类中的 `dictSort` 属性。
  - `column="dict_sort"`: 对应数据库查询结果中的 `dict_sort` 列。
  - 其余的 `<result>` 标签都遵循同样的模式，将数据库列名（下划线命名）映射到 Java 对象的属性名（驼峰命名）。

##### 总结：

总而言之，`<resultMap>` 就是 MyBatis 中连接数据库表列和 Java 对象属性的“桥梁”和“翻译器”。它让你可以精确地控制数据从数据库到 Java 对象的映射过程，尤其是在列名和属性名不一致或存在复杂关联关系时，它扮演着不可或缺的角色。



### 二、< sql >标签

MyBatis 中的 `<sql>` 标签是一个非常实用的元素，它的主要作用是**定义可重用的 SQL 片段**。

简单来说，如果你在多个 SQL 查询语句中发现有重复的 SQL 代码（比如相同的 `SELECT` 字段列表，或者相同的 `WHERE` 条件片段），你就可以把这些重复的部分提取出来，定义在一个 `<sql>` 标签里。然后，在需要使用这些 SQL 片段的地方，通过 `<include>` 标签来引用它。

##### 如何使用 `<sql>` 标签？

`<sql>` 标签通常包含一个 `id` 属性，用于唯一标识这个 SQL 片段。然后在其他 SQL 语句中，通过 `<include refid="your_sql_id"/>` 来引用它。

**示例：**

假设你有一个 `user` 表，经常需要查询 `id, username, email` 这三个字段。

XML

```xml
<mapper namespace="com.example.mapper.UserMapper">

    <sql id="userColumns">
        id,
        username,
        email
    </sql>

    <sql id="commonUserWhere">
        <where>
            <if test="id != null">
                AND id = #{id}
            </if>
            <if test="username != null and username != ''">
                AND username LIKE CONCAT('%', #{username}, '%')
            </if>
        </where>
    </sql>

    <select id="selectUserById" resultType="com.example.model.User">
        SELECT
        <include refid="userColumns"/>  FROM user
        WHERE id = #{id}
    </select>

    <select id="selectUsersByConditions" resultType="com.example.model.User">
        SELECT
        <include refid="userColumns"/>  FROM user
        <include refid="commonUserWhere"/> </select>

    <update id="updateUser">
        UPDATE user
        SET
            email = #{email}
        <include refid="commonUserWhere"/> </update>

</mapper>
```

在这个例子中：

- `userColumns` 这个 `id` 的 `<sql>` 片段包含了常用的用户字段。
- `commonUserWhere` 这个 `id` 的 `<sql>` 片段包含了动态的用户查询条件。
- 在 `selectUserById` 和 `selectUsersByConditions` 查询中，都通过 `<include refid="userColumns"/>` 引用了字段列表，避免了重复编写 `id, username, email`。
- 在 `selectUsersByConditions` 和 `updateUser` 中，都通过 `<include refid="commonUserWhere"/>` 引用了动态的 WHERE 条件，使得条件逻辑得以复用。





### 三、动态 SQL 标签

```xml
<sql id="commonUserWhere">

<where>

<if test="id != null">

AND id = #{id}

</if>

<if test="username != null and username != ''">

AND username LIKE CONCAT('%', #{username}, '%')

</if>

</where>

</sql>
```

##### 这段代码的整体含义和工作流程：

这段 `<sql>` 片段定义了一个灵活的查询条件。当它被 `<include>` 引用时，MyBatis 会这样做：

1. **检查 `id` 参数：** 如果传入的参数对象（通常是方法参数或 Map）中，`id` 属性的值不为 `null`，那么生成的 SQL 中就会包含 `AND id = [id的值]`。
2. **检查 `username` 参数：** 如果传入参数中的 `username` 属性的值不为 `null` 且不为空字符串，那么生成的 SQL 中就会包含 `AND username LIKE '%[username的值]%`。
3. **智能拼接 `WHERE`：**
   - 如果 `id` 和 `username` 都为 `null` 或空，那么 `<where>` 标签内部不会生成任何 SQL，最终 SQL 中也就**不会出现 `WHERE` 关键字**。
   - 如果只有 `id` 不为 `null`，生成的 SQL 片段将是 `WHERE id = [id的值]` (`<where>` 移除了开头的 `AND`)。
   - 如果只有 `username` 不为 `null` 且不为空，生成的 SQL 片段将是 `WHERE username LIKE '%[username的值]%` (`<where>` 移除了开头的 `AND`)。
   - 如果 `id` 和 `username` 都不为 `null`/空，生成的 SQL 片段将是 `WHERE id = [id的值] AND username LIKE '%[username的值]%`。



##### `<where>` 标签

`<where>` 是 MyBatis 提供的一个非常有用的**动态 SQL 标签**。它的主要作用是**智能地处理 `WHERE` 子句的拼接问题**。

- **智能添加 `WHERE` 关键字：** 如果 `<where>` 内部的条件最终有任何一个被评估为 `true`（即生成了 SQL 片段），那么 `<where>` 标签会自动在生成 SQL 的最前面加上 `WHERE` 关键字。

- **智能移除多余的 `AND` 或 `OR`：** 如果 `<where>` 内部的第一个条件以 `AND` 或 `OR` 开头，`<where>` 标签会自动移除这个多余的 `AND` 或 `OR`。例如，如果 `id != null` 为真，`AND id = #{id}` 会生成，`<where>` 会自动把开头的 `AND` 移除，最终只剩下 `id = #{id}`。

  

##### `<if test="表达式">` 标签

`<if>` 标签也是 MyBatis 的一个核心**动态 SQL 标签**，它用于根据一个条件（`test` 属性中的表达式）来决定是否包含其内部的 SQL 片段。

- **`test` 属性：** 这里的 `test` 属性接收一个 OGNL (Object-Graph Navigation Language) 表达式。MyBatis 会评估这个表达式，如果表达式的结果为 `true`，那么 `<if>` 标签内部的 SQL 片段就会被包含到最终生成的 SQL 中；如果为 `false`，则忽略。
- **示例中的表达式：**
  - `id != null`: 判断传入参数中的 `id` 属性是否不为 `null`。
  - `username != null and username != ''`: 判断传入参数中的 `username` 属性是否不为 `null` 且不为空字符串。
- **SQL 片段：** `<if>` 标签内部的 SQL 片段会作为条件的一部分。
  - `AND id = #{id}`: 如果 `id` 不为 `null`，这个片段就会被添加到 `WHERE` 子句中。`#{id}` 是 MyBatis 的参数占位符，它会从传入的参数对象中获取 `id` 的值。
  - `AND username LIKE CONCAT('%', #{username}, '%')`: 如果 `username` 不为 `null` 且不为空字符串，这个片段会被添加。`CONCAT('%', #{username}, '%')` 是 SQL 的函数，用于实现模糊查询。



### 四、`CONCAT` 函数的用法

`CONCAT` 是一个 SQL 函数，用于**连接（拼接）两个或多个字符串**。

- **基本语法：** `CONCAT(string1, string2, string3, ...)`
- **作用：** 它会将所有传入的字符串参数连接起来，形成一个新的字符串。
- **示例：**
  - `SELECT CONCAT('Hello', ' ', 'World');` 结果是 `'Hello World'`
  - `SELECT CONCAT('User', 123);` 结果是 `'User123'` (数字会被隐式转换为字符串)

##### `CONCAT('%', #{username}, '%')` 的含义

现在我们来解释 `CONCAT('%', #{username}, '%')`：

1. **`%` (百分号)：** 在 SQL 的 `LIKE` 操作符中，`%` 是一个**通配符**，代表零个、一个或多个任意字符。
   - `'abc%'`：匹配以 "abc" 开头的所有字符串。
   - `'%abc'`：匹配以 "abc" 结尾的所有字符串。
   - `'%abc%'`：匹配包含 "abc" 的所有字符串。
2. **`#{username}`：** 这是 MyBatis 的参数占位符。当 MyBatis 执行 SQL 语句时，它会从你的 Java 代码中获取 `username` 参数的值，并将其安全地替换到这个位置。MyBatis 会自动处理参数的类型和 SQL 注入问题。
3. **`CONCAT('%', #{username}, '%')` 组合起来：** 这个表达式的目的是构造一个用于 `LIKE` 模糊查询的字符串。
   - 假设 `#{username}` 的值为 `'张三'`。
   - 那么 `CONCAT('%', '张三', '%')` 的结果就是字符串 `'%张三%'`。

------

##### 在 `LIKE` 子句中的应用

当这个构造出来的字符串用于 `LIKE` 子句时，它的效果就是：

```
WHERE 列名 LIKE CONCAT('%', #{username}, '%')
```

- **实际执行的 SQL (以 `username = '张三'` 为例)：** `WHERE 列名 LIKE '%张三%'`
- **含义：** 查询 `列名` 中包含 `张三` 这个字符串的所有记录。



##### 为什么要用 `CONCAT` 而不是直接拼接？

在某些情况下，你可能会看到有人这样写模糊查询： `WHERE username LIKE '%' + #{username} + '%'` (在 SQL Server 中) 或者 `WHERE username LIKE '%' || #{username} || '%'` (在 Oracle 中)

但在 **MySQL** 和 **MyBatis 的通用实践**中，使用 `CONCAT` 函数有以下好处：

1. **数据库兼容性更好：** `CONCAT` 是 SQL 标准的一部分，大多数主流数据库都支持它（虽然不同数据库可能有细微语法差异，但 `CONCAT(str1, str2, ...)` 是非常普遍的）。而 `+` 或 `||` 作为字符串连接符在不同数据库中行为可能不同。
2. **MyBatis 的推荐方式：** 在 MyBatis 的 XML Mapper 文件中，使用 `CONCAT` 函数来拼接字符串是明确且推荐的做法，它能确保 SQL 语句在不同数据库方言下的良好兼容性（尤其是在你可能切换数据库时）。
3. **避免 SQL 注入风险：** 无论是直接在 Java 代码中拼接字符串然后传给 MyBatis，还是在 MyBatis XML 中使用 `${}` 这种非预编译占位符来拼接，都存在 SQL 注入的风险。而 `CONCAT('%', #{username}, '%')` 这种方式，MyBatis 会先将 `#{username}` 预编译处理，再将结果传递给 `CONCAT` 函数，从而有效避免了 SQL 注入。

所以，`CONCAT('%', #{username}, '%')` 的意义就是**安全地构建一个用于模糊查询的字符串，查询数据库中包含某个关键字的所有记录。**





### 五、`<foreach>` 标签

`<foreach>` 标签允许你构建动态的 SQL `IN` 子句。

```xml
<delete id="deleteDictDataByIds" parameterType="Long">
    delete from sys_dict_data where dict_code in
    <foreach collection="array" item="dictCode" open="(" separator="," close=")">
       #{dictCode}
      </foreach>
</delete>
```

1. **`<delete id="deleteDictDataByIds" parameterType="Long">`**
   - `delete`: 表示这是一个删除操作的 SQL 语句。
   - `id="deleteDictDataByIds"`: 这个删除操作的唯一标识符。在 Mapper 接口中，你会有一个同名的方法来调用它。
   - `parameterType="Long"`: 这个属性在这里有点**特殊**。通常，`parameterType` 会指定传入的单个参数的类型，或者一个包含多个参数的 Map/POJO。然而，当与 `<foreach collection="array">` 结合使用时，`parameterType` 实际上指的是数组中元素的类型（或者说，MyBatis 期望接收一个 `Long` 类型的数组）。**更准确地说，当传入的是原始类型数组（如 `long[]` 或 `Long[]`）时，MyBatis 会将其识别为 `collection="array"`。**
2. **`delete from sys_dict_data where dict_code in`**
   - 这是 SQL 删除语句的固定部分。它表示要从 `sys_dict_data` 表中删除记录，并且删除的条件是 `dict_code` 字段的值在某个集合中。
3. **`<foreach>` 标签** 这是这段代码的核心，用于构建 SQL 的 `IN` 子句。
   - **`collection="array"`**:
     - 这个属性指定了要遍历的集合或数组的名称。
     - 当你的 Mapper 方法接收一个**数组**（例如 `Long[] dictCodes` 或 `long[] dictCodes`）作为参数时，MyBatis 内部会将这个数组包装成一个 Map，其中数组的键名就是 `"array"`。
     - 所以，`collection="array"` 就是指遍历传入的那个数组参数。
     - **注意：** 如果你的 Mapper 方法接收的是 `List`，那么 `collection` 应该设置为 `"list"`。如果接收的是 `Set`，则设置为 `"set"`。如果接收的是一个 Map，并且要遍历 Map 中的某个键对应的值，则设置为该键名。如果只有一个参数且是简单类型或POJO，MyBatis会将其作为默认值，此时`collection`可以省略或使用其属性名。
   - **`item="dictCode"`**:
     - 这个属性指定了在每次迭代中，当前遍历到的元素将被赋值给哪个变量名。
     - 在这里，每次从 `array` 中取出一个元素，都会将其值赋给 `dictCode` 这个临时变量。你可以在 `<foreach>` 标签内部通过 `#{dictCode}` 来引用这个变量。
   - **`open="("`**:
     - 这个属性指定了在整个 `<foreach>` 循环生成的 SQL 片段的**最前面**添加的字符串。
     - 在这里，它会生成 `IN (`。
   - **`separator=","`**:
     - 这个属性指定了在每次迭代生成的 SQL 片段**之间**添加的字符串。
     - 在这里，它会在每个 `#{dictCode}` 之间添加逗号 `,`。
   - **`close=")`"**:
     - 这个属性指定了在整个 `<foreach>` 循环生成的 SQL 片段的**最后面**添加的字符串。
     - 在这里，它会生成 `)`。
   - **`#{dictCode}`**:
     - 这是每次迭代实际生成的 SQL 片段。它会取出当前 `item` 变量 `dictCode` 的值，并作为参数插入到 SQL 中。





### 六、`<set>` 标签

`<set>` 标签是 MyBatis 提供的一个**动态 SQL 标签**，主要用于**智能地处理 `UPDATE` 语句中的 `SET` 子句**。

**它的主要优势在于：**

1. **自动添加 `SET` 关键字：** 如果 `<set>` 内部有任何一个条件被评估为 `true`（即生成了 SQL 片段），那么 `<set>` 标签会自动在生成的 SQL 最前面加上 `SET` 关键字。
2. **智能移除多余的逗号 (`,`)：** 这是 `<set>` 最重要的功能。如果 `<set>` 内部的任何一个 `if` 语句在生成的 SQL 片段中以逗号 `,` 结尾，`<set>` 标签会自动移除最后一个多余的逗号。







### 七、TRUNCATE TABLE` vs `DELETE FROM

`TRUNCATE TABLE` 语句的作用是 **清空表中的所有数据，但保留表的结构（即不删除表本身）**。

可以把它理解为“快速清空”或“重置”一张表。

| 特性             | `TRUNCATE TABLE table_name;`                                 | `DELETE FROM table_name;`                          |
| ---------------- | ------------------------------------------------------------ | -------------------------------------------------- |
| **删除内容**     | **清空表中的所有数据，保留表结构。**                         | **删除表中的所有数据（如果无 `WHERE` 子句）。**    |
| **DML/DDL**      | 属于 **DDL** (数据定义语言) 操作。                           | 属于 **DML** (数据操作语言) 操作。                 |
| **执行速度**     | **非常快**，因为它直接截断或重新初始化数据文件，不记录每一行的删除。 | **相对较慢**，因为它逐行删除数据，并记录事务日志。 |
| **事务日志**     | 极少记录事务日志，所以不能回滚 (ROLLBACK)。                  | 记录详细的事务日志，所以可以回滚 (ROLLBACK)。      |
| **`WHERE` 子句** | **不能**使用 `WHERE` 子句来删除部分数据。                    | **可以**使用 `WHERE` 子句来删除满足条件的数据。    |
| **自增 ID**      | 通常会**重置**自增（AUTO_INCREMENT）列的计数器，从 1 或初始值重新开始。 | 不会重置自增 ID，新的插入会接着之前的最大值继续。  |
| **触发器**       | 不会触发 `DELETE` 触发器。                                   | 会触发 `DELETE` 触发器。                           |







### 八、CAST() 与 IFNULL()

`CAST()` 和 `IFNULL()` 都是 SQL 中非常实用的函数，用于数据类型转换和空值处理。



`CAST()` 函数用于将一个表达式从一种数据类型转换为另一种数据类型。这在很多场景下都非常有用，比如当你需要进行类型匹配的比较、排序，或者将数据以特定格式展示时。

```
CAST(expression AS data_type)
```

```sql
-- 假设 current_timestamp 是 '2025-07-23 14:30:00'
SELECT CAST(CURRENT_TIMESTAMP AS DATE);
-- 结果: '2025-07-23'
```



`IFNULL()` 函数用于处理 `NULL` 值。当某个表达式的结果为 `NULL` 时，你可以指定一个替代值；如果表达式不为 `NULL`，则返回表达式本身的值。

```
IFNULL(expression1, expression2)
```

- **`expression1`**: 你要检查的原始表达式。
- **`expression2`**: 如果 `expression1` 为 `NULL`，则返回的替代值。

##### 示例

**替换列中的 `NULL` 值：** 假设你有一个 `price` 列，其中有些商品的价格是 `NULL`，你想把 `NULL` 显示为 `0.00`。

```
SELECT product_name, IFNULL(price, 0.00) AS actual_price
FROM products;
```



**注意： IFNULL 似乎可以被另一个函数-- COALESCE替换，语法格式相同。**





### 九、连续左外连接LEFT  JOIN问题

**Q:tableA left join tableB on... left join tableC on ...  我想知道这种多个表左外连接时，MySQL的处理逻辑是怎样的，比如那些不符合on连接条件但在A表或B表中的行会被算入结果集吗？** 

**A:**

------

当你在 MySQL 中进行多个表的 `LEFT JOIN` 时，处理逻辑是**从左到右逐个连接**。这意味着连接会**保留左侧表的所有行**，即使右侧表没有匹配项。

------

##### MySQL 多个 `LEFT JOIN` 的处理逻辑

我们用一个例子来说明：`TableA LEFT JOIN TableB ON ... LEFT JOIN TableC ON ...`

1. **首先处理 `TableA LEFT JOIN TableB ON (join_condition_AB)`：**
   - MySQL 会从 `TableA` 中取出每一行。
   - 对于 `TableA` 的每一行，它会尝试在 `TableB` 中寻找满足 `join_condition_AB` 的匹配行。
   - **如果找到匹配项：** `TableA` 的行会与 `TableB` 的匹配行组合，形成一个“中间结果集”。
   - **如果没找到匹配项：** `TableA` 的行仍然会被包含在这个“中间结果集”中，但 `TableB` 的所有列在该行中都将是 `NULL`。
   - 这个步骤的结果，是一个包含 `TableA` 所有行（无论是否在 `TableB` 中找到匹配）的扩展表。
2. **然后处理 `(中间结果集) LEFT JOIN TableC ON (join_condition_BC)`：**
   - MySQL 会将上一步生成的“中间结果集”视为新的左表。
   - 对于“中间结果集”中的每一行，它会尝试在 `TableC` 中寻找满足 `join_condition_BC` 的匹配行。
   - **如果找到匹配项：** “中间结果集”的行会与 `TableC` 的匹配行组合，形成最终的结果集。
   - **如果没找到匹配项：** “中间结果集”的行仍然会被包含在最终的结果集中，但 `TableC` 的所有列在该行中都将是 `NULL`。

------

##### 关于“不符合 ON 连接条件但在 A 表或 B 表中的行”是否算入结果集

是的，它们会被算入结果集，但这需要具体分析“在哪一阶段”以及“哪个表”不符合条件。

**对于 `TableA`：**

- `TableA` 中的**所有行**都将出现在最终结果集中，因为它是最左边的表。
- 如果 `TableA` 的某行在 `TableB` 中没有匹配（根据 `join_condition_AB`），那么该行的 `TableB` 列将为 `NULL`。
- 如果 `TableA` 的某行在与 `TableB` 组合后，形成的中间结果在 `TableC` 中没有匹配（根据 `join_condition_BC`），那么该行的 `TableC` 列将为 `NULL`。

**对于 `TableB`：**
- `TableB` 中的行**只有在满足 `join_condition_AB` 且能与 `TableA` 匹配时**，才会被包含在“中间结果集”中。
- `TableB` 中那些**不符合 `join_condition_AB` 或者在 `TableA` 中没有匹配项的行**，**不会**出现在最终结果集中。因为 `TableB` 是 `TableA` 的右表，`LEFT JOIN` 保证的是左表的所有行都存在。

**对于 `TableC`：**
- `TableC` 中的行**只有在满足 `join_condition_BC` 且能与“中间结果集”匹配时**，才会被包含在最终结果集中。

- `TableC` 中那些**不符合 `join_condition_BC` 或者在“中间结果集”中没有匹配项的行**，**不会**出现在最终结果集中。因为 `TableC` 是“中间结果集”的右表，`LEFT JOIN` 保证的是其左表（即“中间结果集”）的所有行都存在。

  

# 合集篇

##    1.连表查询结果集映射为Java对象

使用<resultMap>标签,type为需要映射成的类。需注意该标签在<select>标签外使用，且<select>需要指定resultMap属性：

**栗子**：

```
 <resultMap id="ProductWithPublishMap" type="cn.ah.idata.datafield.platform.service.node.product.domain.resp.TinyProductResponse">
        <result property="productId" column="product_id"/>
        <result property="productName" column="product_name"/>
        <result property="productType" column="product_type"/>
        <result property="industryClassify" column="industry_classify"/>
        <result property="publishTime" column="publish_time"/>
        <result property="platformId" column="platform_id"/>
        <result property="platformName" column="platform_name"/>
        <result property="provider" column="provider"/>
        <!-- 关键部分：指定 typeHandler -->
        <result property="dataSpaces" column="data_spaces"
                typeHandler="com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler"/>
        <result property="useCount" column="use_count"/>
        <result property="measureMethod" column="measure_method"/>
        <result property="price" column="price"/>
        <result property="unit" column="unit"/>
        <result property="productRegionProvince" column="product_region_province"/>
        <result property="productRegionCity" column="product_region_city"/>
        <result property="productRegionArea" column="product_region_area"/>
        <result property="productRegionName" column="product_region_name"/>
        <result property="dataSubject" column="data_subject"/>
        <result property="description" column="description"/>
        <result property="logo" column="logo"/>
        <result property="measures" column="measures"
                typeHandler="com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler"/>
    </resultMap>

<!--    <select id="selectProductWithPublish" resultType="cn.ah.idata.datafield.platform.service.node.product.domain.resp.TinyProductResponse">-->
    <select id="selectProductWithPublish" resultMap="ProductWithPublishMap">
        SELECT
        p.product_id,
        p.product_name,
        p.product_type,
        p.industry_classify,
        p.publish_time,
        p.platform_id,
        p.platform_name,
        p.provider,
        p.data_spaces,
        p.use_count,
        p.measure_method,
        p.price,
        p.unit,
        p.measures,
        r.product_region_province,
        r.product_region_city,
        r.product_region_area,
        r.product_region_name,
        r.data_subject,
        r.description,
        r.logo
        FROM data_product_publish p
        INNER JOIN data_product_regist r
        ON p.product_id = r.product_id
        <where>

            AND p.publish_status = '1'

            <if test="productId != null and productId != ''">
                AND p.product_id = #{productId}
            </if>
            <if test="productName != null and productName != ''">
                AND p.product_name = #{productName}
            </if>
            <if test="productType != null and productType != ''">
                AND p.product_type = #{productType}
            </if>
            <if test="industry != null and industry != ''">
                AND p.industry_classify = #{industry}
            </if>
            <if test="provider != null and provider != ''">
                AND p.provider = #{provider}
            </if>
            <if test="startTime != null">
                AND p.publish_time &gt;= #{startTime}
            </if>
            <if test="endTime != null">
                AND p.publish_time &lt;= #{endTime}
            </if>
            <if test="productRegionProvince != null and productRegionProvince != ''">
                AND r.product_region_province = #{productRegionProvince}
            </if>
            <if test="productRegionCity != null and productRegionCity != ''">
                AND r.product_region_city = #{productRegionCity}
            </if>
            <if test="productRegionArea != null and productRegionArea != ''">
                AND r.product_region_area = #{productRegionArea}
            </if>
            <if test="dataSubject != null and dataSubject != ''">
                AND r.data_subject = #{dataSubject}
            </if>
            <if test="deliveryMethod != null and deliveryMethod != ''">
                AND r.delivery_method = #{deliveryMethod}
            </if>

            <!-- 新增 isFree 逻辑 -->
            <if test="isFree != null and isFree != ''">
                <choose>
                    <when test="isFree == 'true'">
                        AND p.price = 0
                    </when>
                    <when test="isFree == 'false'">
                        AND p.price &gt; 0
                    </when>
                </choose>
            </if>


        </where>

        <!-- 新增 publishTimeOrder 排序 -->
        <if test="publishTimeOrder != null and publishTimeOrder != ''">
            ORDER BY p.publish_time
            <choose>
                <when test="publishTimeOrder == 'ASC'">ASC</when>
                <when test="publishTimeOrder == 'DESC'">DESC</when>
            </choose>
        </if>
    </select>
```

## 2记录映射字段为 JSON --> Java对象 

如果使用MyBatis在xml写sql语句,需要使用

```
<result property="dataSpaces" column="data_spaces"
        typeHandler="com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler"/>
```

## 3动态SQL标签

参考上面的栗子。需要注意的是：①<where>标签下可以直接硬写AND，`<where>` 标签会自动帮你去掉第一个 `AND` 或 `OR`，所以放心直接写。②测试的条件列可以来源于mapper方法的参数对象的字段，mp会自动处理。

## 4IF标签嵌套

```xml
<!-- 先根据req.providerName是否有值判断，如果没有值则跳过；如果有值则进一步判断 -->
<if test="req.providerName != null and req.providerName != ''">
    <if test="req.providers != null and req.providers.size > 0">
        AND p.provider IN
        <foreach collection="req.providers" item="prov" open="(" separator="," close=")">
            #{prov}
        </foreach>
    </if>
    <if test="req.providers == null or req.providers.size == 0">
        AND 1 = 0   <!-- providerName有值但没匹配，强制查空 -->
    </if>
</if>
```

**注意技巧： AND 1 = 0 强制查空**

附：这里之所以有两个子if，是因为req.providers集合为空时， IN子句会变成IN()，导致sql语句报错，所以需额外处理集合为空的状况。

## 5`<foreach>`集合遍历查询

```
 <if test="req.providers != null and req.providers.size > 0">
        AND p.provider IN
        <foreach collection="req.providers" item="prov" open="(" separator="," close=")">
            #{prov}
        </foreach>
 </if>
```

注意点同上。

## 6.`<choose>` `<when>` `<otherwise>`

这里以一个排序的例子说明：

```xml
<choose>
    <!-- 前端指定了时间排序 -->
    <when test="req.publishTimeOrder != null and req.publishTimeOrder != ''">
        ORDER BY p.publish_time
        <choose>
            <when test="req.publishTimeOrder == 'ASC'">ASC</when>
            <when test="req.publishTimeOrder == 'DESC'">DESC</when>
        </choose>
    </when>

    <!-- 前端指定了访问量排序 -->
    <when test="req.accessCountOrder != null and req.accessCountOrder != ''">
        ORDER BY p.visit_count
        <choose>
            <when test="req.accessCountOrder == 'ASC'">ASC</when>
            <when test="req.accessCountOrder == 'DESC'">DESC</when>
        </choose>
    </when>

    <!-- 默认排序 -->
    <otherwise>
        ORDER BY p.publish_time DESC
    </otherwise>
</choose>
```

## 7.Mybatis if 判断等于一个字符串--OGNL表达式解析

引用[Mybatis if 判断等于一个字符串 - wjj1013 - 博客园](https://www.cnblogs.com/handsome1013/p/12093036.html) 以下为原文

用这两种方法就可以了

在使用if标签的时候常常会用到

```
<if test=" name!=null && name =='1' "><if/> 
```

这样子写会出现 后面的  name =='1' 失效问题。 这个很多人会踩的坑。

网上有解决办法就是 

```
<if test=‘ name!=null && name =="1" '><if/>
```

  把这个转换成 单引号。这样就解决了。

不过我觉得这样解决太麻烦可以这样解决

```
<if test=" name!=null && name =='1'.toString() "><if/> 
```

这样就可以完美解决了。。

 

在做开发的时候遇到这样一个问题：当传入的type的值为y的时候，if判断内的sql也不会执行。

```
 <if test="type=='y'">
   and status = 0
 </if>
```

仔细想想：**mybatis是使用的OGNL表达式来进行解析的，在OGNL的表达式中，'y'会被解析成字符，因为java是强类型的，char 和 一个String 会导致不等。所以if标签中的sql不会被解析**。

所以，需要解决这个问题，只需要把代码修改成：

```
 <if test='type=="y"'> //注意是双引号，不是单引号！！！
   and status = 0
 </if>
```

就可以执行了，这样"y"解析出来是一个字符串，两者相等！
