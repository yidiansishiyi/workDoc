> 作者：小智
> 

# 智能BI项目

## 1、什么是BI

BI是商业智能（Business Intelligence）的缩写。它是一种通过收集、分析和呈现数据来帮助企业做出更好决策的技术和过程。BI通常涉及使用各种数据仓库、数据挖掘和数据分析工具，以提取有关企业绩效、市场趋势、客户行为等方面的信息。这些数据可以被转化为可视化图表、报表和仪表盘，可协助企业管理层了解他们所运营的公司的情况，并根据这些信息制定更明智的商业决策。

如下图所示：

![img](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/22957787-d7f446b2a08a616f.png)

主流BI平台：[帆软BI](https://www.finebi.com/)、[小马BI](https://bi.zhls.qq.com/#/)、[微软Power BI](https://powerbi.microsoft.com/zh-cn/)

[阿里云的BI平台](https://chartcube.alipay.com/)

传统的 BI 平台：

1. 手动上传数据
2. 手动选择分析所需的数据行和列（由数据分析师完成）
3. 需要手动选择所需的图表类型（由数据分析师完成）
4. 生成图表并保存配置

## 2、本项目的BI平台

区别于传统的BI，用户（数据分析者）只需要导入最原始的数据集，输入想要进行分析的目标（比如帮我分析一下网站的增长趋势)，就能利用AI自动生成一个符合要求的图表以及分析结论。此外，还会有图表管理、异步生成等功能。

**优点：让不会数据分析的用户也可以通过输入目标快速完成数据分析，大幅节约人力成本，将会用到 AI 接口生成分析结果**

## 3、需求分析

1. 智能分析：用户输入目标和原始数据（图表类型），可以自动生成图表和分析结论
2. 图表管理
3. **图表生成的异步化（消息队列）**
4. 对接 AI 能力



## 4、项目架构图

### 4.1 基础流程

基础流程：客户端输入分析诉求和原始数据，向业务后端发送请求。业务后端利用AI服务处理客户端数据，保持到数据库，并生成图表。处理后的数据由业务后端发送给AI服务，AI服务生成结果并返回给后端，最终将结果返回给客户端展示。

> 要根据用户的输入生成图标，借助AI服务

![image-20230626133359830](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230626133359830.png)

> 上图的流程会出现一个问题：
>
> 假设一个 AI 服务生成图表和分析结果要等50秒，如果有大量用户需要生成图表，每个人都需要等待50秒，那么 AI 服务可能无法受这种压力。为了解决这个问题，可以采用消息队列技术。
>
> 这类以于在餐厅点餐时，为了避免顾客排队等待，餐厅会给顾客一个取餐号码，上顾客可以先去坐下或做其他事情，等到餐厅叫到他们的号码时再去领取餐点，这样就能节省等待时间。
>
> 同样地，通过消息队列，用户可以提交生成图表的请求，这些请求会进入队列，AI 服务会衣次处理队列中的请求，从而避免了同时处理大量请求造成的压力，同时也影更好地控制资源的使用。

### 4.2 优化流程（异步化）

![image-20230626134246728](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230626134246728.png)

优化流程（异步化）：客户端输入分析诉求和原始数据，向业务后端发送请求。业务后端将请求事件放入消息队列，并为客户端生成取餐号，让要生成图表的客户端去排队，消息队列根据I服务负载情况，定期检查进度，如果AI服务还能处理更多的图表生成请求，就向任务处理模块发送消息。



任务处理模块调用AI服务处理客户端数据，AI 服务异步生成结果返回给后端并保存到数据库，当后端的AI工服务生成完毕后，可以通过向前端发送通知的方式，或者通过业务后端监控数据库中图表生成服务的状态，来确定生成结果是否可用。若生成结果可用，前端即可获取并处理相应的数据，最终将结果返回给客户端展示。在此期间，用户可以去做自己的事情。

## 5、技术栈

### 5.1 前端

1. Vue3
2. TypeScript
3. element pro
4. 可视化开发库：**Echarts** 
5. apifox 项目文档以及对应的ts代码





### 5.2 后端

1. SpringBoot3、SpringSecurity6 、 JWT
2. MySQL数据库
3. Redis：Redissson限流控制
4. MyBatis Plus 数据库访问结构
5. 消息队列：RabbitMQ
6. AI能力：Open AI接口开发
7. Excel上传和数据的解析：Easy Excel
8. Knife4j 最新4.0  项目文档
9. Hutool 工具库

## 6、数据库设计

### 图表信息表

我这里是自己定义的方式，没有用我的脚手架就不要按照我的方式，不然后面改bug挺多的

```sql
-- 图表信息表
DROP TABLE IF EXISTS `bi_chart`;
create table if not exists bi_chart
(
    id         bigint auto_increment comment 'id' primary key,
    goal       text                               null comment '分析目标',
    chartName  varchar(128)                       null comment '图标名称',
    chartData  text                               null comment '图表信息',
    chartType  varchar(256)                       null comment '图表类型',
    genChart   text                               null comment '生成的图表信息',
    genResult  text                               null comment '生成的分析结论',
    userId     bigint                             null comment '创建图标用户 id',
    create_time datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    update_time datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete   tinyint  default 0                 not null comment '是否删除'
    ) comment '图表信息表' collate = utf8mb4_unicode_ci;
```



## 7. 创建对应的三层框架的文件

我这里只放实体类，其他的用MybatisX插件生成

```java
@TableName(value ="bi_chart")
@Data
public class BiChart extends BaseEntity {
    /**
     * id
     */
    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    /**
     * 分析目标
     */
    @TableField(value = "goal")
    private String goal;

    /**
     * 图表信息
     */
    @TableField(value = "chartName")
    private String chartName;

    /**
     * 图表信息
     */
    @TableField(value = "chartData")
    private String chartData;

    /**
     * 图表类型
     */
    @TableField(value = "chartType")
    private String chartType;

    /**
     * 生成的图表信息
     */
    @TableField(value = "genChart")
    private String genChart;

    /**
     * 生成的分析结论
     */
    @TableField(value = "genResult")
    private String genResult;

    /**
     * 创建图标用户 id
     */
    @TableField(value = "userId")
    private Long userId;

    /**
     * 是否删除
     */
    @TableField(value = "deleted")
    @TableLogic
    private Integer deleted;

}
```

# 智能分析业务开发（二）

## 1、业务流程

1. 用户输入
   1. 分析目标
   2. 上传原始数据(excel)
   3. 更精细地控制图表：比如图表类型、图表名你等
2. 后端校验
   1. 校验用户的输入是否合法（比如长度）
   2. 成本控制（次数统计和校验、鉴权等
3. 把处理后的数据输入给 AI 模型（调用AI接口），让 AI 模型给我们提供图表信息、结论文本
4. 图表信息（是一段 JSON 配置，是一设代码）、结论文本在前端进行展示

## 2、开发接口

### 2.1 定义上传所需参数

上传腾讯云cos文件需要三个参数：

1. MultipartFile对象来腾讯云cos的api获取上传的文件
2. GenChartByAiRequest对象来获取生成图表的相关参数
3. HttpServletRequest对象来获取当前登录用户的信息



其中GenChartByAiRequest对象放在dto里，作为请求时需要的对象：

```java
@Data
public class GenChartByAiRequest implements Serializable {

    /**
     * 图表名称
     */
    private String chartName;

    /**
     * 分析目标
     */
    private String goal;

    /**
     * 图表类型
     */
    private String chartType;

    private static final long serialVersionUID = 1L;
}
```

### 2.2 校验传过来的参数

一般接受到参数，我们都需要校验参数是否是空，如果为空直接打回去，就避免后面传输的问题

这里接受到genChartByAiRequest，我们需要进行判断

有个小工具：GenerateAllSetter Postfix Completion 

使用这个工具，直接genChartByAiRequest.getall，就可以生成以下代码

```java
String chartName = genChartByAiRequest.getChartName();
String goal = genChartByAiRequest.getGoal();
String chartType = genChartByAiRequest.getChartType();
```

再使用自制的自定义if处理工具ThrowUtils，避免大量的if判断

```java
public class ThrowUtils {

    /**
     * 条件成立则抛异常
     *
     * @param condition
     * @param runtimeException
     */
    public static void throwIf(boolean condition, RuntimeException runtimeException) {
        if (condition) {
            throw runtimeException;
        }
    }

    /**
     * 条件成立则抛异常
     *
     * @param condition
     * @param errorCode
     */
    public static void throwIf(boolean condition, ErrorCode errorCode) {
        throwIf(condition, new BusinessException(errorCode, String.valueOf(errorCode)));
    }

    /**
     * 条件成立则抛异常
     *
     * @param condition
     * @param errorCode
     * @param message
     */
    public static void throwIf(boolean condition, ErrorCode errorCode, String message) {
        throwIf(condition, new BusinessException(errorCode, message));
    }
}
```

然后进行校验，就节省了大量重复工作

```java
// 校验
ThrowUtils.throwIf(StringUtils.isBlank(chartType), ErrorCode.PARAMS_ERROR, "分析类型为空");
ThrowUtils.throwIf(StringUtils.isBlank(goal), ErrorCode.PARAMS_ERROR, "分析目标为空");
ThrowUtils.throwIf(StringUtils.isNotBlank(chartName) && chartName.length() > 100 , ErrorCode.PARAMS_ERROR, "图标名称过长");
```

其实还有另一个方法，就是在GenChartByAiRequest实体类里使用@Validated，发起的数据就会进行校验

学习链接：[SpringBoot——关于controller参数校验](https://blog.csdn.net/FlyLikeButterfly/article/details/128659555)

### 2.3 如何向AI题词（prompt）？

**使用AI过程**

1. 用户输入
2. 系统预设（提前告诉也职责、功能、回复格式要求）+分析目标+压缩后的数据
3. 最简单的系统预设：你是一个数据分析刊师，接下来我会给你我的分析目标和原始数据，请告诉我分析结论。

**原始数据压缩**

AI接口普遍都有输入字数限制，尽可能压缩数据，能够允许多传一些数据，现在网上有了一些解决办法，其实根本方法就是把数据简化，压缩，然后分模块，分批进行。

Token：好像是4096

#### prompt提词技巧

#### 提词技巧1:持续输入，持续优化

我们输入：

> 我给你三行数据，请帮我分析网站的增长趋势，数据如下:
>
> 第一行:日期: 1号, 用户数: 10人
>
> 第二行:日期: 2号，用户数: 20人
>
> 第三行:日期: 3号，用户数: 30人

Al答:

> 根据给出的数据，我们可以看出这个网站的用户数在三天内呈现快速增长的趋势。
>
> 具体来说,该网站用户数从第一天的10人增长到第二天的20人，增加了10人，
>
> 增长率为100%;相比之下,第三天用户数从第二天的20人增长到第三天的30人，
>
> 增加了同样的10人，增长率降至50%。如果这种增长趋势持续下去,我们可以
>
> 预期该网站在未来几天内还会继续吸引更多用户。当然，若要得出更加准确
>
> 的结论,需要更多数据的支持。

---

#### 提词技巧2:数据压缩（封装数据）

其实就是：内容压缩，比如把很长的内容提取关键词，也可以让Al来做，如下

第二次问:

> 我给你三行数据，请帮我分析网站的增长趋势，数据如下:
>
> 表头:日期,用户数
>
> 1号,10
>
> 2号,20
>
> 3号,30

Al回答:

> 根据给出的数据，我们可以计算出每日用户增长量:
>
> ● 第一天增长了10个新用户;
>
> ● 第二天增长了20 \- 10\=10个新用户;
>
> ● 第三天增长了30 \- 20\=10个新用户。
>
> 可以看出，每天的用户增长量都相等，呈现一个线性增长趋势。如果这个网站
>
> 在未来的时间里能够保持现在的增长速度和策略不变,预计每过一天会增加10个
>
> 用户。但需要注意的是,这个预测是基于目前的数据情况，实际情况受到许多因
>
> 素的影响，不能完全准确预测未来用户增长。

---

#### 提词技巧3:使用Al压缩数据

第三次问:

> 我给你三行数据，请帮我用最少的字数压缩这段数据
>
> 第一行:日期: 1号, 用户数: 10人
>
> 第二行:日期: 2号，用户数: 20人
>
> 第三行:日期: 3号，用户数: 30人

Al回答:

> 日期1\-3号,用户数分别为10/20/30人。

#### 提词技巧4:添加系统预设

![img](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/1684659315109-748ef826-bb7b-4a3a-b133-1e4d09035db0.png)

但是呐，其实应该在创建自定义的时候，在系统里添加预设会更准确~~~

除了系统预设外，额外关联一问一答两条消息这种也是非常欧克的！相当于给A!一个提示。

### 2.4 使用csv对excel文件的数据进行提取

#### 简单使用easyexcel

官方文档：[easyexcel官方文档](https://easyexcel.opensource.alibaba.com/docs/current/)

创建一个ExcelUtils工具，并且测试一下是否能够读的到excel的数据

1. 首先创建一个excel，这里创建了一个"网站数据.xlsx"

| 日期 | 用户数 |
| ---- | ------ |
| 1号  | 10     |
| 2号  | 20     |
| 3号  | 30     |

测试代码如下：

```java
@Slf4j
public class ExcelUtils {
    /**
     * excel 转 csv
     *
     * @param multipartFile
     * @return
     */
    public static String excelToCsv(MultipartFile multipartFile) {
        File file = null;
        try {
            file = ResourceUtils.getFile("classpath:网站数据.xlsx");
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
        // 读取数据
        List<Map<Integer, String>> list = null;

        list = EasyExcel.read(file)
                .excelType(ExcelTypeEnum.XLSX)
                .sheet()
                .headRowNumber(0)
                .doReadSync();
        System.out.println(list);
        return " 成功 ";

    }

    public static void main(String[] args) {
        excelToCsv(null);
    }
}
```

注意！！！不要直接在idea里创建xlsx，会导致奇奇怪怪的问题！！！我是遇到了NotOfficeXmlFileException错误！！

这件是格式不对导致的。

```shell
Exception in thread "main" org.apache.poi.openxml4j.exceptions.NotOfficeXmlFileException: 
No valid entries or contents found, this is not a valid OOXML (Office Open XML) file
```

解决bug之后，用debug查看一下数据

![](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230712220458214.png)

#### 获取数据进行拼接

```java
public static String excelToCsv(MultipartFile multipartFile) {
    File file = null;
    try {
        file = ResourceUtils.getFile("classpath:网站数据.xlsx");
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    }
    // 读取数据
    List<Map<Integer, String>> list = null;
    try {
        list = EasyExcel.read(file)
                .excelType(ExcelTypeEnum.XLSX)
                .sheet()
                .headRowNumber(0)
                .doReadSync();
    } catch (IOException e) {
        log.error("表格处理错误", e);
    }
    if (CollUtil.isEmpty(list)) {
        return "";
    }
    // 转换为 csv
    StringBuilder stringBuilder = new StringBuilder();
    // 读取表头
    LinkedHashMap<Integer, String> headerMap = (LinkedHashMap) list.get(0);
    List<String> headerList = headerMap.values().stream().filter(ObjectUtils::isNotEmpty).collect(Collectors.toList());
    stringBuilder.append(StringUtils.join(headerList, ",")).append("\n");
    // 读取数据
    for (int i = 1; i < list.size(); i++) {
        LinkedHashMap<Integer, String> dataMap = (LinkedHashMap) list.get(i);
        List<String> dataList = dataMap.values().stream().filter(ObjectUtils::isNotEmpty).collect(Collectors.toList());
        stringBuilder.append(StringUtils.join(dataList, ",")).append("\n");
    }
    System.out.println(stringBuilder.toString());
    return stringBuilder.toString();
}
```

然后再次测试一下，就可以发现数据没问题啦！！

![image-20230712224703458](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230712224703458.png)



# 智能分析业务开发（三）

## 1、三种AI数据分析应用方式

我们需要一步步让AI生产我们所需要的代码，这需要一步步的prompt，让AI稳定的输出我们所需要的

### 1.1 AI生成结论



询问AI：

```java
你是一个数据分析师，请帮我分析网站用户的增长趋势
原始数据如下：
日期，用户数
1号，10
2号，20
3号，30
```

AI回答：

```java
通过观察给出的原始数据，我们可以计算每天的用户增长量并绘制出用户增长趋势图。

首先，计算每天的增长量，可以通过当前日期的用户数减去前一天的用户数得到。然后，我们将增长量绘制成一个图表，横轴表示日期，纵轴表示用户增长量。

以下是根据提供的数据得到的增长趋势图：

日期 用户增长量
1号 10
2号 10
3号 10

根据数据可以看出，在这个时间段内，每天的用户增长量均为10。

根据这个趋势图可以推断，用户增长是稳定的，每天增长10个用户。
```

### 1.2 AI生成图表

AI无法直接生成现成的图表，但是AI可以生成图表代码

然后可以把代码利用前端的组件库([Echarts](https://echarts.apache.org/zh/index.html))在网页进行展示   =>    [Echarts在线调用](https://echarts.apache.org/examples/zh/editor.html?c=line-simple)

预期生成的图表代码

```javaScript
option = {
  xAxis: {
    type: 'category',
    data: ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']
  },
  yAxis: {
    type: 'value'
  },
  series: [
    {
      data: [150, 230, 224, 218, 135, 147, 260],
      type: 'line'
    }
  ]
};
```

示例图表：

<img src="https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/1684589460530-46789b9e-1077-4d8d-b604-f216f68403f1.png" alt="img" style="zoom: 80%;" />

### 1.3 AI 提问分步实现技巧

为了让AI更好地理解我们的输入，并给出预期精确地输出，需要严格控制我们的提问词

#### 1.3.1 使用系统预设 + 控制输入格式

便于AI精确地理解我们的需求

我们给AI添加预设，并且控制输入输出格式：

```java
你是一个数据分析师和前端开发专家，接下来我会按照以下固定格式给你提供内容：
分析需求：
{数据分析的需求或者目标}
原始数据：
{csv格式的原始数据，用，作为分隔符}
请根据以上内容，帮我生成数据分析结论和可视化图表代码
```

然后AI输出就会变得不一样：

![image-20230713094750937](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230713094750937.png)

#### 1.3.2 控制输出格式

通过控制输出的格式，AI返回的内容能够更加方便地为我们所用

Prompt预设：

```java
你是一个数据分析师和前端开发专家，接下来我会按照以下固定格式给你提供内容：
分析需求：
{数据分析的需求或者目标}
原始数据：
{csv格式的原始数据，用,作为分隔符}
请根据这两部分内容，按照以下指定格式生成内容（此外不要输出任何多余的开头、结尾、注释）
【【【【【
{前端 Echarts V5 的 option 配置对象的json格式代码，合理地将数据进行可视化，不要生成任何多余的内容，比如注释}
【【【【【
{明确的数据分析结论、越详细越好，不要生成多余的注释}
```

用户输入：

```java
分析需求：
分析网站用户的增长情况
原始数据：
日期，用户数
1号，10
2号，20
3号，30
```

AI生成的内容：

```html
【【【【【
{
    "title": {
        "text": "网站用户增长情况",
        "x": "center"
    },
    "xAxis": {
        "data": ["1号", "2号", "3号"]
    },
    "yAxis": {},
    "series": [{
        "name": "用户数",
        "type": "line",
        "data": [10, 20, 30]
    }]
}
【【【【【
根据给定的数据，可以看出网站用户数在不断增长。
1号时有10个用户，2号时增长到20个用户，3号时增长到30个用户。可以明显看出网站的用户数呈现上升的趋势。
```

将上面的代码复制到Echarts执行：

![image-20230713100341358](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230713100341358.png)



#### 1.3.4 指定示例问答

可以指定一个示例one-shot或者多个示例few-shot，让AI根据示例的例子来回答

![image-20230713101055747](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230713101055747.png)

然后再试一下，就可以按照我们的想法输出啦~~

## 2、调用AI方式

#### 2.1 直接调用OpenAI或者其他 AI 原始模型官网的接口

比如OpenAl或者其他Al原始大模型官网的接口 官方文档：https://platform.openai.com/docs/api-reference

优点：不经封装，最灵活，最原始 缺点：要钱、要魔法

本质上OpenAl就是提供了HTTP接口，我们可以用任何语言去调用

1. 在请求头中指定OPENAI API KEY

   Authorization:Bearer OPENAI API KEY

2. 找到你要使用的接☐，比如Al对话接口：https://platform.openai.com/docs/api-reference/chat/create

3. 按照接口文档的示例，构造HTTP请求，比如用Hutool工具类、或者HTTPClient

```
/**
 * AI 对话（需要自己创建请求响应对象）
 *
 * @param request
 * @param openAiApiKey
 * @return
 */
public CreateChatCompletionResponse createChatCompletion(CreateChatCompletionRequest request, String openAiApiKey) { 
    if (StringUtils.isBlank(openAiApiKey)) {
        throw new BusinessException(ErrorCode.PARAMS_ERROR, "未传 openAiApiKey");
     }
     String url = "https://api.openai.com/v1/chat/completions";
     String json = JSONUtil.toJsonStr(request);
     String result = HttpRequest.post(url)
                                 .header("Authorization", "Bearer " + openAiApiKey)
                                 .body(json)
                                 .execute()
                                 .body();
     return JSONUtil.toBean(result, CreateChatCompletionResponse.class);
}
```



#### 2.2 使用云服务商提供的封装接口

比如：Azure云 优点：本地都用 缺点：依然要钱，而且可能比直接调用原始的接口更贵



#### 2.3 鱼聪明AI开放平台

鱼聪明 AI 提供了现成的SDK来让大家更方便地使用AI能力

鱼聪明 Al 网站：https://yucongming.com/ (也可以直接公众号搜索【鱼聪明Al】移动端使用)

优点：目前不要钱，而且有很多现成的模型(prompt系统预设)给大家用

缺点：不完全灵活，但是可以定义自己的模型

参考文章进行操作：https://github.com/liyupi/yucongming-java-sdk



## 3、使用AI开放平台调用

### 3.1 初试调用鱼AI

第一步：引入sdk

```java
<dependency>
    <groupId>com.yucongming</groupId>
    <artifactId>yucongming-java-sdk</artifactId>
    <version>0.0.3</version>
</dependency>
```

第二步： 获取密钥

在 [鱼聪明 AI 开放平台](https://www.yucongming.com/dev) 获取开发者密钥对

第三步：初始化YuCongMingClient 对象

方法 1：自主 new 对象

```
String accessKey = "你的 accessKey";
String secretKey = "你的 secretKey";
YuCongMingClient client = new YuCongMingClient(accessKey, secretKey);
```

方法 2：通过配置注入对象

修改配置：

```
yuapi:
  client:
    access-key: 你的 access-key
    secret-key: 你的 secret-key
```

使用客户端对象：

```
@Resource
private YuCongMingClient client;
```

我这里用了方法2，简单一点

第四步：创建 AiManager.java 用来对接第三方接口

```java
@Component
public class AiManager {

    @Resource
    private YuCongMingClient yuCongMingClient;

    /**
     * AI 对话
     *
     * @param modelId
     * @param message
     * @return
     */
    public String doChat(long modelId, String message) {
        DevChatRequest devChatRequest = new DevChatRequest();
        devChatRequest.setModelId(modelId);
        devChatRequest.setMessage(message);
        BaseResponse<DevChatResponse> response = yuCongMingClient.doChat(devChatRequest);
        if (response == null) {
            throw new BusinessException(ErrorCode.SYSTEM_ERROR, "AI 响应错误");
        }
        return response.getData().getContent();
    }
}
```

别忘记加@Component，不然后面就会失败

第五步：测试连接是否成功

```java
@SpringBootTest
class AiManagerTest {

    @Resource
    private AiManager aiManager;

    @Test
    void doChat() {
        String doChat = aiManager.doChat(1675772021360898050L,"你好");
        System.out.println(doChat);
    }
}
```

![image-20230713110506274](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230713110506274.png)

没有问题！！

第六步：测试我们自己建的模型

```
@SpringBootTest
class AiManagerTest {

    @Resource
    private AiManager aiManager;

    @Test
    void doChat() {
        Long modelId = 1679305672904269825L;
        String message = "分析需求：\n" +
                "分析网站用户的增长情况\n" +
                "原始数据：\n" +
                "日期，用户数\n" +
                "1号，10\n" +
                "2号，20\n" +
                "3号，30\n" ;
        String doChat = aiManager.doChat(modelId,message);
        System.out.println(doChat);
    }
}
```

![image-20230713111005361](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230713111005361.png)

完美~~~

### 3.2 处理结果

#### 3.1 拼接数据进行分类

无需写 prompt，直接调用现有模型，https://www.yucongming.com，公众号搜【鱼聪明AI】
```java
final String prompt = "你是一个数据分析师和前端开发专家，接下来我会按照以下固定格式给你提供内容：\n" +
       "分析需求：\n" +
       "{数据分析的需求或者目标}\n" +
        "原始数据：\n" +
        "{csv格式的原始数据，用,作为分隔符}\n" +
        "请根据这两部分内容，按照以下指定格式生成内容（此外不要输出任何多余的开头、结尾、注释）\n" +
        "【【【【【\n" +
        "{前端 Echarts V5 的 option 配置对象js代码，合理地将数据进行可视化，不要生成任何多余的内容，比如注释}\n" +
        "【【【【【\n" +
        "{明确的数据分析结论、越详细越好，不要生成多余的注释}";
```

然后处理数据进行拼接

```java
// final String prompt = "你是一个数据分析师和前端开发专家，接下来我会按照以下固定格式给你提供内容：。。
long biModelId = CommonConstant.BI_MODEL_ID;
// 分析需求：
// 分析网站用户的增长情况
// 原始数据：
// 日期,用户数
// 1号,10
// 2号,20
// 3号,30

// 构造用户输入
StringBuilder userInput = new StringBuilder();
userInput.append("分析需求：").append("\n");

// 拼接分析目标
String userGoal = goal;
if (StringUtils.isNotBlank(chartType)) {
    userGoal += "，请使用" + chartType;
}
userInput.append(userGoal).append("\n");
userInput.append("原始数据：").append("\n");

// 压缩后的数据
String csvData = ExcelUtils.excelToCsv(multipartFile);
userInput.append(csvData).append("\n");

String result = aiManager.doChat(biModelId, userInput.toString());
String[] splits = result.split("【【【【【");
if (splits.length < 3) {
     throw new BusinessException(ErrorCode.SYSTEM_ERROR, "AI 生成错误");
}

String genChart = splits[1].trim();
String genResult = splits[2].trim();
```

#### 3.2 将数据进行封装

得到结果之后，我们对结果的数据进行封装，就需要创建一个vo实体类

```java
@Data
public class BiVo {
	
    private String genChart;

    private String genResult;

    private Long chartId;
}
```

然后把数据插入数据库

```java
// 插入到数据库
BiChart chart = new BiChart();
chart.setChartName(chartName);
chart.setGoal(goal);
chart.setChartData(csvData);
chart.setChartType(chartType);
chart.setGenChart(genChart);
chart.setGenResult(genResult);
chart.setUserId(loginUser.getId());
boolean saveResult = chartService.save(chart);
ThrowUtils.throwIf(!saveResult, ErrorCode.SYSTEM_ERROR, "图表保存失败");
BiResponseVo biResponse = new BiResponseVo();
biResponse.setGenChart(genChart);
biResponse.setGenResult(genResult);
biResponse.setChartId(chart.getId());
return Result.success(biResponse);
```

最后得到完整的方法：

```java
@PostMapping("/gen")
public Result<BiResponseVo> genChartByAi(@RequestPart("file") MultipartFile multipartFile,
                                   GenChartByAiRequest genChartByAiRequest, HttpServletRequest request) {
    String chartName = genChartByAiRequest.getChartName();
    String goal = genChartByAiRequest.getGoal();
    String chartType = genChartByAiRequest.getChartType();
    // 校验
    ThrowUtils.throwIf(StringUtils.isBlank(chartType), ErrorCode.PARAMS_ERROR, "分析类型为空");
    ThrowUtils.throwIf(StringUtils.isBlank(goal), ErrorCode.PARAMS_ERROR, "分析目标为空");
    ThrowUtils.throwIf(StringUtils.isNotBlank(chartName) && chartName.length() > 100 , ErrorCode.PARAMS_ERROR, "图标名称过长");
    // 校验文件
    long size = multipartFile.getSize();
    String originalFilename = multipartFile.getOriginalFilename();
    // 校验文件大小
    final long ONE_MB = CommonConstant.ONE_MB;
    ThrowUtils.throwIf(size > ONE_MB, ErrorCode.PARAMS_ERROR, "文件超过 1M");
    // 校验文件后缀 aaa.png
    String suffix = FileUtil.getSuffix(originalFilename);
    final List<String> validFileSuffixList = Arrays.asList("xlsx");
    ThrowUtils.throwIf(!validFileSuffixList.contains(suffix), ErrorCode.PARAMS_ERROR, "文件后缀非法");

    SysUser loginUser = userService.getLoginUser(request);
    // 限流判断，每个用户一个限流器
    redisLimiterManager.doRateLimit("genChartByAi_" + loginUser.getId());
    // 省略添加prompt
    long biModelId = CommonConstant.BI_MODEL_ID;
    // 分析需求：
    // 分析网站用户的增长情况
    // 原始数据：
    // 日期,用户数
    // 1号,10
    // 2号,20
    // 3号,30

    // 构造用户输入
    StringBuilder userInput = new StringBuilder();
    userInput.append("分析需求：").append("\n");

    // 拼接分析目标
    String userGoal = goal;
    if (StringUtils.isNotBlank(chartType)) {
        userGoal += "，请使用" + chartType;
    }
    userInput.append(userGoal).append("\n");
    userInput.append("原始数据：").append("\n");

    // 压缩后的数据
    String csvData = ExcelUtils.excelToCsv(multipartFile);
    userInput.append(csvData).append("\n");

    String result = aiManager.doChat(biModelId, userInput.toString());
    String[] splits = result.split("【【【【【");
    if (splits.length < 3) {
        throw new BusinessException(ErrorCode.SYSTEM_ERROR, "AI 生成错误");
    }

    String genChart = splits[1].trim();
    String genResult = splits[2].trim();
    // 插入到数据库
    BiChart chart = new BiChart();
    chart.setChartName(chartName);
    chart.setGoal(goal);
    chart.setChartData(csvData);
    chart.setChartType(chartType);
    chart.setGenChart(genChart);
    chart.setGenResult(genResult);
    chart.setUserId(loginUser.getId());
    boolean saveResult = chartService.save(chart);
    ThrowUtils.throwIf(!saveResult, ErrorCode.SYSTEM_ERROR, "图表保存失败");
    BiResponseVo biResponse = new BiResponseVo();
    biResponse.setGenChart(genChart);
    biResponse.setGenResult(genResult);
    biResponse.setChartId(chart.getId());
    return Result.success(biResponse);
}
```

下一步进行测试，但是呐，我的系统是跟鱼皮不一样，没办法获得用户登录，所以我就过滤掉，而且传参数接受不到，我就一直改，改到以下情况才接受得到参数

```java
/**
 * 智能分析（同步）
 *
 * @param multipartFile
 * @param genChartByAiRequest
 * @param request
 * @return
 */
@Operation(summary = "AI智能分析", security = {@SecurityRequirement(name = "Authorization")})
@PostMapping("/gen")
    public Result<BiResponseVo> genChartByAi(@RequestPart("file") MultipartFile multipartFile,
                                             @RequestParam("genChartByAiRequest") String genChartByAiRequestJson,
                                             HttpServletRequest request) {
    GenChartByAiRequest genChartByAiRequest = JSONUtil.parseObj(genChartByAiRequestJson).toBean(GenChartByAiRequest.class);
    String chartName = genChartByAiRequest.getChartName();
    String goal = genChartByAiRequest.getGoal();
    String chartType = genChartByAiRequest.getChartType();
    genChartByAiRequest.setChartName(chartName);
    genChartByAiRequest.setGoal(goal);
    genChartByAiRequest.setChartType(chartType);
    // 校验
    ThrowUtils.throwIf(StringUtils.isBlank(chartType), ErrorCode.PARAMS_ERROR, "分析类型为空");
    ThrowUtils.throwIf(StringUtils.isBlank(goal), ErrorCode.PARAMS_ERROR, "分析目标为空");
    ThrowUtils.throwIf(StringUtils.isNotBlank(chartName) && chartName.length() > 100 , ErrorCode.PARAMS_ERROR, "图标名称过长");
    // 校验文件
    long size = multipartFile.getSize();
    String originalFilename = multipartFile.getOriginalFilename();
    // 校验文件大小
    final long ONE_MB = CommonConstant.ONE_MB;
    ThrowUtils.throwIf(size > ONE_MB, ErrorCode.PARAMS_ERROR, "文件超过 1M");
    // 校验文件后缀 aaa.png
    String suffix = FileUtil.getSuffix(originalFilename);
    final List<String> validFileSuffixList = Arrays.asList("xlsx");
    ThrowUtils.throwIf(!validFileSuffixList.contains(suffix), ErrorCode.PARAMS_ERROR, "文件后缀非法");

    /*SysUser loginUser = userService.getLoginUser(request);
    // 限流判断，每个用户一个限流器
    redisLimiterManager.doRateLimit("genChartByAi_" + loginUser.getId());*/

    /*无需写 prompt，直接调用现有模型，https://www.yucongming.com，公众号搜【鱼聪明AI】
    final String prompt = "你是一个数据分析师和前端开发专家，接下来我会按照以下固定格式给你提供内容：\n" +
           "分析需求：\n" +
           "{数据分析的需求或者目标}\n" +
            "原始数据：\n" +
            "{csv格式的原始数据，用,作为分隔符}\n" +
            "请根据这两部分内容，按照以下指定格式生成内容（此外不要输出任何多余的开头、结尾、注释）\n" +
            "【【【【【\n" +
            "{前端 Echarts V5 的 option 配置对象js代码，合理地将数据进行可视化，不要生成任何多余的内容，比如注释}\n" +
            "【【【【【\n" +
            "{明确的数据分析结论、越详细越好，不要生成多余的注释}";*/
    long biModelId = CommonConstant.BI_MODEL_ID;
    // 分析需求：
    // 分析网站用户的增长情况
    // 原始数据：
    // 日期,用户数
    // 1号,10
    // 2号,20
    // 3号,30

    // 构造用户输入
    StringBuilder userInput = new StringBuilder();
    userInput.append("分析需求：").append("\n");

    // 拼接分析目标
    String userGoal = goal;
    if (StringUtils.isNotBlank(chartType)) {
        userGoal += "，请使用" + chartType;
    }
    userInput.append(userGoal).append("\n");
    userInput.append("原始数据：").append("\n");

    // 压缩后的数据
    String csvData = ExcelUtils.excelToCsv(multipartFile);
    userInput.append(csvData).append("\n");

    String result = aiManager.doChat(biModelId, userInput.toString());
    String[] splits = result.split("【【【【【");
    if (splits.length < 3) {
        throw new BusinessException(ErrorCode.SYSTEM_ERROR, "AI 生成错误");
    }

    String genChart = splits[1].trim();
    String genResult = splits[2].trim();

    // 插入到数据库
    /*BiChart chart = new BiChart();
    chart.setChartName(chartName);
    chart.setGoal(goal);
    chart.setChartData(csvData);
    chart.setChartType(chartType);
    chart.setGenChart(genChart);
    chart.setGenResult(genResult);
    chart.setUserId(loginUser.getId());
    boolean saveResult = chartService.save(chart);
    ThrowUtils.throwIf(!saveResult, ErrorCode.SYSTEM_ERROR, "图表保存失败");*/

    BiResponseVo biResponse = new BiResponseVo();
    biResponse.setGenChart(genChart);
    biResponse.setGenResult(genResult);
    // biResponse.setChartId(chart.getId());
    biResponse.setChartId(11L);
    return Result.success(biResponse);
}
```

然后调用后是能拿到数据的

## 4、前端开发

前言：

今天在公司问了前后端开发的相互配合方式，也去问了前端的开发人员了解怎么做

1. 需求分析阶段：产品经理接收客户提出的需求，与开发人员交流实现的可能性，完成一套需求分析文档。在会议上，前后端开发人员和产品经理一起了解需求的逻辑。
2. UI设计阶段：UI设计师设计界面，将设计好的界面交给前端开发人员，前端开发人员搭建界面组件。
3. 后端开发阶段：后端开发人员通过GitLab接收需求文档，确定需求的开发逻辑，为正式开发做好准备。如果需要添加表，需要经过各组的负责人和架构等评审后确定OK才能建表。后端开发人员定义初步的表字段相关的数据信息，确定API接口，实现后端逻辑。
4. 前端开发阶段：前端开发人员根据信息定义数据类型和API接口，并在组件上绑定好对应的内容。
5. 开发过程阶段：后端开发人员和前端开发人员在开发过程中互相交流，例如后端接收的请求参数和前端发起的请求参数等信息会在这个时间互相配合，完成需求。
6. 评审阶段：开发基本实现后，开发人员会开展评审会议，讲解自己的代码实现，其他开发人员和产品经理会提出问题和建议，最终确定是否通过评审。
7. 测试阶段：开发完成后，进行单元测试、集成测试、系统测试等测试，测试人员对系统进行全面的测试，发现问题后反馈给开发人员进行修复。
8. 上线阶段：测试通过后，将代码部署到生产环境中，上线运行。在上线后，需要进行监控和维护，及时发现和解决问题。

### 4.1 新增页面并基础设置

#### 4.1.1 创建页面

在目录src下进入router文件夹，找到index.ts，并且添加页面

```tsx
{
		path: "/",
		component: Layout,
		redirect: "/bi",
		children: [
			{
				path: '/bi/addChart', // 页面的访问路径
				component: () => import('@/views/addChart/index.vue'), // 页面组件的引用
				name: 'AddChart', // 页面的名称
				meta: { title: '智能分析', icon: 'chart' } // 页面的元信息，比如页面标题、图标等
			},
		]
	},
```

![image-20230713162236016](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230713162236016.png)

这里注意，添加的时候，要设置它的父组件是Layout，不然，没有侧边栏等

#### 4.1.2 配置页面组件

我使用了element plus组件，根据分析，我们需要一个表单来实现，去官网看一下组件：[Element Plus](https://element-plus.gitee.io/zh-CN/)

找到表单，复制进行粘贴，然后进行修改，还需要添加一个上传文件的组件，然后排序一下

我这里花了大量的时间，因为是第一次真正意义上的做前端，所以花了很长的时间了解各个组件的意义

组件最后如下：

```vue
<template>
	<div class="app-container">
		<el-form :model="formData" rules="rules" label-width="160px" label-position="left">

			<el-form-item class="chartGoal" label="分析目标：">
				<el-input v-model="formData.goal" type="textarea" />
			</el-form-item>

			<el-form-item class="chartName" label="图表名称：">
				<el-input v-model="formData.chartName" />
			</el-form-item>

			<el-form-item class="chartType" label="图表类型：">
				<el-select v-model="formData.chartType" placeholder="请选择图表类型">
					<el-option v-for="(option, index) in options" :key="index" :label="option.label" :value="option.value">
					</el-option>
				</el-select>
			</el-form-item>

			<el-form-item class="uploadExcel" label="上传excel数据文件：">
				<el-upload
					v-model:file-list="fileList"
					action=""
					:http-request="uploadFile"
					:file-list="excelFilelist"
					:limit="1"
				>

					<el-button type="primary" class="uploadButton">
						<el-icon class="el-icon--upload">
							<i-ep-upload-filled />
						</el-icon>
						点击上传
					</el-button>
					<div class="el-upload__remind">只允许上传xls/xlsx files</div>
					<template #tip>
						<div class="el-upload__tip">

						</div>
					</template>
				</el-upload>

			</el-form-item>

			<el-form-item>
				<el-button class="submitbut" type="primary" @click="submitData" >提交</el-button>
				<el-button @click="resetForm">重置</el-button>
			</el-form-item>

		</el-form>
	</div>
</template>

<style lang="scss" scoped>
.app-container {
	margin-left: 30px;
	margin-top: 20px;
	width: 50%;
}

.chartGoal{
  margin-bottom: 30px;
}

.chartName{
  margin-bottom: 30px;
}

.chartType{
  margin-bottom: 30px;
}

.uploadExcel{
	margin-bottom: 20px;
}


.uploadButton{
  border
```



#### 4.1.3 定义组件对应所需的数据和接口信息

1. 在api文件夹下的bi文件夹里面，创建两个文件，index.ts和types.ts，在types.ts里把接口信息和对应的属性写好

```ts
/**
 * BI智能传输数据类型
 */
export interface BiForm {
 /**
  * 分析目标
  */
 goal?: string;
 /**
  * 图表名称
  */
 chartName?: string;
 /**
  * 图表类型
  */
 chartType?: string;
}

/**
 * BI响应的结果
 */
 export interface BiResult {
	/**
	 * 图表数据
	 */
	genChart?: string;
	/**
	 * 图表结论
	 */
	 genResult?: string;
}
```

2. 下一步把组件对应所需的数据定义好

```vue
<script setup lang="ts">
import { reactive, ref } from "vue";
import { UploadUserFile } from "element-plus";
import { BiForm } from "@/api/bi/types";
import { genChartByAi, uploadFileOss} from "@/api/bi/index";

const fileList = ref<UploadUserFile[]>([]);

const formData = reactive<BiForm>({ goal: '', chartName: '', chartType: '' });

const rules = reactive({
 goal: [{ required: true, message: "分析目标不能为空", trigger: "blur" }],
 chartName: [{ required: true, message: "图表名称不能为空", trigger: "blur" }],
 chartType: [{ required: true, message: "图表类型不能为空", trigger: "blur" }],
});

const options: { value: string, label: string }[] = [
 { value: '饼图', label: '饼图' },
 { value: '地图', label: '地图' },
 { value: '折线图', label: '折线图' },
 { value: '柱状图', label: '柱状图' },
 { value: '条形图', label: '条形图' },
 { value: '散点图', label: '散点图' },
 { value: 'K线图', label: 'K线图' },
];
</script>
```

### 4.2 定义请求函数

#### 4.2.1 上传文件函数定义

上传文件需要调用接口，把文件上传到oss，我这里使用的是阿里云的，所以上传到阿里云服务器

这里我们先把上传文件的axios请求写好：

```tsx
/**
 * 文件上传到阿里云服务器
 * @param file
 */
export function uploadFileOss(file: File){
 const fileData = new FormData();
 fileData.append('file', file);
 return request({
  url: '/api/v1/files',
  method: 'post',
  data: fileData,
  headers: {
   'Content-Type': 'multipart/form-data'
  }
 });
}
```

下一步写好我们上传文件的按钮调用的函数：

```vue
<script>
    function resetForm() {
     // 重置表单数据和上传文件列表
     formData.goal = "";
     formData.chartName = "";
     formData.chartType = "";
     fileList.value = [];
    };

    function uploadFile(params: { file: any; }) {
	const file = params.file;
	if (!file) {
		ElMessage.warning("上传Excel文件不能为空");
		return false;
	}
	if (!/\.xlsx|\.xls|\.XLSX|\.XLS$/i.test(file.name)) {
		ElMessage.warning("上传Excel只能为xlsx、xls格式");
		fileList.value = [];
		return false;
	}
	uploadFileOss(file)
		.then(res => {
			console.log('[uploadFile] 上传成功', res);
		})
		.catch(err => {
			console.error('[uploadFile] 上传失败', err);
		});
}
</script>
```

后端部分：

记得在application-dev里把oss的type改成oss

#### 4.2.2 提交的函数定义

一样，先写好axios请求

```ts
/**
 * 发起智能分析
 * @param GenChartByAiRequestParams
 * @param File
 */
export function genChartByAi(multipartFile: File, params: BiForm) {
 const formData = new FormData();
 formData.append('file', multipartFile, multipartFile.name);
 formData.append('genChartByAiRequest', JSON.stringify(params));
 return request({
  url: '/api/v1/chart/gen',
  method: 'post',
  data: formData,
  headers: {
   'Content-Type': 'multipart/form-data'
  }
 });
}
```

下一步再写好对应提交智能分析的函数

```vue
<script>
function submitData(){
// 检查上传文件列表中是否有文件
	const multipartFile = fileList.value[0]?.raw;
	if (!multipartFile) {
		ElMessage.warning("上传Excel文件不能为空");
		return false;
	}
	// 提交表单数据
	console.log('表单数据', formData, fileList.value);

	genChartByAi(multipartFile, formData)
		.then(res => {
			console.log('[submitData] 分析成功', res);
			ElMessage.success("分析成功")
		})
		.catch(err => {
			console.error('[submitData] 请求失败', err);
			ElMessage.warning("分析失败：",err);
		});
}
</script>
```

后端最后我也是实现成功啦！！

```java
/**
 * 智能分析（同步）
 *
 * @param multipartFile
 * @param genChartByAiRequestJson
 * @param request
 * @return
 */
@Operation(summary = "AI智能分析", security = {@SecurityRequirement(name = "Authorization")})
@PostMapping("/gen")
    public Result<BiResponseVo> genChartByAi(@RequestPart("file") MultipartFile multipartFile,
                                             @RequestParam("genChartByAiRequest") String genChartByAiRequestJson,
                                             HttpServletRequest request) {
    GenChartByAiRequest genChartByAiRequest = JSONUtil.parseObj(genChartByAiRequestJson).toBean(GenChartByAiRequest.class);
    String chartName = genChartByAiRequest.getChartName();
    String goal = genChartByAiRequest.getGoal();
    String chartType = genChartByAiRequest.getChartType();
    // 校验
    ThrowUtils.throwIf(StringUtils.isBlank(chartType), ErrorCode.PARAMS_ERROR, "分析类型为空");
    ThrowUtils.throwIf(StringUtils.isBlank(goal), ErrorCode.PARAMS_ERROR, "分析目标为空");
    ThrowUtils.throwIf(StringUtils.isNotBlank(chartName) && chartName.length() > 100 , ErrorCode.PARAMS_ERROR, "图标名称过长");
    // 校验文件
    long size = multipartFile.getSize();
    String originalFilename = multipartFile.getOriginalFilename();
    // 校验文件大小
    final long ONE_MB = CommonConstant.ONE_MB;
    ThrowUtils.throwIf(size > ONE_MB, ErrorCode.PARAMS_ERROR, "文件超过 1M");
    // 校验文件后缀 aaa.png
    String suffix = FileUtil.getSuffix(originalFilename);
    final List<String> validFileSuffixList = Arrays.asList("xlsx");
    ThrowUtils.throwIf(!validFileSuffixList.contains(suffix), ErrorCode.PARAMS_ERROR, "文件后缀非法");

    UserInfoVO loginUser = userService.getUserLoginInfo();
    // 限流判断，每个用户一个限流器
    redisLimiterManager.doRateLimit("genChartByAi_" + loginUser.getUserId());

    /*无需写 prompt，直接调用现有模型，https://www.yucongming.com，公众号搜【鱼聪明AI】
    final String prompt = "你是一个数据分析师和前端开发专家，接下来我会按照以下固定格式给你提供内容：\n" +
           "分析需求：\n" +
           "{数据分析的需求或者目标}\n" +
            "原始数据：\n" +
            "{csv格式的原始数据，用,作为分隔符}\n" +
            "请根据这两部分内容，按照以下指定格式生成内容（此外不要输出任何多余的开头、结尾、注释）\n" +
            "【【【【【\n" +
            "{前端 Echarts V5 的 option 配置对象js代码，合理地将数据进行可视化，不要生成任何多余的内容，比如注释}\n" +
            "【【【【【\n" +
            "{明确的数据分析结论、越详细越好，不要生成多余的注释}";*/
    long biModelId = CommonConstant.BI_MODEL_ID;
    // 分析需求：
    // 分析网站用户的增长情况
    // 原始数据：
    // 日期,用户数
    // 1号,10
    // 2号,20
    // 3号,30

    // 构造用户输入
    StringBuilder userInput = new StringBuilder();
    userInput.append("分析需求：").append("\n");

    // 拼接分析目标
    String userGoal = goal;
    if (StringUtils.isNotBlank(chartType)) {
        userGoal += "，请使用" + chartType;
    }
    userInput.append(userGoal).append("\n");
    userInput.append("原始数据：").append("\n");

    // 压缩后的数据
    String csvData = ExcelUtils.excelToCsv(multipartFile);
    userInput.append(csvData).append("\n");

    String result = aiManager.doChat(biModelId, userInput.toString());
    String[] splits = result.split("【【【【【");
    if (splits.length < 3) {
        throw new BusinessException(ErrorCode.SYSTEM_ERROR, "AI 生成错误");
    }

    String genChart = splits[1].trim();
    String genResult = splits[2].trim();

    // 插入到数据库
    BiChart chart = new BiChart();
    chart.setChartName(chartName);
    chart.setGoal(goal);
    chart.setChartData(csvData);
    chart.setChartType(chartType);
    chart.setGenChart(genChart);
    chart.setGenResult(genResult);
    chart.setUserId(loginUser.getUserId());
    boolean saveResult = chartService.save(chart);
    ThrowUtils.throwIf(!saveResult, ErrorCode.SYSTEM_ERROR, "图表保存失败");

    BiResponseVo biResponse = new BiResponseVo();
    biResponse.setGenChart(genChart);
    biResponse.setGenResult(genResult);
    biResponse.setChartId(chart.getId());
    return Result.success(biResponse);
}
```

然后运行，调用，我就解决啦~

### 4.3 优化前端页面（el-row的使用）

使用el-row的布局方式，把页面布置的好看一些

可以参考学习博客：[Element 布局组件el-row和el-col 详解](https://blog.csdn.net/zxlyx/article/details/125895348)

具体过程略，主要是提前把页面设计好，下面给出整个页面的详细代码

```vue
<script setup lang="ts">
import { ref, reactive } from 'vue'
import { UploadUserFile } from "element-plus";
import { BiForm } from "@/api/bi/types";
import { genChartByAi, uploadFileOss} from "@/api/bi/index";

/**
 * 文件
 */
const fileList = ref<UploadUserFile[]>([]);

/**
 * 表单
 */
const formData = reactive<BiForm>({ goal: '', chartName: '', chartType: '' });

/**
 * 表单规则
 */
const rules = reactive({
 goal: [{ required: true, message: "分析目标不能为空", trigger: "blur" }],
 chartName: [{ required: true, message: "图表名称不能为空", trigger: "blur" }],
 chartType: [{ required: true, message: "图表类型不能为空", trigger: "blur" }],
});

/**
 * 图表类型种类
 */
const options: { value: string, label: string }[] = [
 { value: '饼图', label: '饼图' },
 { value: '地图', label: '地图' },
 { value: '折线图', label: '折线图' },
 { value: '柱状图', label: '柱状图' },
 { value: '条形图', label: '条形图' },
 { value: '散点图', label: '散点图' },
 { value: 'K线图', label: 'K线图' },
];

/**
 * 重置表单
 */
function resetForm() {
 // 重置表单数据和上传文件列表
 formData.goal = "";
 formData.chartName = "";
 formData.chartType = "";
 fileList.value = [];
};

/**
 * 上传文件
 */
function uploadFile(params: { file: any; }) {
	const file = params.file;
	if (!file) {
		ElMessage.warning("上传Excel文件不能为空");
		return false;
	}
	if (!/\.xlsx|\.xls|\.XLSX|\.XLS$/i.test(file.name)) {
		ElMessage.warning("上传Excel只能为xlsx、xls格式");
		fileList.value = [];
		return false;
	}
	uploadFileOss(file)
		.then(res => {
			console.log('[uploadFile] 上传成功', res);
		})
		.catch(err => {
			console.error('[uploadFile] 上传失败', err);
		});
}

/**
 * 发起智能分析
 */
function submitData(){
// 检查上传文件列表中是否有文件
 const multipartFile = fileList.value[0]?.raw;
 if (!multipartFile) {
  ElMessage.warning("上传Excel文件不能为空");
  return false;
 }
 // 提交表单数据
 console.log('表单数据', formData, fileList.value);

 genChartByAi(multipartFile, formData)
  .then(res => {
   console.log('[submitData] 分析成功', res);
   ElMessage.success("分析成功")
  })
  .catch(err => {
   console.error('[submitData] 请求失败', err);
   ElMessage.warning("分析失败：",err);
  });
}

</script>

<template>
 <div class="app-container">
  <el-row :gutter="30">
   <!--智能分析表单-->
   <el-col :span="12">
    <el-card shadow="never" >
     <div class="card-title">
      <el-text><b>智能分析</b></el-text>
      <el-divider></el-divider>
     </div>
     <el-form class="leftForm" :model="formData" rules="rules" label-width="160px" label-position="left">

      <el-form-item class="chartGoal" label="分析目标：">
       <el-input v-model="formData.goal" type="textarea" />
      </el-form-item>

      <el-form-item class="chartName" label="图表名称：">
       <el-input v-model="formData.chartName" />
      </el-form-item>

      <el-form-item class="chartType" label="图表类型：">
       <el-select v-model="formData.chartType" placeholder="请选择图表类型">
        <el-option v-for="(option, index) in options" :key="index" :label="option.label" :value="option.value">
        </el-option>
       </el-select>
      </el-form-item>

      <el-form-item class="uploadExcel" label="上传excel数据文件：">
       <el-upload
        v-model:file-list="fileList"
        action=""
        :http-request="uploadFile"
        :limit="1"
       >

        <el-button type="primary" class="uploadButton">
         <el-icon class="el-icon--upload">
          <i-ep-upload-filled />
         </el-icon>
         点击上传
        </el-button>
        <div class="el-upload__remind">只允许上传xls/xlsx files</div>
        <template #tip>
         <div class="el-upload__tip">

         </div>
        </template>
       </el-upload>

      </el-form-item>

      <el-form-item>
       <el-button class="submitbut" type="primary" @click="submitData" >提交</el-button>
       <el-button @click="resetForm">重置</el-button>
      </el-form-item>

     </el-form>
    </el-card>
   </el-col>

   <!--分析结果-->
   <el-col :span="12">
    <el-row class="right-row" justify="space-between">
     <!--可视化图表-->
     <el-col :style="{marginBottom: '10px'}">
      <el-card shadow="never" :style="{height: '150px'}">
       <div class="card-title">
        <el-text><b>可视化图表</b></el-text>
        <el-divider></el-divider>
       </div>
       <div class="genChart">
        <el-text>请先再左侧进行提交</el-text>
       </div>
      </el-card>
     </el-col>

     <!--分析结论-->
     <el-col :style="{marginTop: '10px'}">
      <el-card shadow="never" :style="{height: '150px'}">
       <div class="card-title">
        <el-text><b>分析结论</b></el-text>
        <el-divider></el-divider>
       </div>
       <div class="genResult">
        <el-text>请先再左侧进行提交</el-text>
       </div>
      </el-card>
     </el-col>
    </el-row>
   </el-col>
  </el-row>

 </div>
</template>

<style lang="scss" scoped>
.app-container {
 margin-left: 30px;
  margin-right: 30px;
 margin-top: 20px;
}

.card-title{
 margin-top: 5px;
}

.leftForm{
 height: 470px;
}

.chartGoal{
  margin-bottom: 30px;
}

.chartName{
  margin-bottom: 30px;
}

.chartType{
  margin-bottom: 30px;
}

.uploadExcel{
 margin-bottom: 20px;
}


.uploadButton{
  border: 1px solid grey;
 color: black;
 background-color: white;
 width: 125px;
}

.el-icon--upload{
 margin-right: 12px;
}

.el-upload__remind{
  width: 150px; // 设置宽度
 font-size: 13px;
 margin-left: 10px;
}

.el-upload__tip {
  width: 150px; // 设置宽度
 margin-top: -10px;
}

</style>
```

### 4.4 使用echart数据可视化

#### 4.4.1 安装导入echart库

1. 官方echart数据可视化安装

官方：[Apache ECharts](https://echarts.apache.org/zh/index.html)

安装 echarts：

注意不要在项目目录执行，会报错

```javascript
npm install echarts --save
```

2. vue3中使用echarts的配置

```tsx
// 按需引入echarts图(这里引入的是折线图)
import * as echarts from 'echarts/core';
import { GridComponent, GridComponentOption } from 'echarts/components';
import { LineChart, LineSeriesOption } from 'echarts/charts';
import { UniversalTransition } from 'echarts/features';
import { CanvasRenderer } from 'echarts/renderers';

echarts.use([GridComponent, LineChart, CanvasRenderer, UniversalTransition]);

type EChartsOption = echarts.ComposeOption<
	GridComponentOption | LineSeriesOption
>;
```

#### 4.4.2 使用echart库

1. 创建一个子组件charts.vue，用来将数据传给父组件，生成图表

```vue
<template>
    <div :style="{ height: chartHeight, width: '100%' }" ref="chart"></div>
</template>
<script lang='ts' setup>
import { ref, onMounted, onBeforeUnmount, markRaw } from 'vue'
import * as echarts from 'echarts/core'


const chart = ref<HTMLDivElement>()

const myChart = ref()
// 接受父组件传过来的option，和echarts的高度

// 可以根据父组件传过来的option对象生成折线图、柱状图、饼图等等。
const props = defineProps(['option', 'chartHeight'])

onMounted(() => {
    // 函数体
    // console.log(props.option);
    // ！！！这里必须用markRaw包裹住，否则当页面宽度变化控制台会报错
    myChart.value = markRaw(echarts.init(chart.value as HTMLDivElement))

    myChart.value.setOption(props.option)
    // 监听页面视图变化echarts图的宽度变化
    window.addEventListener("resize", () => {
        myChart.value.resize()
    })
})

// 组件销毁前一定要取消监听的事情，不然会印象性能和暂用内存
onBeforeUnmount(() => {
    window.removeEventListener("resize", () => {
        myChart.value.resize()
    })
})
</script>
 
<style scoped></style>
```

2. 在index.vue使用该组件，并且放在响应的位置，我还设置了组件没有传递数据的情况下显示“请先在左侧进行提交”

```vue
<script setup>
// 按需引入echarts图(这里引入的是折线图)
import * as echarts from 'echarts/core';
import { GridComponent, GridComponentOption } from 'echarts/components';
import { LineChart, LineSeriesOption } from 'echarts/charts';
import { UniversalTransition } from 'echarts/features';
import { CanvasRenderer } from 'echarts/renderers';

echarts.use([GridComponent, LineChart, CanvasRenderer, UniversalTransition]);

type EChartsOption = echarts.ComposeOption<
	GridComponentOption | LineSeriesOption
>;

const option = ref<EChartsOption>({
	"title": {
		"text": "网站用户增长情况"
	},
	"xAxis": {
		"type": "category",
		"data": ["1号", "2号", "3号"]
	},
	"yAxis": {
		"type": "value"
	},
	"series": [{
		"data": [10, 20, 30],
		"type": "line"
	}]
})

/**
 * 如果option有数据，返回true，展示图表
 */
 const showElCharts = computed(() => {
  return option.value.series
})

/**
 * 如果option没有数据，返回true，显示"请先在左侧进行提交"
 */
const notShowElCharts = computed(() => {
	return option.value.series ? '' :  '请先在左侧进行提交'
})
</script>
<template>
	<div class="app-container">
			<!--分析结果-->
			<el-col :span="12">
				<el-row class="right-row" justify="space-between">
					<!--可视化图表-->
					<el-col :style="{ marginBottom: '10px' }">
						<el-card shadow="never" :style="{height: showElCharts ? '' : '150px'}">
							<div class="card-title">
								<el-text><b>可视化图表</b></el-text>
								<el-divider></el-divider>
							</div>
							<div class="genChart">
								<Charts :option="option" chartHeight="300px" v-if="showElCharts" />
								<el-text v-else>{{ notShowElCharts }}</el-text>
							</div>

						</el-card>
					</el-col>

					<!--分析结论-->
					<el-col :style="{ marginTop: '10px' }">
						<el-card shadow="never" :style="{ height: '150px' }">
							<div class="card-title">
								<el-text><b>分析结论</b></el-text>
								<el-divider></el-divider>
							</div>
							<div class="genResult">
								<el-text v-if="showElCharts">{{ showGenResult }}</el-text>
								<el-text v-else>请先再左侧进行提交</el-text>
							</div>
						</el-card>
					</el-col>
				</el-row>
			</el-col>
		</el-row>

	</div>
</template>
```

### 4.5 将数据请求响应后显示在界面

#### 4.5.1 定义为响应式并展示

```vue
/**
 * 可视化图表
 * 数据后面是发起请求后，拿到响应的数据，传到这里
 */
const option = ref<EChartsOption>({

})

/**
 * 如果option有数据，返回true，展示图表
 */
 const showElCharts = computed(() => {
  return option.value.series
})

/**
 * 如果option没有数据，返回true，显示"请先在左侧进行提交"
 */
const notShowElCharts = computed(() => {
 return option.value.series ? '' :  '请先在左侧进行提交'
})

/**
 * 分析结论
 * 数据后面是发起请求后，拿到响应的数据，传到这里
 */
const showGenResult = ref<BiResult>({})
```

这里要注意，option.value.series才是展现数据的

提交按钮添加成功响应改变图表option和showGenResult的数据

```tsx
/**
 * 发起智能分析
 */
function submitData() {
 // 检查上传文件列表中是否有文件
 const multipartFile = fileList.value[0]?.raw;
 if (!multipartFile) {
  ElMessage.warning("上传Excel文件不能为空");
  return false;
 }
 // 提交表单数据
 console.log('表单数据', formData, fileList.value);

 genChartByAi(multipartFile, formData)
  .then(res => {
   console.log('[submitData] 分析成功', res);
   ElMessage.success("分析成功")

   const data = JSON.parse(res.data.genChart);
   option.value = data;

   // 更新showGenResult
   showGenResult.value = res.data.genResult;

   console.log(option)
   console.log(showGenResult)
  })
  .catch(err => {
   console.error('[submitData] 请求失败', err);
   ElMessage.warning("分析失败：", err);
  });
}
```

这里注意，因为我们后端传过来的数据是String类型，想要更新数据，就需要转换数据为JSON格式

#### 4.5.2 优化图表能够实时更新

在子组件中添加监听器，监听数据实时变化，同时变化图表，并且全局引入echart

```vue
<template>
	<div :style="{ height: chartHeight, width: '100%' }" ref="chart"></div>
</template>

<script lang='ts' setup>
import { ref, onMounted, onBeforeUnmount, watchEffect, defineProps, markRaw } from 'vue'
import * as echarts from 'echarts';
/*
import * as echarts from 'echarts/core'

// 为选用的图表类型引入必须的组件
import { LineChart, BarChart, PieChart, RadarChart } from 'echarts/charts'
import { GridComponent, TooltipComponent, LegendComponent } from 'echarts/components'
import { CanvasRenderer } from 'echarts/renderers';
*/

// 在echarts中注册使用的组件
/*
echarts.use([LineChart, BarChart, PieChart, RadarChart,
	GridComponent, TooltipComponent, LegendComponent, CanvasRenderer]);
*/


const chart = ref<HTMLDivElement>()
const chartInstance = ref<echarts.ECharts>()
const props = defineProps({
	option: Object,
	chartHeight: {
		type: String,
		default: '300px'
	}
})

onMounted(() => {
	chartInstance.value = echarts.init(chart.value as HTMLDivElement);
	chartInstance.value.setOption(markRaw(props.option));
	window.addEventListener('resize', () => {
		chartInstance.value.resize();
	})
})

// 在选择的option数据发生变化时，将数据同步到echarts实例中
watchEffect(() => {
	if (chartInstance.value) {
		chartInstance.value.setOption(markRaw(props.option));
	}
})

onBeforeUnmount(() => {
	window.removeEventListener('resize', () => {
		chartInstance.value.resize();
	})
})
</script>

<style scoped></style>

```

更改option对应内容

```
/**
 * 可视化图表
 * 数据后面是发起请求后，拿到响应的数据，传到这里
 */
const option = markRaw(ref<EChartsOption>({}));
```

`markRaw` 是 Vue3 中的 API，用来标记一个或多个对象，并告诉 Vue3 不要对其进行响应式处理，也就是将其标记为原始值。

在传递复杂对象时，如果对象每次都会触发响应式更新，可能会降低性能或导致性能不稳定，甚至有时会导致无限循环。这时可以使用 `markRaw` 来优化代码性能。

在你的代码中，`option` 使用了 `ref`，因此在外部使用时，需要通过调用 `value` 来获取其值。所以，在使用 `markRaw` 时，需要将其作用于 `option.value`。

#### 4.5.3 提交按钮添加进度条

1. 使用element plus的进度条组件，顺便重新设置一下重置表单按钮

```tsx
/**
 * 重置表单
 */
function resetForm() {
 // 重置表单数据和上传文件列表
 formData.goal = "";
 formData.chartName = "";
 formData.chartType = "";
 fileList.value = [];
 percentage.value =0;
 option.value = {};
};

/**
 * 进度条
 */
const percentage = ref()
const customColor = ref('#409eff')
```

2. 在提交后，实行进度条变动

使用const timer = setInterval(() => {},300)，当到达100的时候执行clearInterval(timer);清空计时器

```tsx
/**
 * 发起智能分析
 */
function submitData() {
 // 检查上传文件列表中是否有文件
 const multipartFile = fileList.value[0]?.raw;
 if (!multipartFile) {
  ElMessage.warning("上传Excel文件不能为空");
  return false;
 }
 // 提交表单数据
 console.log('表单数据', formData, fileList.value);

 // 清空图表
 option.value = {};

 // 设置进度条初始值为0
 percentage.value = 0;

 let count = 0;
 const timer = setInterval(() => {
  if (count < 95) {
   percentage.value = ++count;
  } else if (count == 95){
   percentage.value = 95;
  } else {
   clearInterval(timer);
   percentage.value = 100;
  }
 }, 300);



 genChartByAi(multipartFile, formData)
   .then(res => {
    // 设置进度条为100%
    clearInterval(timer);
    percentage.value = 100

    console.log('[submitData] 分析成功', res);
    ElMessage.success("分析成功")

    const data = JSON.parse(res.data.genChart);
    option.value = data;

    // 更新showGenResult
    showGenResult.value = res.data.genResult;

    console.log(option)
    console.log(showGenResult)
   })
   .catch(err => {
    console.error('[submitData] 请求失败', err);
    ElMessage.warning("分析失败：", err);
    percentage.value = 0
   });
}
```

最后我这里面还进行了一些css的调整，看起来好看些，就这样啦~

```vue
<script setup lang="ts">
import {ref, reactive} from 'vue'
import {UploadUserFile} from "element-plus";
import {BiForm, BiResult} from "@/api/bi/types";
import {genChartByAi, uploadFileOss} from "@/api/bi/index";
import Charts from "./charts.vue"

// 按需引入echarts图(这里引入的是折线图)
import * as echarts from 'echarts/core';
import {GridComponent, GridComponentOption} from 'echarts/components';
import {LineChart, LineSeriesOption} from 'echarts/charts';
import {UniversalTransition} from 'echarts/features';
import {CanvasRenderer} from 'echarts/renderers';

echarts.use([GridComponent, LineChart, CanvasRenderer, UniversalTransition]);

type EChartsOption = echarts.ComposeOption<GridComponentOption | LineSeriesOption>;


/**
 * 可视化图表
 * 数据后面是发起请求后，拿到响应的数据，传到这里
 */
const option = ref<EChartsOption>({})
//const option = markRaw(ref<EChartsOption>({}));

/**
 * 如果option有数据，返回true，展示图表
 */
const showElCharts = computed(() => {
	return option.value.series
})

/**
 * 如果option没有数据，返回true，显示"请先在左侧进行提交"
 */
const notShowElCharts = computed(() => {
	return option.value.series ? '' : '请先在左侧进行提交'
})

/**
 * 分析结论
 * 数据后面是发起请求后，拿到响应的数据，传到这里
 */
const showGenResult = ref<BiResult>({})

/**
 * 文件
 */
const fileList = ref<UploadUserFile[]>([]);

/**
 * 表单
 */
const formData = reactive<BiForm>({goal: '', chartName: '', chartType: ''});


/**
 * 表单规则
 */
const rules = reactive({
	goal: [{required: true, message: "分析目标不能为空", trigger: "blur"}],
	chartName: [{required: true, message: "图表名称不能为空", trigger: "blur"}],
	chartType: [{required: true, message: "图表类型不能为空", trigger: "blur"}],
});

/**
 * 图表类型种类
 */
const chartTypes: { value: string, label: string }[] = [
	{value: '饼图', label: '饼图'},
	{value: '地图', label: '地图'},
	{value: '折线图', label: '折线图'},
	{value: '柱状图', label: '柱状图'},
	{value: '条形图', label: '条形图'},
	{value: '散点图', label: '散点图'},
	{value: 'K线图', label: 'K线图'},
];

/**
 * 重置表单
 */
function resetForm() {
	// 重置表单数据和上传文件列表
	formData.goal = "";
	formData.chartName = "";
	formData.chartType = "";
	fileList.value = [];
	percentage.value =0;
	option.value = {};
};

/**
 * 进度条
 */
const percentage = ref()
const customColor = ref('#409eff')

/**
 * 上传文件
 */
function uploadFile(params: { file: any; }) {
	const file = params.file;
	if (!file) {
		ElMessage.warning("上传Excel文件不能为空");
		return false;
	}
	if (!/\.xlsx|\.xls|\.XLSX|\.XLS$/i.test(file.name)) {
		ElMessage.warning("上传Excel只能为xlsx、xls格式");
		fileList.value = [];
		return false;
	}
	uploadFileOss(file)
			.then(res => {
				console.log('[uploadFile] 上传成功', res);
			})
			.catch(err => {
				console.error('[uploadFile] 上传失败', err);
			});
}


/**
 * 发起智能分析
 */
function submitData() {
	// 检查上传文件列表中是否有文件
	const multipartFile = fileList.value[0]?.raw;
	if (!multipartFile) {
		ElMessage.warning("上传Excel文件不能为空");
		return false;
	}
	// 提交表单数据
	console.log('表单数据', formData, fileList.value);

	// 清空图表
	option.value = {};

	// 设置进度条初始值为0
	percentage.value = 0;

	let count = 0;
	const timer = setInterval(() => {
		if (count < 95) {
			percentage.value = ++count;
		} else if (count == 95){
			percentage.value = 95;
		} else {
			clearInterval(timer);
			percentage.value = 100;
		}
	}, 300);



	genChartByAi(multipartFile, formData)
			.then(res => {
				// 设置进度条为100%
				clearInterval(timer);
				percentage.value = 100

				console.log('[submitData] 分析成功', res);
				ElMessage.success("分析成功")

				const data = JSON.parse(res.data.genChart);
				option.value = data;

				// 更新showGenResult
				showGenResult.value = res.data.genResult;

				console.log(option)
				console.log(showGenResult)
			})
			.catch(err => {
				console.error('[submitData] 请求失败', err);
				ElMessage.warning("分析失败：", err);
				percentage.value = 0
			});
}

</script>

<template>
	<div class="app-container">
		<el-row :gutter="30">
			<!--智能分析表单-->
			<el-col :span="12">
				<el-card shadow="never">
					<div class="card-title">
						<el-text><b>智能分析</b></el-text>
						<el-divider></el-divider>
					</div>
					<el-form class="leftForm" :model="formData" rules="rules" label-width="160px" label-position="left">

						<el-form-item class="chartGoal" label="分析目标：">
							<el-input v-model="formData.goal" type="textarea"/>
						</el-form-item>

						<el-form-item class="chartName" label="图表名称：">
							<el-input v-model="formData.chartName"/>
						</el-form-item>

						<el-form-item class="chartType" label="图表类型：">
							<el-select v-model="formData.chartType" placeholder="请选择图表类型">
								<el-option v-for="(chartType, index) in chartTypes" :key="index" :label="chartType.label"
													 :value="chartType.value">
								</el-option>
							</el-select>
						</el-form-item>

						<el-form-item class="uploadExcel" label="上传excel数据文件：">
							<el-upload v-model:file-list="fileList"
												 action=""
												 :http-request="uploadFile"
												 :limit="1"
												 :on-progress="uploadVideoProcess">

								<el-button type="primary" class="uploadButton">
									<el-icon class="el-icon--upload">
										<i-ep-upload-filled/>
									</el-icon>
									点击上传
								</el-button>
								<div class="el-upload__remind">只允许上传xls/xlsx files</div>
								<template #tip>
									<div class="el-upload__tip">

									</div>
								</template>
							</el-upload>

						</el-form-item>

						<el-form-item>
							<el-button class="submitbut" type="primary" @click="submitData">提交</el-button>
							<el-button @click="resetForm">重置</el-button>
						</el-form-item>
						<div class="progress">
							<el-progress :percentage="percentage" :color="customColor"/>
						</div>
					</el-form>
				</el-card>
			</el-col>

			<!--分析结果-->
			<el-col :span="12">
				<el-row class="right-row" justify="space-between">
					<!--可视化图表-->
					<el-col :style="{ marginBottom: '10px' }">
						<el-card shadow="never" :style="{height: showElCharts ? '' : '150px'}">
							<div class="card-title">
								<el-text><b>可视化图表</b></el-text>
								<el-divider></el-divider>
							</div>
							<div class="genChart">
								<Charts :option="option" chartHeight="400px" v-if="showElCharts"/>
								<el-text v-else>{{ notShowElCharts }}</el-text>
							</div>

						</el-card>
					</el-col>

					<!--分析结论-->
					<el-col :style="{ marginTop: '10px' }">
						<el-card shadow="never" :style="{ height: showElCharts ? '' : '150px' }">
							<div class="card-title">
								<el-text><b>分析结论</b></el-text>
								<el-divider></el-divider>
							</div>
							<div class="genResult">
								<el-text v-if="showElCharts">{{ showGenResult }}</el-text>
								<el-text v-else>请先再左侧进行提交</el-text>
							</div>
						</el-card>
					</el-col>
				</el-row>
			</el-col>
		</el-row>

	</div>
</template>

<style lang="scss" scoped>
.app-container {
	margin-left: 30px;
	margin-right: 30px;
	margin-top: 20px;
}

.row {
	display: flex;
	flex-direction: column;
	height: 100%;
}

.analysis-result-card {
	height: 100%;
	display: flex;
}

.progress {
}

.demo-progress .el-progress--circle {
	margin-right: 15px;
}

.genChart,
.genResult {
	flex: 1;
	overflow-y: auto;
}

.el-card-title {
	margin-top: 5px;
	padding: 20px;
}


.card-title {
	margin-top: 5px;
}

.leftForm {
	height: 515px;
}

.chartGoal {
	margin-bottom: 30px;
}

.chartName {
	margin-bottom: 30px;
}

.chartType {
	margin-bottom: 30px;
}

.uploadExcel {
	margin-bottom: 20px;
}

.uploadButton {
	border: 1px solid grey;
	color: black;
	background-color: white;
	width: 125px;
}

.el-icon--upload {
	margin-right: 12px;
}

.el-upload__remind {
	width: 150px; // 设置宽度
	font-size: 13px;
	margin-left: 10px;
}

.el-upload__tip {
	width: 150px; // 设置宽度
	margin-top: -10px;
}
</style>


```

## 5、优化后端代码

### 5.1 提取验证ValidationUtil类

将前面一大段验证信息抽出来，如果在其他地方需要使用这个验证，就可以直接复用

```
GenChartByAiRequest genChartByAiRequest = JSONUtil.parseObj(genChartByAiRequestJson).toBean(GenChartByAiRequest.class);
String chartName = genChartByAiRequest.getChartName();
String goal = genChartByAiRequest.getGoal();
String chartType = genChartByAiRequest.getChartType();
// 校验
ThrowUtils.throwIf(StringUtils.isBlank(chartType), ErrorCode.PARAMS_ERROR, "分析类型为空");
ThrowUtils.throwIf(StringUtils.isBlank(goal), ErrorCode.PARAMS_ERROR, "分析目标为空");
ThrowUtils.throwIf(StringUtils.isNotBlank(chartName) && chartName.length() > 100 , ErrorCode.PARAMS_ERROR, "图表名称过长");
// 校验文件
long size = multipartFile.getSize();
String originalFilename = multipartFile.getOriginalFilename();
// 校验文件大小
final long ONE_MB = CommonConstant.ONE_MB;
ThrowUtils.throwIf(size > ONE_MB, ErrorCode.PARAMS_ERROR, "文件超过 1M");
// 校验文件后缀 aaa.png
String suffix = FileUtil.getSuffix(originalFilename);
final List<String> validFileSuffixList = Arrays.asList("xlsx");
ThrowUtils.throwIf(!validFileSuffixList.contains(suffix), ErrorCode.PARAMS_ERROR, "文件后缀非法");
```

提取为ValidationUtil

```java
public class ValidationUtil {

    private static final int CHART_NAME_LEN = 100;

    public static void validateGenChartByAiRequest(GenChartByAiRequest genChartByAiRequest) {
        if (StringUtils.isBlank(genChartByAiRequest.getChartType())) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "分析类型不能为空");
        }

        if (StringUtils.isBlank(genChartByAiRequest.getGoal())) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "分析目标不能为空");
        }

        if (StringUtils.isNotBlank(genChartByAiRequest.getChartName())
                && genChartByAiRequest.getChartName().length() > CHART_NAME_LEN) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "图表名称过长");
        }
    }

    public static void validateFile(MultipartFile multipartFile, long maxSize, List<String> validFileSuffixList) {
        if (multipartFile.getSize() > maxSize) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "文件大小超出限制");
        }

        String originalFilename = multipartFile.getOriginalFilename();
        String suffix = FileUtil.getSuffix(originalFilename);
        if (!validFileSuffixList.contains(suffix)) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "文件类型不符合要求");
        }
    }
}
```

### 5.2 提取处理数据的工具类

因为我们处理图表数据的格式是一样的，拼接的格式也是一样的，为了避免重复的使用，提取出来

```java
public class BiChartUtils {

    /**
     * 拼接用户输入的信息
     * @param goal
     * @param chartType
     * @param multipartFile
     * @return
     */
    public static String getUserInput(String goal, String chartType, MultipartFile multipartFile) {
        StringBuilder userInput = new StringBuilder();
        userInput.append("分析需求：").append("\n");

        // 拼接分析目标
        String userGoal = goal;
        if (StringUtils.isNotBlank(chartType)) {
            userGoal += "，请使用" + chartType;
        }
        userInput.append(userGoal).append("\n");
        userInput.append("原始数据：").append("\n");

        // 压缩后的数据
        String csvData = ExcelUtils.excelToCsv(multipartFile);
        userInput.append(csvData).append("\n");

        return userInput.toString();
    }

    /**
     * 构建BiChart信息
     * @param chartName
     * @param goal
     * @param chartType
     * @param csvData
     * @param genChart
     * @param genResult
     * @param loginUser
     * @return
     */
    public static BiChart getBiChart(String chartName, String goal, String chartType, String csvData, String genChart, String genResult, UserInfoVO loginUser) {
        BiChart chart = new BiChart();
        chart.setChartName(chartName);
        chart.setGoal(goal);
        chart.setChartData(csvData);
        chart.setChartType(chartType);
        chart.setGenChart(genChart);
        chart.setGenResult(genResult);
        chart.setUserId(loginUser.getUserId());
        return chart;
    }
}
```

### 5.3 提取常量，添加可用性

```java
public interface ChartConstant {
    /**
     * AI生成的内容分隔符
     */
    String GEN_CONTENT_SPLITS = "【【【【【";

    /**
     * AI 生成的内容的元素为3个
     */
    int GEN_ITEM_NUM = 3;

    /**
     * 生成图表的数据下标
     */
    int GEN_CHART_IDX = 1;

    /**
     * 生成图表的分析结果的下标
     */
    int GEN_RESULT_IDX = 2;

    /**
     * 文件上传的大小限制1M
     */
    long File_SIZE_ONE = 1024 * 1024L;

    /**
     * 图表上传文件后缀白名单
     */
    List<String> VALID_FILE_SUFFIX= Arrays.asList("xlsx","csv","xls","json");
}
```

### 5.4 基本优化完毕

Controller类如下：

```java
@Operation(summary = "AI智能分析", security = {@SecurityRequirement(name = "Authorization")})
    @PostMapping("/gen")
        public Result<BiResponseVo> genChartByAi(@RequestPart("file") MultipartFile multipartFile,
                                                 @RequestParam("genChartByAiRequest") String genChartByAiRequestJson) {
        GenChartByAiRequest genChartByAiRequest = JSONUtil.parseObj(genChartByAiRequestJson).toBean(GenChartByAiRequest.class);

        ValidationUtil.validateGenChartByAiRequest(genChartByAiRequest);
        /**
         * 三个参数：文件，文件最大大小，后缀限制
         */
        ValidationUtil.validateFile(multipartFile, File_SIZE_ONE, VALID_FILE_SUFFIX);

        BiResponseVo biResponse = chartService.getBiResponse(genChartByAiRequest,multipartFile);
        ThrowUtils.throwIf(biResponse == null, ErrorCode.SYSTEM_ERROR, "AI生成错误");
        return Result.success(biResponse);
    }
```

BiChartService类

```java
public interface BiChartService extends IService<BiChart> {

    /**
     * 拿到AI响应的数据
     * @param genChartByAiRequest
     * @param multipartFile
     * @return
     */
    BiResponseVo getBiResponse(GenChartByAiRequest genChartByAiRequest, MultipartFile multipartFile);
}
```

BiChartServiceImpl类

```java
@Service
public class BiChartServiceImpl extends ServiceImpl<BiChartMapper, BiChart> implements BiChartService {

    @Resource
    private SysUserService userService;

    @Resource
    private RedisLimiterManager redisLimiterManager;

    @Resource
    private BiChartService chartService;

    @Resource
    private AiManager aiManager;

    /**
     * 拿到AI响应的数据
     * @param genChartByAiRequest
     * @param multipartFile
     * @return BiResponseVo
     */
    @Override
    public BiResponseVo getBiResponse(GenChartByAiRequest genChartByAiRequest, MultipartFile multipartFile) {
        String chartName = genChartByAiRequest.getChartName();
        String goal = genChartByAiRequest.getGoal();
        String chartType = genChartByAiRequest.getChartType();

        UserInfoVO loginUser = userService.getUserLoginInfo();
        // 限流判断，每个用户一个限流器
        redisLimiterManager.doRateLimit("genChartByAi_" + loginUser.getUserId());

        long biModelId = CommonConstant.BI_MODEL_ID;

        // 构造用户输入
        String userInput = BiChartUtils.getUserInput(goal, chartType, multipartFile);

        // 处理数据
        String chartResult = aiManager.doChat(biModelId, userInput);
        // 解析内容
        String[] splits = chartResult.split(GEN_CONTENT_SPLITS);
        if (splits.length < GEN_ITEM_NUM) {
            throw new BusinessException(ErrorCode.SYSTEM_ERROR, "AI生成错误");
        }
        // 首次生成的内容
        String genChart = splits[GEN_CHART_IDX].trim();
        String genResult = splits[GEN_RESULT_IDX].trim();

        // 插入到数据库
        String csvData = ExcelUtils.excelToCsv(multipartFile);
        BiChart chart = BiChartUtils.getBiChart(chartName, goal, chartType, csvData, genChart, genResult, loginUser);
        boolean saveResult = chartService.save(chart);
        ThrowUtils.throwIf(!saveResult, ErrorCode.SYSTEM_ERROR, "图表保存失败");

        BiResponseVo biResponse = new BiResponseVo();
        biResponse.setGenChart(genChart);
        biResponse.setGenResult(genResult);
        biResponse.setChartId(chart.getId());

        return biResponse;
    }
}
```

优化完毕~~~



## 6. 优化后运行程序出现bug

![image-20230715113437952](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230715113437952.png)

翻译过来就是：

这个错误提示是因为程序中Controller和Service之间形成了循环依赖，即BiChartServiceImpl中使用了ChartController中的Bean，而ChartController中又使用了BiChartServiceImpl中的Bean，因此Spring容器无法完成Bean的实例化。

我想了一下，应该是使用mp导致的，这样spring就不知道先加载哪个bean

导致的原因，我看到一篇文章，跟我想的一样：[关于Springboot+MybatisPlus架构循环依赖问题研究](https://blog.csdn.net/qq_36381800/article/details/119536486)



解决方案一：（不推荐）

如果无法避免循环依赖，可以将Spring的循环引用检测设置为允许。在application.properties或application.yml文件中添加如下配置：

```
spring.main.allow-circular-references=true
```

解决方案二：（推荐）

 使用@Lazy注解

```
@Lazy
@Resource
private BiChartService chartService;
```

这样就解决了
