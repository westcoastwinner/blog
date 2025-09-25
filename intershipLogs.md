设计Entity

字段int/Integer

```
@TableName(value = "process_address", autoResultMap = true)
```

前端传回的json对象->Java 

## Wrapper与SQL的各种问题集合

**逆天的wrapper复用问题：**
for循环里用一个外部定义的wrapper.eq会无限叠加！！！
应在for循环里面每次重新定义

**逆天的wrapper.or()优先级问题**：

由于SQL中AND优先级高于OR，所以在SQL语句中 

```
WHERE a OR b AND c 
```

会被解析为

```
WHERE a OR (b AND c)
```

但这一点在MP的or()中体现不出来，例如：

```java
LambdaQueryWrapper<ProcessAddress> wrapper = new LambdaQueryWrapper<>(); if(ObjectUtils.isNotEmpty(req)){ 
    String message = req.getMessage(); 
    wrapper.like(ProcessAddress::getAddressName, message) 
        .or() 
        .like(ProcessAddress::getConnectorCode, message); 
}
wrapper.eq(ProcessAddress::getCreatedBy,userId) .eq(ProcessAddress::getDeleteFlag,"0"); List<ProcessAddress> records = processAddressMapper.selectList(wrapper);
```

在执行时这条语句会被解析为：

```
WHERE address_name LIKE '%xxx%'
   OR (connector_code LIKE '%xxx%' AND created_by = #{userId} AND delete_flag = '0')
```

 正确写法

你要让 `addressName` 和 `connectorCode` 作为一组 `OR` 条件，然后再和其他条件一起 `AND`。
 在 MyBatis-Plus 里可以用 **`and()` 包裹子条件**：

```
LambdaQueryWrapper<ProcessAddress> wrapper = new LambdaQueryWrapper<>();
if (ObjectUtils.isNotEmpty(req)) {
    String message = req.getMessage();
    wrapper.and(w -> w.like(ProcessAddress::getAddressName, message)
                      .or()
                      .like(ProcessAddress::getConnectorCode, message));
}
wrapper.eq(ProcessAddress::getCreatedBy, userId)
       .eq(ProcessAddress::getDeleteFlag, "0");
```

生成的 SQL 就是：

```
WHERE (address_name LIKE '%xxx%' OR connector_code LIKE '%xxx%')
  AND created_by = #{userId}
  AND delete_flag = '0'
```

**逆天的wrapper.in（）问题**

当然，该问题原罪不是MP，写在这里为了工整...

你调用的是：

```
dataSpaceMapper.selectBatchIds(dataSpacesIds);
```

如果 `dataSpacesIds` 是个 **空集合**，MyBatis-Plus 会拼出：

```
... WHERE space_id IN ()
```

这在 MySQL 中会直接报错。

------

解决办法

你需要在调用前判断一下集合是否为空，避免传空集合进去。比如：

```
List<DataSpaceEntity> dataSpaceEntities = Collections.emptyList();
if (CollectionUtils.isNotEmpty(dataSpacesIds)) {
    dataSpaceEntities = dataSpaceMapper.selectBatchIds(dataSpacesIds);
}
```



实习日志

| 序号 | 任务                                                         | 时间      | 备注 |
| ---- | ------------------------------------------------------------ | --------- | ---- |
| 1    | 完成加工地址管理和交付地址管理的数据库设计、接口设计、代码开发 | 8.20-8.27 |      |
|      |                                                              |           |      |
|      |                                                              |           |      |

# 1

### 需求1

据原型，加工地址管理界面有以下需求

1.分页查询（列表/条件）；2.修改地址的相关信息；3.删除地址，根据地址绑定的订单是否上架决定；4.新增地址5.对其他服务提供一个接口 6.调用rpc获取业务节点列表

### 数据库设计

**process_address**  - **process_address_env  **- **process_env**  中间为桥表 （多对多关系）

### 接口设计 

参考接口文档



### 需求2

据原型，交付地址管理界面有以下需求：

1.分页查询（列表/条件）；2.将地址设置为默认：要么没有默认地址，要么最多只有一个默认地址。每个地址后面有个设为默认的左右按钮，重复点击可设为默认/取消默认；3.新增地址

### 数据库设计

**delivery_address**

### 接口设计 

参考接口文档



## 经验总结

### 1.注意查询列表时要要根据用户id，用户id需要先判断是否登录：

```
String userId = SecurityUtils.getUserIdentity();
if (ObjectUtils.isEmpty(userId)) {
    throw new BusinessException("用户未登录");
}
```

### 2.@Transactional

对于一个对表有多次操作的方法，需要加事务（一般在业务层实现类加）。注意默认 运行时异常 时回滚，其他情况需要指明；

### 3.后端开发的本质

是读写和处理数据，要把握两个口：①与前端交互的口②与存储介质交互的口。

对于前端的请求，通常要求请求参数放在请求体中，以application/json请求，这样controller方法可以加@Requestbody从请求体中获取参数。
由于前端请求是以json传递，springboot不需要封装的请求对象(DTO)实现序列化接口，Spring 会使用 **`HttpMessageConverter`**（默认是 Jackson）来把请求体中的 JSON 自动反序列化为你方法参数对应的 Java 对象（通常就是 DTO）。

对于数据库表，在程序中需要定义Entity对象，这个对象与表中字段一般一一对应或者只多不少。

对于处理好的数据，在返回前端时，往往会遇到以下情形：a.有敏感字段 b.多表查询需要合并结果。这时候通常定义一个VO对象（现代开发中有时定义DTO也可以，命名无所谓）。



### 4.UUID（Universally Unique Identifier，通用唯一识别码）

是分布式系统里常用的一种“全局唯一 ID”生成方式。不依赖数据库自增，不需要中心服务。

UUID 不是单一生成规则，而是有不同“版本”的标准（RFC 4122），常见有：

- **UUIDv1**（基于时间戳 + 节点 MAC 地址）
  - 用时间戳和计算机的硬件地址生成。
  - 保证不同机器、不同时间不会重复。
  - 缺点：可能暴露 MAC 地址，带来隐私问题。
- **UUIDv4**（基于随机数）
  - 完全由伪随机数（122 位随机）生成。
  - 重复概率极低（几乎为零），常见于 Java 的 `UUID.randomUUID()`。
- **UUIDv5**（基于命名空间 + 哈希）
  - 用一个名字（比如域名）加上命名空间，通过 SHA-1 生成。
  - 在相同输入下始终生成相同 UUID（确定性）。

在实际开发中常用 **UUIDv4**（随机数）来生成。

如果要排序（比如数据库主键），UUID 不太合适（因为随机性导致索引性能差），常用 **雪花算法（Snowflake）** 这类方案。

### 5.stream方法实践

```
//平民版 -- 把对象集合映射为对象的某个属性的集合
List<Integer> envIds = processAddressEnvs.stream().map(ProcessAddressEnv::getEnvId).collect(Collectors.toList());

//进阶版 -- 根据某些条件进行过滤
objects.stream().filter(...)。collect();

//至尊版 -- 把对象集合映射为集合内的两个属性一一对应的map
Map<String, String> collect = dataSpaceEntities.stream().collect(Collectors.toMap(DataSpaceEntity::getSpaceId, DataSpaceEntity::getSpaceName));
```

#### 🔹reduce()方法

------

#####  **`Stream.reduce`**

- **位置**：在 `Stream` API 里
- **作用**：把一堆元素“折叠”成一个结果
- **操作对象**：一个流里的元素
- **结果**：是一个单值（可能是数值、字符串、对象……）

例子：

```
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
int sum = nums.stream()
              .reduce(0, (a, b) -> a + b);

System.out.println(sum); // 10
```

这里 `reduce` 把 `[1,2,3,4]` 折叠成了一个值 `10`。

我想知道reduce这个方法所需的两个参数是什么意思？

###### 

好问题 👍，`Stream.reduce` 最容易让人迷糊的就是它的参数。其实 **reduce = 归约/折叠**，就是不断把流里的元素“聚合”成一个最终结果。

------

##### reduce 的常见两种写法

**1. 两个参数版本**

```
<T> T reduce(T identity, BinaryOperator<T> accumulator)
```

- **identity**：初始值（又叫“归约起点”）。
  - 流为空时就直接返回它。
  - 它还会作为第一个参数传给累加器。
- **accumulator**：累加器函数 `(partialResult, element) -> newPartialResult`。
  - `partialResult`：上一次计算的结果（第一次时是 `identity`）。
  - `element`：流里的当前元素。
  - 返回值：新的部分结果。

**例子：求和**

```
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
int sum = nums.stream()
              .reduce(0, (a, b) -> a + b);

System.out.println(sum); // 10
```

这里：

- `identity = 0`
- 累加器 `(a, b) -> a + b`
- 计算过程：`(((0+1)+2)+3)+4 = 10`

------

**2. 一个参数版本**

```
Optional<T> reduce(BinaryOperator<T> accumulator)
```

- 没有 `identity`，所以返回值是 `Optional<T>`（因为流可能为空）。
- 用法一样，只是没初始值。

**例子：求最大值**

```
List<Integer> nums = Arrays.asList(3, 5, 2, 8);
Optional<Integer> max = nums.stream()
                            .reduce((a, b) -> a > b ? a : b);

System.out.println(max.get()); // 8
```

------

**🔑 总结**

- **identity**：初始值 & 空流的兜底值。
- **accumulator**：定义“如何把部分结果和新元素合并”的函数。

------

#### 🔹distinct()方法

#### 🔹count()方法

#### 🔹mapToInt()方法

#### 🔹sum()方法

### 6.时间日期格式

前端传回的时间格式一般应该用字符串接收，但在数据库中查询时，这个格式应转换为Java 的LocalDateTime

```
LocalDateTime beginTime=LocalDateTime.now(),endTime=LocalDateTime.now();
if(StringUtils.isNotEmpty(req.getBeginTime())) {
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    beginTime = LocalDateTime.parse(req.getBeginTime(), formatter);

}
if(StringUtils.isNoneEmpty(req.getEndTime())){
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    endTime = LocalDateTime.parse(req.getEndTime(), formatter);
}
```

至于这个时间格式在数据库中的查询（= > <）则可能由数据库自行处理，我们无需关注。

### 7.MyBatis问题 迁移至

### 8.分级目录表的设计与查询

### 9.设为默认/取消默认且保持最多只有一个默认地址的逻辑：

```
DeliveryAddress deliveryAddress = deliveryMapper.selectOne(new LambdaQueryWrapper<DeliveryAddress>().eq(DeliveryAddress::getId, req.getId()));
//先查询该用户的所有地址
if(deliveryAddress.getDef().equals("0")){
    LambdaUpdateWrapper<DeliveryAddress> wrapper = new LambdaUpdateWrapper<>();
    wrapper.eq(DeliveryAddress::getDef, "1")
            .eq(DeliveryAddress::getCreatedBy, userId)
            .set(DeliveryAddress::getDef, "0");
    deliveryMapper.update(null, wrapper);

    LambdaUpdateWrapper<DeliveryAddress> wrapper1 = new LambdaUpdateWrapper<>();
    wrapper1.eq(DeliveryAddress::getId, req.getId())
            .set(DeliveryAddress::getDef, "1");
    deliveryMapper.update(null, wrapper1);
}else{
    LambdaUpdateWrapper<DeliveryAddress> wrapper = new LambdaUpdateWrapper<>();
    wrapper.eq(DeliveryAddress::getId, req.getId())
            .set(DeliveryAddress::getDef, "0");
    deliveryMapper.update(null, wrapper);
}
```

### 10.HTTP(s)+RestfulAPI方式服务请求与调用

**Feign 的角色：聪明的 HTTP 客户端**

**Feign 本质上就是一个 HTTP 客户端**。它最核心的功能是：

1. **接口-请求转换**：您只需要定义一个带有注解的 Java 接口（比如您之前提到的 `@FeignClient`），Feign 就会根据接口上的注解（如 `@PostMapping`、`@GetMapping`、`@RequestBody` 等），自动生成并封装出符合 RESTful 规范的 **HTTP 请求**。
2. **简化调用**：它将远程过程调用（RPC）的复杂性隐藏起来，让您感觉就像在调用一个本地方法，从而大大简化了服务调用方的开发工作。

可以把 Feign 想象成一个专为 Java 设计的、非常聪明的快递员。您告诉它：“帮我把这个 `IdentityVerifyRpcRequest` 包裹，通过 `POST` 方式寄到 `/connect/v1/serviceNode/identity/verify/challenge` 地址。” Feign 就会自动打包、贴好标签，然后去发送 HTTP 请求。



**Spring Boot 的角色：强大的请求分发员**

您的类比也非常恰当。在服务提供方，**Spring Boot 的确扮演着分发员（DispatcherServlet）的角色**。它的工作流程是：

1. **请求监听**：它持续监听指定的端口，等待接收来自 Feign 或其他客户端的 HTTP 请求。
2. **请求分发**：当一个 HTTP 请求到达时，Spring Boot 会解析请求的 **URL 路径**（如 `/connect/v1/serviceNode/identity/verify/challenge`）和 **HTTP 方法**（`POST`）。
3. **映射到方法**：它会根据这些信息，在所有 `@RestController` 中找到那个用 **`@PostMapping`** 声明了相同路径的方法。
4. **参数绑定与调用**：一旦找到匹配的方法，它会将 HTTP 请求体中的数据自动绑定到方法参数上，并最终调用该方法来执行业务逻辑。

因此，**Feign** 负责在客户端把“包裹”打包并发送出去，而 **Spring Boot** 则在服务器端接收这个“包裹”，并准确地派发到负责处理它的方法上。它们各司其职，共同完成了整个远程服务调用过程。

### 11.访问量统计

由于通常这个需求既包含访问量的+1和展示，又包含排序，所以如果使用普通的Redis String类型`redisTemplate.opsForValue.increment(key)`则会导致获取产品列表页时访问量的排序需要调用Java的list.sort方法在（Java虚拟机）内存中排序。

因此通常有两个更优方案

#### 方案一

当统计的是PV(Page Visit)时，由于允许用户刷新页面增加访问量，故可直接使用Redis的`Sorted Set(ZSET)`结构，该结构天然带排序。

注意，该数据结构是集合，包含setName,setMember,memberScore三个字段。

#### 方案二

当统计的是UV(User Visit)时，由于不允许用户刷新页面增加访问量，故必须使用Redis的`HyperLogLog`，该结构允许增加一个参数（如用户Id）来据此去重。

你调用：

```
redisTemplate.opsForHyperLogLog().add("product:visit:uv:" + productId, userId);
```

逻辑是：

- key = `product:visit:uv:1001`
- value = `userId`（用户唯一标识，比如账号 ID / IP / 会话 ID）
- Redis 会自动去重，只要同一个 `userId` 已经加过，再次加不会影响结果。

结果是：

- 用户 A 访问商品 1001 三次
- 用户 B 访问商品 1001 两次
- `HyperLogLog` 统计 UV = 2（因为只有 A、B 两个不同用户）。

获取 UV 的方法：

```
Long uvCount = redisTemplate.opsForHyperLogLog().size("product:visit:uv:" + productId);
```

 **HyperLogLog** 

- 近似统计：它不是精确集合，而是概率统计，误差在 0.81% 左右。
- 空间占用极低：无论你统计 10 个还是 10 亿个用户，内存都固定在 ~12KB。
- 适合做大规模 UV 去重，但如果要求绝对精确，可以用 `SET` 代替（但内存会爆炸）。

Redis 在 2.8.9 版本添加了 HyperLogLog 结构。

Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

------

什么是基数?

比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。 基数估计就是在误差可接受的范围内，快速计算基数。





最后，还可考虑是否访问量按天统计，这样的话只需改变key的设计（带上日期）。同时，由于商品数量增长引发redis压力，可考虑 **Redis实时高并发写入+MySQL持久化/查询**的方案。



### 12.分页查询

一般来说，可直接使用`mapper.selectPage(Page<?> page, warpper)`方法或IService接口封装的page方法，但此方法只适用于单表查询。在处理连表查询时，sql语句写在mapper.xml文件中，业务层调用mapper的自定义方法。这时，实现分页查询一般有两种途径：

✅方案一：使用 MyBatis-Plus 自带分页插件

最简单的是改 Mapper 方法签名：

```
// Mapper
IPage<TinyProductResponse> selectProductWithPublish(Page<?> page, @Param("req") ProductQueryReq req);
```

调用时这样写：

```
Page<TinyProductResponse> page = new Page<>(req.getPageNum(), req.getPageSize());
IPage<TinyProductResponse> result = dataProductPublishMapper.selectProductWithPublish(page, productQueryReq);
return AjaxResult.success(result);
```

这样 MyBatis-Plus 会自动拼接 `LIMIT` 语句，并帮你查询 `total`。效率高、写法干净。

------

✅ 方案二：手写 SQL + limit/offset

如果你的 SQL 比较复杂，或者没法用 MyBatis-Plus 插件，你可以：

- 写一个 **count SQL** 查询总数；

- 写一个 **分页 SQL**，在结尾加上：

  ```
  LIMIT #{pageSize} OFFSET #{(pageNum - 1) * pageSize}
  ```

- Java 代码里分别执行这两个查询，再组装 Page 对象。

### 13.字符串（集合）操作api

**拆分字符串--split**

```java
String a="a,b,c";
List<String> s=a.split(",");//"a","b","c"

//流操作  dataSpaceIds--"a,b,c"
ids = Arrays.stream(dataSpaceIds.split(","))
                        .map(String::trim)
                        .filter(s -> !s.isEmpty())
                        .collect(Collectors.toList());
```

**合并字符串--join**

```java
//spaceNames "a" "b" "c"
List<String> spaceNames = dataSpaces.stream().map(DataSpaceEntity::getSpaceName).collect(Collectors.toList());

String res=String.join(",",spaceNames);//"a,b,c"

//流操作--强大的Collectors工具类
String result = dataSpaces.stream()
        .map(DataSpaceEntity::getSpaceName)
        .collect(Collectors.joining(","))
```

### 14.SQL 查询的一般执行顺序

1. **FROM / JOIN**：先构造出所有表的笛卡尔积或关联结果。
2. **WHERE**：过滤数据行。
3. **GROUP BY**：按照指定列进行分组，每组会生成一个集合。
4. **聚合函数（COUNT, SUM, AVG...）**：在每个组内部进行统计计算。
5. **HAVING**：对分组之后的结果进行过滤。
6. **SELECT**：输出需要的列（如果用了聚合函数，必须保证非聚合列出现在 GROUP BY 中）。
7. **ORDER BY**：最后排序。

因此，**group by** 是根据笛卡尔积后的（中间）表的字段来分组。

**案例：统计每个数据空间表（data_space）中的成员数**

```
data_space
{
space_id
space_name
...
}

space_member
{
id
space_id
identity_id
...
}

```



```sql
//方案一
select ds.space_id, count(sm.id) as member_count
from data_space ds
left join space_member sm
on ds.space_id=sm.space_id
group by ds.space_id

//方案二
select ds.space_id, coalesce(member_count,0)
from data_space ds
left join(
  select sm.space_id,count(sm.id) as member_count
  from space_member sm
  group by sm.space_id
) sm on ds.space_id=sm.space_id
```

### 15.BigDecimal

这是Java的类，用来处理精确小数（尤其是金额）的类型，比 `double` 更安全（不会有二进制浮点误差）。

SpringMVC/Spring Boot 的参数绑定机制可以自动把前端传过来的 `字符串/数字` 转换成 `BigDecimal`，你不需要写额外的转换器。例如前端传 `"123.45"`，Spring Boot 自动会转成 `new BigDecimal("123.45")`。

这个类型和数据库中的Decimal对应。

在**控制精度**上，该类提供方法：

```
//newScale--小数位数，roundingMode--舍入规则
public BigDecimal setScale(int newScale, int roundingMode) ;

//注意用法，该方法本质是对已有的一个BigDecimal对象的内含值做精度控制，所以调用后需要返回值：
BigDecimal old=new BigDecimal("1234.5678");
BigDecimal new = old.setScale(2,//四舍五入) ;//"1234.5678"->"1234.57"
```

在**数值比较**上，该类不可直接比较，需要借助内置min\max方法比较，而该方法不可比较null值，所以往往还需自己额外写判空规则。（注意，这和数据库的decimal很不同，后者可以直接比较）

