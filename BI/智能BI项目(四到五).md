# 智能分析业务开发（四）

## 1、添加我的图表功能

### 1.1 前端页面设计

经过开发上一个页面的经验，我发现要首先搭好界面，想好基础模块，不然后面再来穿插会非常的麻烦，所以需要先做好最基本的页面模型，再来一步步开发会比较轻松

我这里使用Echart换了一种方式实现，参考链接[vue3+ts使用Echart](https://blog.csdn.net/m0_59006402/article/details/131005915?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EYuanLiJiHua%7EPosition-2-131005915-blog-131228441.235%5Ev38%5Epc_relevant_anti_t3&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EYuanLiJiHua%7EPosition-2-131005915-blog-131228441.235%5Ev38%5Epc_relevant_anti_t3&utm_relevant_index=3)

这种方式更好，把Echart放在component文件夹里，比较舒服

```vue
<template>
 <div class="app-container">
  <div class="input-div">
   <el-input
    v-model="queryParams.chartName"
    placeholder="请输入图表名称"
    class="input-select"
   >
    <template #append>
     <el-button type="primary" @click="handleQuery">
      <i-ep-search />
      搜索
     </el-button>
    </template>
   </el-input>
  </div>
  <el-row :gutter="15">
   <el-col class="col"
       :span="12"
       v-for="(o, index) in 4 "
       :key="o"
       :offset="index > 0 ? 2 : 0">
    <el-card class="card">
     <div class="card-title">
      <div>
       <img class="card-logo" src="src/assets/paperfly-logo.png">
      </div>
      <div class="card-text">
       <span class="card-text1">纸飞机网站分析图</span>
       <span class="card-text2">图表类型：{{ 图表类型 }}</span>
      </div>
     </div>
     <div>
      <span class="card-text3">分析目标：{{ 分析目标 }}</span>
      <Echarts class="myEcharts" :options="option" :width="'100%'" :height="'300px'" />
     </div>
    </el-card>
   </el-col>
  </el-row>

  <pagination
   class="page"
   v-if="total >= 0"
   v-model:total="total"
   v-model:page="queryParams.pageNum"
   v-model:limit="queryParams.pageSize"
   v-model:page-sizes="pageSizes"
   @pagination="handleQuery"
  />

 </div>
</template>

<style lang="scss" scoped>
.app-container {
 margin-left: 30px;
 margin-right: 30px;
}

.input-select {
 margin-bottom: 10px;

}

.col {
 margin-left: 0;
 margin-bottom: 15px;
}

.card {
 margin-left: 0;
}

.card-title {
 height: 50px;
 display: flex;
 flex-direction: row;
}

.card-logo {
 width: 40px;
 height: 40px;
}

.card-text {
 display: flex;
 flex-direction: column;
 justify-content: flex-start;
 margin-left: 10px;
}

.card-text1 {
 font-size: 10px;
 margin-top: 1px;
}

.card-text2 {
 font-size: 5px;
 align-self: flex-start;
 margin-top: 4px;
 color: rgba(128, 126, 126, 0.8)
}

.card-text3{
}

.myEcharts{
 margin-top: 20px;
}

.page {
 margin-top: -15px;
}
</style>
```

### 1.2 前端页面参数类型设置

首先想好这个页面会展示哪些数据，就在type.ts里面定义好

```ts
export interface ChartPageVO {

 /**
  * 图表名称
  */
 chartName?: string;

 /**
  * 图表类型
  */
 chartType?: string;

 /**
  * 分析目标
  */
 goal?: string;

 /**
  * 生成的echart所需的数据
  */
 genChart?: string;

 /**
  * 创建时间
  */
 createTime?: Date;
}

export interface ChartQuery extends PageQuery{
	chartName?: string;
	chartType?: string;
	genChart?: string;
}
```

在页面上定义好对应的类型

```vue
<script setup lang="ts">
   
const options = ref<any>(null);
const loading = ref(false);
const total = ref(1);
const queryParams = reactive<ChartQuery>({
	pageNum: 1,
	pageSize: 4,
});
const pageSizes = [4, 8, 12, 16, 20, 24];
const chartList = ref<ChartPageVO[]>(
	[]
);
</script>
```

其实我一开始是先用假数据把页面做好，然后再一步步把所需要的数据类型放上去，再去完成函数等功能，看自己的步骤把，我这里没有记录一步步实现的过程，但是还是建议大家自己做一下，感觉才比较好

### 1.3 后端分页查询的实现

首先定义分页查询所需要数据VO，用来在控制层接收数据类型

```java
@Schema(description ="图表分页对象")
@Data
public class ChartPageVo {

    /**
     * id
     */
    @Schema(description="图表ID")
    private Long id;

    /**
     * 分析目标
     */
    @Schema(description="分析目标")
    private String goal;

    /**
     * 图表名称
     */
    @Schema(description="图表名称")
    private String chartName;

    /**
     * 图表信息
     */
    @Schema(description="图表信息")
    private String chartData;

    /**
     * 图表类型
     */
    @TableField(value = "图表类型")
    private String chartType;

    /**
     * 生成的图表信息
     */
    @Schema(description="生成的图表信息")
    private String genChart;

    /**
     * 生成的分析结论
     */
    @Schema(description="生成的分析结论")
    private String genResult;

    /**
     * 创建图标用户 id
     */
    @Schema(description="创建图标用户 id")
    private Long userId;

    /**
     * 是否删除
     */
    @Schema(description="是否删除")
    private Integer deleted;
}
```

创建controller层对应的方法

```java
/**
 * 用户查询记录分页列表
 * @param chartQueryRequest
 * @return
 */
@Operation(summary = "用户查询记录分页列表", security = {@SecurityRequirement(name = "Authorization")})
@GetMapping("/page")
public PageResult<BiChart> getChartPage( @ParameterObject ChartQueryRequest chartQueryRequest) {
    int pageNum = chartQueryRequest.getPageNum();
    int pageSize = chartQueryRequest.getPageSize();

    Page<BiChart> result = chartService.page(new Page<>(pageNum, pageSize), getQueryWrapper(chartQueryRequest));

    return PageResult.success(result);
}
```

然后这里的getQueryWrapper单独封装成一个方法放在同一个类下面

```java
/**
 * 获取查询包装类
 *
 * @param chartQueryRequest
 * @return
 */
private QueryWrapper<BiChart> getQueryWrapper(ChartQueryRequest chartQueryRequest) {
    QueryWrapper<BiChart> queryWrapper = new QueryWrapper<>();
    if (chartQueryRequest == null) {
        return queryWrapper;
    }
    Long id = chartQueryRequest.getId();
    String chartName = chartQueryRequest.getChartName();
    String goal = chartQueryRequest.getGoal();
    String chartType = chartQueryRequest.getChartType();
    Long userId = chartQueryRequest.getUserId();

    // 声明排序字段和排序方式
    String sortField = "create_time";
    String sortOrder = CommonConstant.SORT_ORDER_ASC;

    queryWrapper.eq(id != null && id > 0, "id", id);
    queryWrapper.like(StringUtils.isNotBlank(chartName), "chartName", chartName);
    queryWrapper.eq(StringUtils.isNotBlank(goal), "goal", goal);
    queryWrapper.eq(StringUtils.isNotBlank(chartType), "chartType", chartType);
    queryWrapper.eq(ObjectUtils.isNotEmpty(userId), "userId", userId);

    // 添加排序条件
    queryWrapper.orderBy(true, sortOrder == CommonConstant.SORT_ORDER_DESC, sortField);

    return queryWrapper;
}
```

### 1.4 前端页面完善



写好发起请求的方法，写在index.ts里面

```ts
/**
 * 获取图表分页列表
 *
 * @param queryParams
 */
export function getChartPage(
 queryParams: ChartQuery
): AxiosPromise<PageResult<ChartPageVO[]>> {
 return request({
  url: '/api/v1/chart/page',
  method: 'get',
  params: queryParams
 });
}
```

然后把方法实现完全，这里就掠过具体步骤了

这里注意要初始化chartList数据，然后用onMounted初始化handleQuery，这样一点开就会出现啦~

```vue
<script setup lang="ts">
import Echarts from "@/components/Echarts/index.vue";
import { onMounted, ref } from "vue";
import { ChartPageVO, ChartQuery } from "@/api/bi/types";
import { getChartPage } from "@/api/bi";

const options = ref<any>(null);

const loading = ref(false);
const total = ref(1);
const queryParams = reactive<ChartQuery>({
 pageNum: 1,
 pageSize: 4,
});

const pageSizes = [4, 8, 12, 16, 20, 24];

const chartList = ref<ChartPageVO[]>(
 []
);

onMounted(() => {
 handleQuery();
});

/**
 * 查询
 */
function handleQuery() {
 loading.value = true;
 getChartPage(queryParams)
  .then(({ data }) => {
   chartList.value = data.list;
   console.log(chartList.value?.length);
   total.value = data.total;
  })
  .finally(() => {
   loading.value = false;
  });
}
</script>

<template>
 <div class="app-container">
  <div class="input-div">
   <el-input
    v-model="queryParams.chartName"
    placeholder="请输入图表名称"
    class="input-select"
   >
    <template #append>
     <el-button type="primary" @click="handleQuery">
      <i-ep-search />
      搜索
     </el-button>
    </template>
   </el-input>
  </div>
  <el-row :gutter="15">
   <el-col class="col"
       :span="12"
       v-for="(o, index) in chartList.length "
       :key="o"
       :offset="index > 0 ? 2 : 0">
    <el-card class="card">
     <div class="card-title">
      <div>
       <img class="card-logo" src="src/assets/paperfly-logo.png">
      </div>
      <div class="card-text">
       <span class="card-text1">纸飞机网站分析图</span>
       <span class="card-text2">图表类型：{{ chartList[index].chartType }}</span>
      </div>
     </div>
     <div>
      <span class="card-text3">分析目标：{{ chartList[index].goal }}</span>
      <Echarts class="myEcharts" :options="JSON.parse(chartList[index].genChart)" :width="'100%'" :height="'300px'" />
     </div>
    </el-card>
   </el-col>
  </el-row>

  <pagination
   class="page"
   v-if="total >= 0"
   v-model:total="total"
   v-model:page="queryParams.pageNum"
   v-model:limit="queryParams.pageSize"
   v-model:page-sizes="pageSizes"
   @pagination="handleQuery"
  />

 </div>
</template>

<style lang="scss" scoped>
.app-container {
 margin-left: 30px;
 margin-right: 30px;
}

.input-select {
 margin-bottom: 10px;

}

.col {
 margin-left: 0;
 margin-bottom: 15px;
}

.card {
 margin-left: 0;
}

.card-title {
 height: 50px;
 display: flex;
 flex-direction: row;
}

.card-logo {
 width: 40px;
 height: 40px;
}

.card-text {
 display: flex;
 flex-direction: column;
 justify-content: flex-start;
 margin-left: 10px;
}

.card-text1 {
 font-size: 10px;
 margin-top: 1px;
}

.card-text2 {
 font-size: 5px;
 align-self: flex-start;
 margin-top: 4px;
 color: rgba(128, 126, 126, 0.8)
}

.card-text3{
}

.myEcharts{
 margin-top: 20px;
}

.page {
 margin-top: -15px;
}
</style>
```

这里后期我会使用redis进行数据缓存功能，后面再去实现把，这里暂时不做

![image-20230716215905789](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230716215905789.png)

![image-20230716215942672](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230716215942672.png)

这里有个bug，在gpt返回数据的时候，title的值，并没有与图表名称一样，这里需要去鱼AI修改

## 2、系统优化



现在的网站足够安全么？

1. 如果用户上传一个超大的文件怎么办？
2. 如果用户用科技疯狂点击提交，怎么办？
3. 如果 AI 的生成太慢（比如需要一分钟），又有很多用户要同时生成，给系统造成了压力，怎么兼顾用户
4. 体验和系统的可用性？

### 2.1 安全性

如果用户上传一个超大的文件怎么办？比如1000G?

只要涉及到用户自主上传的操作，一定要校验文件（图像）

校验的维度：

1. 文件的大小
2. 文件的后缀
3. 文件的内容（成本要高一些）
4. 文件的合规性（比如敏感内容，建议用第三方的审核功能）

扩展点：接入腾讯云的图片万象数据审核(COS对象存储的审核功能)

```
// 校验文件
long fileSize = multipartFile.getSize();
String originalFilename = multipartFile.getOriginalFilename();
// 校验文件大小
ThrowUtils.throwIf(fileSize > FILE_MAX_SIZE, ErrorCode.PARAMS_ERROR, "文件大小超过 1M");
// 校验文件后缀
String suffix = FileUtil.getSuffix(originalFilename);
ThrowUtils.throwIf(!VALID_FILE_SUFFIX.contains(suffix), ErrorCode.PARAMS_ERROR, "不支持该类型文件");
/**
 * 文件大小 1M
 */
long FILE_MAX_SIZE = 1 * 1024 * 1024L;

/**
 * 文件后缀白名单
 */
List<String>  VALID_FILE_SUFFIX= Arrays.asList("xlsx","csv","xls","json");
```

> 事实上，仅仅校验文件后缀并不能完全保证文件的安全性，因为攻击者可以通过将非法内容更改后缀名来绕过校验。通过修改文件后缀的方式欺骗校验机制，将一些恶意文件
>
> 伪装成安全的文件类型。现在这个校验的维度是从浅到深，仅仅依靠校验后缀是远远不够的，还需要结合其他严格措施来加强文件的安全性。

### 2.2 数据存储

当前现状：我们把每个图表的原始数据全部存放在了同一个数据表 (chart表) 的字段里。

导致问题：

1. 如果用户上传的原始数据量很大、图表数日益增多，查询Chart 表就会很慢。
2. 对于 BI 平台，用户是有查看原始数据、对原始数据进行简单查询的需求的。现在如果把所有数据存放在一个字段（列）中，查询时，只能取出这个列的所有内容。



解决方案构思： 如果将原始数据以表格的形式存储在一个独立的数据表中，而不是放在一个小的格子里，实际上会更方便高效。由于数据表采用了标准的结构方式存储，我们可以通过使用SQL语句进行高效的数据检索，仅查询需要的列或行。此外，我们还可以利用数据库的索引等高效技术，更快、更精确地对数据进行定位和查询，从而提高查询效率和系统的响应速度。

#### 2.2.1 解决方案=>分库分表：

把每个图表对应的原始数据单独保存为一个新的数据表，而不是都存在一个字段里。

优点：

1. 存储时，能够分开存储，互不影响（也增加安全性）
2. 查询时，可以使用各种 SQL 语句灵活取出需要的字段，查询性能更快

> 优点构思：
>
> 使用分开存储的方式可以带来很多好处，其中一个好处就是存储的值相互独立，不会互相影响。例如，如果我们将一个100G的数据保存到同一个表中，其他用户在访问这个
>
> 数据表时会受到很大的影响，甚至在读取这个数据时可能会非常慢。
>
> 而通过将**每个表单独存储**，即使一个用户上传了很大的数据，其他用户在访问时也不会受到影响。这样可以保证数据的安全性和稳定性，同时也提高系统的处理能力和效率。
>
> 以后进行图表数据查询时，可以先根据图表的ID来查找，然后进行数据查询，方便我们排查问题。甚至返回用户原始数据，通过全标扫描的方式直接捞出所有数据，这比对数
>
> 据库查询数据进行处理更加快速和高效。

#### 2.2.2 分库分表（两种方式）

1. 水平分表（Sharding）

优点：

- 单个表的数据量减少，查询效率提高。
- 可以通过增加节点，提高系统的扩展性和容错性。

缺点：

- 事务并发处理复杂度增加，需要增加分布式事务的管理，性能和复杂度都有所牺牲。
- 跨节点查询困难，需要设计跨节点的查询模块。
- 如果存在热点数据，需要进行一定程度的负载均衡。

2. 垂直分库（Vertical Partitioning）

优点：

- 减少单个数据库的数据量，提高系统的查询效率。
- 增加了系统的可扩展性，比水平分表更容易实现。
- 可以根据不同的业务需求，将不同的数据库部署到不同的机器上，实现业务和数据的分离。

缺点：

- 不同数据库之间的维护和同步成本较高。
- 现有系统的改造存在一定的难度。
- 系统的性能会受到数据库之间互相影响的影响。

#### 2.2.3 分库分表的适用场景

一般来说，可以根据以下几个方面来判断是否使用水平分库或垂直分库：

1. 数据量大小和增长趋势：如果数据量很大或者数据增长速度很快，那么可以考虑使用水平分库来解决单个数据库性能下降或容量不足的问题，水平分库能够将数据分散存储到多个节点上，提高了查询性能和系统的扩展性；如果数据量较小或者增长速度较慢，那么可以采用垂直分库或者单库存储来减少维护成本。

2. 业务多样性：如果不同业务场景下数据结构、读写性能、安全性等方面差异较大，可以考虑使用垂直分库，将不同的业务数据分散到不同的数据库中，使得整个系统更加灵活，易于维护和升级。如果业务场景下数据结构和性能差异不大，可以采用单个数据库或水平分库。

3. 系统的可扩展性要求：如果需要快速扩容或者能够支持高并发处理，可以考虑使用水平分库，通过增加节点的方式实现水平扩展，提高系统的容错性和扩展性。如果要求系统的可维护性和稳定性更高，则可以采用垂直分库，将不同的业务数据分散到不同的数据库中，使得整个系统更加灵活、易于维护和升级。


需要根据实际的业务场景和技术需求来综合考虑各个因素，选择合适的分库分表策略。

**我这里没有去实现分库分表，因为是自己使用，并不会进行商用或者扩大使用，所以进行分库分表的意义不大，如果以后需要，再进行分库分表。**

后期想改造成，普通用户创建的图表只能存在1天和10个，添加定时任务功能，1天之后自动删除。

### 2.3 限流

> 我们需要控制用户使用系统的次数，以避免超支，比如给不同等级的用户分配不同的调用次数，防止用户过度使用系统造成破产（例如鱼聪明目前给普通用户提供50次调用次数，会员用户提供100次）。但限制用户调用次数仍存在一定风险，用户仍有可通过疯狂调用来刷量，从而导致系统成本过度消耗。
>
> 假设系统就一台服务器，能同时处理的用户对话数量是有限的，比如系统最多只能支持10个用户同时对话，如果某个用一秒内使用10个账号登录，那么其他用户就无法使用系统。就像去自助餐厅吃饭，如果有人一股脑地把所有美食都拿光了，其他人就无法享用了。比如双11这种大促期间，阿里巴巴就要去限制，不能说所有的用户想抢购都能成功，在前端随机放行一部分用户，而对于其他用户则进行限制，以确保系统不会被恶意用户占满。
>
> 现在要做一个解决方案，就是限流，比如说限制单个用户在每秒只能使用一次，这里我们怎么去思考这个限流的阈值是多少？多少合适呢？

现在的问题：使用系统是需要消耗成本的，用户有可能疯狂刷量，让你破产。

解决问题：

1、控制成本=>限制用户调用总次数

2、用户在短时间内疯狂使用，导致服务器资源被占满，其他用户无法使用=>限流

思考限流阈值多大合适？参考正常用户的使用，比如限制单个用户在每秒只能使用1次。



#### 2.3.1 限流算法

[面试必备：4种经典限流算法讲解](https://juejin.cn/post/6967742960540581918)

1、固定窗口限流

单位时间内允许部分操作

限定：1小时只允许10个用户操作

优点：最简单

缺点：可能出现流量突刺

比如：前59分钟没有1个操作，第59分钟来了10个操作；第1小时01分钟又来了10个操作。相当于2分钟内执行了20个操作，服务器仍然有高峰危险。



2、滑动窗口限流

单位时间内允许部分操作，但是这个单位时间是滑动的，需要指定一个滑动单位。

比如滑动单位：1min

开始前时间限流：0s 1h 2h

一分钟限流时间后：1min 1h1min 2h1min

> 解决了固定窗口限流存在的问题

优点：能够解决上述流量突刺的问题，因为第59分钟时，限流窗口是59分~1小时59分，这个时间段内只能接受10次请求，只要还在这个窗口内，更多的操作就会被拒绝。

缺点：实现相对固定窗口来说比较复杂，限流效果和你的滑动单位有关，滑动单位越小，限流效果越好，但往往很难选取到一个特别合适的滑动单位。



3、漏桶限流

特点：以固定的速率处理请求（漏水），当请求桶满了后，拒绝请求。

举例：每秒处理10个请求，桶的容量是10，每0.1秒固定处理一次请求，如果1秒内来了10个请求，这10此请求都可以处理完，但如果1秒内来了11个请求，最后那个请求就会溢出桶，被拒绝请求。

优点：能够一定程度上应对流量突刺，能够固定速率处理请求，保证服务器的安全。

缺点：没有固定速率处理一批请求，只能一个一个按顺序来处理（固定速率的缺点）

> 漏桶算法（Leaky Bucket Algorithm） 漏桶算法可以看作一个漏水的桶，水以恒定的速度流出，当水流入速度超出漏出速度时，多余的水将会溢出。在此算法中，请求被处理为水滴，漏桶则代表着请求被处理的容器。如果当前桶内水滴量小于等于当前请求数，则处理请求；若水滴数量大于当前请求数，则拒绝该请求。
>
> 实现过程：
>
> 1. 初始化漏桶容量和速率；
> 2. 不断将请求添加至漏桶；
> 3. 如果漏桶已满，则拒绝该请求；
> 4. 当漏桶中有请求时，在一定时间间隔内按照速率恒定地处理请求，并从漏桶中移除该请求。
>
> 漏桶算法在限流的过程中会固定处理请求的速率，因此无法应对突发请求或访问高峰期，容易造成请求处理延时或拒绝。同时，该算法无法准确地保证在短时间内达到整体平均限流速率。



4、令牌桶限流

> 可以解决漏桶存在的问题

管理员先生成一批令牌，每秒生成10个令牌；当用户要操作前，先去拿到一个令牌，有令牌的人就有资格执行操作、同时执行操作；拿不到令牌的就等着

优点：能够并发处理同时的请求，并发性能会更高

需要考虑的问题：还是存在时间单位选取的问题

> 令牌桶算法（Token Bucket Algorithm） 令牌桶算法可以看作一个令牌桶，其中令牌以恒定的速率产生。当一个请求到达时，如果令牌桶中仍然有令牌，则该请求得到处理并从令牌桶中减去一个令牌。如果令牌桶中没有令牌，则请求将被拒绝。在此算法中，令牌代表请求能够被处理的数量，而桶则代表着请求被处理的容器。
>
> 实现过程：
>
> 1. 初始化令牌桶容量和速率；
> 2. 以恒定速率往令牌桶中添加令牌；
> 3. 当请求到达时，如令牌桶中仍有令牌，则从桶中移除一个令牌，并处理该请求；
> 4. 如果没有足够的令牌，拒绝该请求。
>
> 令牌桶算法可以缓解漏桶算法的缺点，但在一些场景下可能存在一定问题。比如在应对短时间内的高并发请求时，由于令牌数有限，引入过大的并发请求会导致严重的性能问题，也可能会造成请求失败或者拒绝。

#### 2.3.2 限流粒度

1. 针对某个方法限流，即单位时间内最多允许同时X个操作使用这个方法
2. 针对某个用户限流，比如单个用户单位时间内最多执行X次操作
3. 针对某个用户X方法限流，比如单个用户单位时间内最多执行X次这个方法

#### 2.3.3 限流实现

1、本地限流（单机限流）

每个服务器单独限流，一般适用于单体项目，就是你的项目只有一个服务器。

在 Java 中，有很多第三方库可以用来实现单机限流：

Guava RateLimiter：这是谷歌 Guava 库提供的限流工具，可以对单位时间内的请求数量进行限制。

```java
import com.google.common.util.concurrent.RateLimiter;
  
public static void main(String\[] args) {

    // 每秒限流5个请求
    RateLimiter limiter = RateLimiter.create(5.0);
    while (true) {
        if (limiter.tryAcquire()) {
            // 处理请求
        } else {
            // 超过流量限制，需要做何处理
        }
	}
}
```

2、分布式限流 (集群限流)

如果你的项目有多个服务器，比如微服务，那么建议使用分布式限流。

1. 把用户的使用频率等数据放到一个集中的存储进行统计，比如Rdis,这样无论用户的请求落到了哪台服务器，都以集中的数据存储内的数据为准 (Redisson，是一个操作Redis的工具库)
2. 在网关集中进行限流和统计（比如Sentinel、Spring Cloud Gateway)

#### 2.3.4 Redisson限流实现

Redisson内置了一个限流江具类，可以帮助你利用Redis来存储、来统计。

[参考官网](https://github.com/redisson/redisson)

1. 看文档
2. 下载源码



实现过程

1. 安装Radis

2. 引入Redisson包

   ```
   <dependency>
      <groupId>org.redisson</groupId>
      <artifactId>redisson</artifactId>
      <version>3.21.3</version>
   </dependency>  
   ```

3. 创建RedissonConfig配置类，用于初始化RedissonClient对象单例：

   ```java
   @Data
   @ConfigurationProperties(prefix = "spring.redis")
   @Configuration
   public class RedissonConfig {
       private Integer database;
       private String host;
       private Integer port;
       private String password;
   
       @Bean
       public RedissonClient redissonClient() {
           Config config = new Config();
           config.useSingleServer()
                   .setDatabase(database)
                   .setAddress("redis://" + host + ":" + port)
                   .setPassword(password);
           RedissonClient redisson = Redisson.create();
           return redisson;
       }
   }
   ```

4. 编写RedisLimiterManager

   什么是Manager??专门提供RedisLimiter限流基础服务的（提供了通用的能力，可以放到任何一个项目里）

   ```java
   @Service
   public class RedisLimiterManager {
   
       @Resource
       private RedissonClient redissonClient;
   
       /**
        * 限流操作
        *
        * @param key 区分不同的限流器，比如不同的用户 id 应该分别统计
        */
       public void doRateLimit(String key) {
           // 创建一个限流器
           RRateLimiter rateLimiter = redissonClient.getRateLimiter(key);
           // 每秒最多访问 2 次
           // 参数1 type：限流类型，可以是自定义的任何类型，用于区分不同的限流策略。
           // 参数2 rate：限流速率，即单位时间内允许通过的请求数量。
           // 参数3 rateInterval：限流时间间隔，即限流速率的计算周期长度。
           // 参数4 unit：限流时间间隔单位，可以是秒、毫秒等。
           rateLimiter.trySetRate(RateType.OVERALL, 2, 1, RateIntervalUnit.SECONDS);
           // 每当一个操作来了后，请求一个令牌
           boolean canOp = rateLimiter.tryAcquire(1);
           ThrowUtils.throwIf(!canOp,ErrorCode.TOO_MANY_REQUEST);
       }
   }
   ```

5. 单元测试

   ```java
   @SpringBootTest
   class RedisLimiterManagerTest {
       @Resource
       private RedisLimiterManager redisLimiterManager;
   
       @Test
       void doRateLimit() throws InterruptedException {
           String userId = "1";
           for (int i = 0; i < 2; i++) {
               redisLimiterManager.doRateLimit(userId);
               System.out.println("成功");
           }
           Thread.sleep(1000);
           for (int i = 0; i < 5; i++) {
               redisLimiterManager.doRateLimit(userId);
               System.out.println("成功");
           }
       }
   }
   ```

6. 应用到要限流的方法中，比如智能分析接口

   ```
   // 用户每秒限流
   redisLimiterManager.doRateLimit("genChartByAi_" + loginUser.getId());
   ```

![image-20230718140059529](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230718140059529.png)

# 智能分析业务开发（五）

## 1、当前系统问题分析

1. 第一个问题:
   图表生成的时间有点长，因为背后用的 AI 能力是需要一定时间来完成处理的。

2. 第二个问题:
   当系统面临大量用户请求时，如果处理能力有限，例如服务器的内存、CPU、网络带宽等资源有限，这可能导致用户处在一个长时间的等待状态。特别是在许多用户同时提交请求的情况下，服务器可能需要较长的时间来处理此外，如果我们后端的 AI 处理能力有限，也有可能引发问题。比如，为了确保平台的安全性，我们可能会限制用户的访问频率，即每秒或每几秒用户只能访问一次或几次。一旦用户过多地提交请求，就会增大 AI 处理的服务器的压力，导致 AI服务器处理不了这么多请求。在这种情况下，其他用户只能等待，而在前端界面也只能显示持续等待的状态。长时间等待后，用户可能会收到服务器繁忙的错误信息。这不仅影响了用户的体验，也对服务器和我们使用的第三方服务带来压力.
   我们还需要考虑服务器如 Tomcat 的线程数限制，在极端情况下，比如每十秒只能处理一个请求，但却有 200 个用户在一秒钟内同时提交请求，这就会导致大量用户请求在服务器上积压，数据也无法及时插入到数据库中。如果用户长时间等待最终仍得到请求失败的结果，这种情况下也会对服务器造成压力。
3. 第三个问题:
   当我们调用第三方服务，比如我们的 AI 处理能力是有限的，如每三秒只能处理一个请求。在这种情况下，大量用户同时请求可能导致 AI 过载，甚至拒绝我们的请求。假设我们正在使用的鱼聪明平台，这是一个提供 AI 回答功能的服务。在我们的开发的智能 Bl 中，如果有 100用户同时访问，就需要 100 次调用鱼聪明 Al。然而，鱼聪明 AI 可能无法在一秒钟内服务 100 个用户。这种情况下，AI 服务会认为我们在攻击它，或者超过了它的处理能力，可能会对我们施加限制。这构成了一个潜在的风险

整理一下：

1. 用户等待时间有点长（因为要等AI生成）
2. 业务服务器可能会有很多请求在处理，导致系统资源紧张，严重时导致服务器宕机或者无法处理新的请求
3. 调用的第三方服务(AI能力)的处理能力是有限的，比如每3秒只处理1个请求，当多个请求时会导致 AI 处理不过来，严重时AI可能会对咱们的后台系统拒绝服务。



**AI提供的解决方案：**

针对当前系统问题的分析，可以考虑以下解决方案：

1. 异步化处理：对于生成图表的过程，可以考虑将其异步化处理，即将生成图表的任务交给后台线程去处理，而不是让用户等待。可以使用消息队列或异步任务框架来实现异步处理，将生成图表的请求放入队列中，后台线程逐个处理请求并生成图表，生成完成后通知用户。

2. 资源优化：针对系统资源紧张的问题，可以考虑对系统进行优化，包括但不限于以下方面：
   - 资源调优：对于业务服务器，可以进行资源调优，包括增加服务器的硬件配置、优化系统参数等，以提高系统的处理能力和稳定性。
   - 负载均衡：可以考虑使用负载均衡技术，将请求分发到多台服务器上进行处理，以分担单台服务器的压力。
   - 缓存技术：对于频繁请求的数据，可以使用缓存技术进行缓存，减少对后台系统的访问，提高系统的响应速度。

3. 限流控制：针对调用的第三方服务的处理能力有限的问题，可以考虑进行限流控制，即限制系统对第三方服务的请求频率，以避免过多的请求导致第三方服务无法处理。可以使用限流算法，如令牌桶算法或漏桶算法，对请求进行限制，确保系统和第三方服务的稳定性。

4. 异常处理：针对第三方服务对系统的拒绝服务问题，可以进行异常处理，包括但不限于以下方面：
   - 重试机制：对于请求被拒绝的情况，可以进行重试，以期待第三方服务能够处理请求。
   - 降级处理：当第三方服务无法处理请求时，可以进行降级处理，如返回默认数据或提示用户稍后再试。

综上所述，针对当前系统问题，可以通过异步化处理、资源优化、限流控制和异常处理等方面的措施来解决问题，提高系统的性能和稳定性。

## 2、异步化处理图表生成讲解（重要）

### 2.1 同步与异步

同步：一件事情做完，再做另外一件事情，不能同时进行其他的任务。（比如你在烧水，就不能去做其他的时间。但是事实并非如此。） 异步：不用等一件事故完，就可以做另外一件事情。等第一件事完成时，可以收到一个通知，通知你这件事做好了，你可以再进行后续处理。（烧水的时候，水壶有一个警报器；烧水时人可以同时处理工作。水烧好后，人听到声音，就知道水烧好了)



### 2.2 业务流程

#### 2.2.1 标准异步化的业务流程

1. 当用户要进行耗时很长的操作时，点击提交后，不需要在界面长时间的等待，而是应该把这个任务保存到数据库中记录下来
2. 用户要执行新任务时：
   1. 任务提交成功：
      1. 如果我们的程序还有多余的空闲线程，可以立刻去执行这个任务。
      2. 如果我们的程序的线程都在繁忙，无法继续处理，那就放到等待队列里。
   2. 任务提交失败：比如我们的程序所有线程都在忙，**任务队列满了**。
      1. 拒绝掉这个任务，再也不去执行。
      2. 通过保存到数据库中的记录来看到提交失败的任务，并且在程序空闲的时候，可以把任务从数据库中回调到程序里，再次去执行此任务。
3. 我们的程序（线程）从任务队列中取出任务依次执行，每完成一件事情要修改一下的任务的状态。
4. 用户可以查询任务的执行状态，或者在任务执行成功或失败时能得到通知（发邮件、系统消息提示、短信），从而优化体验。
5. 如果我们要执行的任务非常复杂，包含很多环节，在每一个小任务完成时，要在程序（数据库中）记录一下任务的执行状态（进度）。

#### 2.2.2 本BI系统的业务流程

1. 用户点击智能分析页面的提交按钮时，先把图表立刻保存到数据库中（作为一个任务）
2. 用户可以在图表管理界面插查看所有的图表的信息和状态  
   - 已生成的
   - 生成中的
   - 生成失败的
3. 用户可以修改生成失败的图表信息，点击重新生成图表  

> 此时就会用到异步化消息

但是现在  也会存在一定的问题：

1. 任务队列的最大容量应该设置多少合适？
2. 程序怎么从任务队列中取出任务去执行，这个任务队列的流程怎么实现的，怎么保证程序最多同时执行多少个任务？
   - 阻塞队列
   - 线程池
   - 增加更多的人手?

## 3. 线程池

什么是线程池？

> 线程池是一种线程管理的机制，它可以在程序启动时创建一定数量的线程，并将这些线程放入池中，供程序使用。线程池可以有效地管理线程的创建、销毁和复用，提高线程的利用率和系统的性能。
>
> 线程池的主要作用是控制线程的数量，避免系统中创建过多的线程，导致系统资源的浪费和性能下降。线程池可以根据系统的负载情况动态地调整线程的数量，以适应不同的工作负载。
>
> 线程池一般包含以下几个组件：
>
> 1. 任务队列（Task Queue）：用于存放待执行的任务，线程池中的线程可以从任务队列中获取任务并执行。
> 2. 线程池管理器（ThreadPool Manager）：负责创建、销毁和管理线程池中的线程。
> 3. 线程工厂（Thread Factory）：用于创建线程，可以自定义线程的创建方式，如线程的名称、优先级等。
> 4. 线程池执行器（ThreadPool Executor）：负责执行任务，从任务队列中获取任务，并将任务分配给线程池中的线程执行。
>
> 线程池的优点包括：
>
> 1. 降低线程创建和销毁的开销：线程池中的线程可以被复用，避免了频繁创建和销毁线程的开销。
> 2. 提高系统的响应速度：线程池可以并发执行多个任务，提高系统的并发处理能力和响应速度。
> 3. 控制线程的数量：线程池可以根据系统的负载情况动态地调整线程的数量，避免系统中创建过多的线程。
> 4. 提供线程的管理和监控：线程池可以提供线程的管理和监控功能，如线程的状态、执行情况等。
>
> 总之，线程池是一种高效管理线程的机制，可以提高系统的性能和稳定性，是多线程编程中常用的技术之一。



本项目为什么使用线程池？

答：

1. 线程的管理比较复杂（比如：什么时候新增线程、什么时候减少空闲线程）
2. 任务存取比较复杂（什么时候接受任务、什么时候拒绝任务、怎么保证大家不抢到同一个线程）



### 3.1 JUC线程池的实现方式

1. 不用自己写，如果是在Spring中，可以用`ThreadPoolTaskExecutor`配合`@Async`注解来实现。(不太建议：进行了封装)
2. 如果是在Java中，可以使用JUC并发编程包，中的`ThreadPoolExecutor`来实现非常灵活地自定义线程池。

JUC中的线程池：

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) 
```

> 怎么确定线程池参数呢？结合实际情况（实际业务场景和系统资源）来测试调整，不断优化。
>
> 
>
> 回归到本BI系统的业务，要考虑系统最脆弱的环节（系统的瓶颈）在哪里？
>
> 现有条件：比如AI生成能力的并发是只允许4个任务同时去执行，AI能力允许20个任务排队。

那么设计成如下业务流程：

![image-20230718150908778](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230718150908778.png)

### 3.2 经典面试题：说说线程池的参数？

此构造方法的参数如下：

1. `int corePoolSize`：核心线程数（正式员工人数）==正常情况下，我们的系统应该能同时工作的线程数（随时就绪的状态)==，线程池中一直存在的线程数，即使线程闲置。

2. `int maximumPoolSize`：（最大线程数 =>  哪怕任务再多，你也只能最多招5个人），==极限情况下，我们的线程池最多有多少个线程？==，线程池中允许存在的最大线程数。

3. `long keepAliveTime`：（空闲线程存活时间），==非核心线程在没有任务的情况下，过多久要删除（理解为开除临时工），从而释放无用的线程资源==。非核心线程的空闲线程存活时间，单位：毫秒。

4. `TimeUnit unit`：（空闲线程存活时间的单位）存活时间的单位，可选的单位包括：`TimeUnit.MILLISECONDS、TimeUnit.SECONDS、TimeUnit.MINUTES、TimeUnit.HOURS、TimeUnit.DAYS`（时分秒）等等。

5. `BlockingQueue<Runnable> workQueue`：（工作队列）==用于存放给线程执行的任务，存在一个队列的长度（一定要设置，不要说队列长度无限，因为也会占用资源)==。

   > 用于存放线程任务的阻塞队列，当线程池中的线程数达到了`corePoolSize`，而阻塞队列中任务满了时，线程池会创建新的线程，直到达到`maximumPoolSize`，此时达到线程池最大容量，若阻塞队列不为空，新加入的任务将会被拒绝，同时也可以通过设置`RejectedExecutionHandler`处理满了阻塞队列和饱和的情况。

6. `ThreadFactory threadFactory`：（线程工厂）线程创建工厂，==控制每个线程的生成、线程的属性（比如线程名）==，用于创建新的线程，可以自定义ThreadFactory。

7. `RejectedExecutionHandler handler`：（拒绝策略）线程池拒绝策略，==任务队列满的时候，我们采取什么措施，比如抛异常、不抛异常、自定义策略==。

   > 当任务无法处理时，会根据设置的拒绝策略进行处理，可选的策略有：AbortPolicy（直接抛出异常，终止程序的执行）、CallerRunsPolicy（让当前的线程来处理该任务）、DiscardPolicy（直接丢弃该任务），DiscardOldestPolicy（丢弃队列中已经存在最久的任务，将当前任务插入队列尝试提交）。

延伸讲解：

> 资源隔离策略：不同的程度的任务，分为不同的队列，比如VIP一个队列，普通用户一个队列。

**线程池的主要处理流程**

```
1.在创建了线程池之后，等待提交过来的任务请求
2.当调用execute()方法添加一个请求任务的时候，线程池会做出如下判断：
2.1 如果正在运行的线程数量小于corePoolSize，那么马上创建线程运行这个程序
2.2 如果正在运行的线程数量大于或者等于corePoolSize，那么将这个任务放入队列
2.3 如果这个时候队列满了并且正在运行的线程数量还小于maximumPoolSize，那么还是要创建非核心线程立刻执行这个任务
2.4 如果队列满了并且正在运行的线程数量大于或等于maximumPoolSize，那么线程池会启动饱和拒绝策略来执行
3. 当一个线程完成任务的时候，它会从队列中取下一个任务来执行
4. 当一个线程无事可做超过一定时间（keepAliveTime）时：线程会判断如果当前运行的线程数大于corePoolSize，那么这个线程就会被停掉。
```

### 3.3 补充知识：任务的分类

任务分为IO密集型和计算密集型是指任务所涉及的操作类型不同。

1. IO密集型：任务需要大量的输入输出操作（I/O），如读取文件、访问数据库、网络通信等，这些操作的执行时间比较长，CPU 等待 IO 的时间比较多，CPU 利用率比较低。
2. 计算密集型：任务需要大量的计算操作，如图像处理、视频编码、加密解密等，这些操作的执行时间比较短，CPU利用率比较高。

在实际应用中，对于 IO 密集型任务，我们可以采用并发或异步方式来优化执行效率，并且可以采用缓存技术来避免频繁的 IO 操作；对于计算密集型任务，我们可以采用多线程、分布式或 GPU 加速等方式来提高执行效率。

### 3.4 线程池实现开发

自定义线程池

```java
/**
 * @author Shier
 * CreateTime 2023/6/7 20:31
 */
@Configuration
public class ThreadPoolExecutorConfig {

    @Bean
    public ThreadPoolExecutor threadPoolExecutor() {
        ThreadFactory threadFactory = new ThreadFactory() {
            private int count = 1;

            @Override
            public Thread newThread(@NotNull Runnable r) {
                // 一定要将这个 r 放入到线程当中
                Thread thread = new Thread(r);
                thread.setName("线程：" + count);
                // 任务++
                count++;
                return thread;
            }
        };
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(2, 4, 100, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(100), threadFactory);
        return threadPoolExecutor;
    }
}
```

队列测试

```java
/**
 * 队列测试
 *
 * @author Shier
 */
@RestController
@RequestMapping("/queue")
@Slf4j
@Profile({ "dev", "local" })
@Api(tags = "QueueController")
@CrossOrigin(origins = "http://localhost:8000", allowCredentials = "true")
public class QueueController {

    @Resource
    private ThreadPoolExecutor threadPoolExecutor;

    @GetMapping("/add")
    public void add(String name) {
        CompletableFuture.runAsync(() -> {
            log.info("任务执行中：" + name + "，执行人：" + Thread.currentThread().getName());
            try {
                Thread.sleep(60000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },threadPoolExecutor);
    }

    @GetMapping("/get")
    public String get() {
        Map<String, Object> map = new HashMap<>();
        int size = threadPoolExecutor.getQueue().size();
        map.put("队列长度:", size);
        long taskCount = threadPoolExecutor.getTaskCount();
        map.put("任务总数:", taskCount);
        long completedTaskCount = threadPoolExecutor.getCompletedTaskCount();
        map.put("已完成任务数:", completedTaskCount);
        int activeCount = threadPoolExecutor.getActiveCount();
        map.put("正在工作的线程数:", activeCount);
        return JSONUtil.toJsonStr(map);
    }
}
```

日志可以看得出：

我们直接干到任务9，就会报错

![image-20230718211407154](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230718211407154.png)

## 4、本项目开发异步化前后端

### 4.1 后端实现工作流程

1. 给chart表新增任务状态字段（比如排队中、执行中、已完成、失败），任务执行信息字段（用于记录任务执行中、或者失败的一些信息)

2. 用户点击智能分析页的提交按钮时，先把图表立刻保存到数据库中，然后提交任务

3. 任务：先修改图表任务状态为"执行中"。等执行成功后，修改为"已完成"、保存执行结果；执行失败后，状态修改为"失败"，记录任务失败信息。

4. 用户可以在图表管理页面查看所有图表（已生成的、生成中的、生成失败）的信息和状态




1. 给chart表新增任务状态字段（比如排队中、执行中、已完成、失败），任务执行信息字段（用于记录任务执行中、或者失败的一些信息)

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
       chartStatus varchar(128) default 'wait'       not null comment 'wait-等待,running-生成中,succeed-成功生成,failed-生成失败',
       execMessage text                              null comment '执行信息',
       userId     bigint                             null comment '创建图标用户 id',
       create_time datetime default CURRENT_TIMESTAMP not null comment '创建时间',
       update_time datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
       isDelete   tinyint  default 0                 not null comment '是否删除'
   ) comment '图表信息表' collate = utf8mb4_unicode_ci;
   ```

   补充实体类字段：

   ```java
   /**
    * 生成的图表信息
    */
   private String genChart;
   
   /**
    * 图表状态 wait-等待,running-生成中,succeed-成功生成,failed-生成失败
    */
   private String chartStatus;
   ```

2. 创建枚举类封装status

```java
public enum ChartStatus {
    WAIT("wait"),
    RUNNING("running"),
    SUCCEED("succeed"),
    FAILED("failed");

    private String value;

    ChartStatus(String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }
}
```

3. controller类创建异步接口

```java
/**
 * 智能分析（异步）
 *
 * @param multipartFile
 * @param genChartByAiRequestJson
 * @return
 */
@Operation(summary = "AI智能分析", security = {@SecurityRequirement(name = "Authorization")})
@PostMapping("/gen/async")
public Result<BiResponseVo> genChartByAiAsync(@RequestPart("file") MultipartFile multipartFile,
                                         @RequestParam("genChartByAiRequest") String genChartByAiRequestJson) {
    GenChartByAiRequest genChartByAiRequest = JSONUtil.parseObj(genChartByAiRequestJson).toBean(GenChartByAiRequest.class);

    ValidationUtil.validateGenChartByAiRequest(genChartByAiRequest);
    /**
     * 三个参数：文件，文件最大大小，后缀限制
     */
    ValidationUtil.validateFile(multipartFile, File_SIZE_ONE, VALID_FILE_SUFFIX);

    BiResponseVo biResponse = chartService.getBiResponseAsync(genChartByAiRequest,multipartFile);

    return Result.success(biResponse);
}
```

4. 将解析AI数据封装为一个接口

```java
/**
 * 处理AI数据
 * @param chartResult
 * @return
 */
OrganizeAiDataVo getOrganizeAiData(String chartResult);
```

实现类

```java
/**
 * 处理AI数据
 * @param chartResult
 * @return
 */
@Override
public OrganizeAiDataVo getOrganizeAiData(String chartResult) {

    // 解析内容
    String[] splits = chartResult.split(GEN_CONTENT_SPLITS);
    if (splits.length < GEN_ITEM_NUM) {
        throw new BusinessException(ErrorCode.SYSTEM_ERROR, "AI生成错误");
    }
    // 首次生成的内容
    String genChart = splits[GEN_CHART_IDX].trim();
    String genResult = splits[GEN_RESULT_IDX].trim();

    OrganizeAiDataVo result = new OrganizeAiDataVo();
    result.setGenResult(genResult);
    result.setGenChart(genChart);

    return result;
}
```

5. 实现异步智能分析接口

```java
/**
 * 经过AI和自己处理后，返回给前端的数据（异步）
 * @param genChartByAiRequest
 * @param multipartFile
 * @return BiResponseVo
 */
@Override
public BiResponseVo getBiResponseAsync(GenChartByAiRequest genChartByAiRequest, MultipartFile multipartFile) {
    String chartName = genChartByAiRequest.getChartName();
    String goal = genChartByAiRequest.getGoal();
    String chartType = genChartByAiRequest.getChartType();

    UserInfoVO loginUser = userService.getUserLoginInfo();
    // 限流判断，每个用户一个限流器
    redisLimiterManager.doRateLimit("genChartByAi_" + loginUser.getUserId());

    // 插入到数据库
    String csvData = ExcelUtils.excelToCsv(multipartFile);
    String chartStatus = ChartStatus.WAIT.getValue();
    BiChart chart = BiChartUtils.getBiChartAsync(chartName, goal, chartType, csvData, chartStatus, loginUser);
    boolean saveResult = chartService.save(chart);
    ThrowUtils.throwIf(!saveResult, ErrorCode.SYSTEM_ERROR, "图表保存失败");

    // 调用对应AI模型的id
    long biModelId = CommonConstant.BI_MODEL_ID;
    // 构造用户输入
    String userInput = BiChartUtils.getUserInput(goal, chartType, multipartFile);

    // todo 建议处理任务队列满了后，抛异常的情况
    CompletableFuture.runAsync(() -> {
        // 先修改图表任务状态为 “执行中”。等执行成功后，修改为 “已完成”、保存执行结果；执行失败后，状态修改为 “失败”，记录任务失败信息。
        BiChart updateChart = new BiChart();
        updateChart.setId(chart.getId());
        updateChart.setChartStatus(ChartStatus.RUNNING.getValue());
        boolean b = chartService.updateById(updateChart);
        if (!b) {
            handleChartUpdateError(chart.getId(), "更新图表执行中状态失败");
            return;
        }
        // 调用Ai
        String chartResult = aiManager.doChat(biModelId, userInput);
        // 处理ai生成的数据
        OrganizeAiDataVo organizeAiData = getOrganizeAiData(chartResult);
        String genChart = organizeAiData.getGenChart();
        String genResult = organizeAiData.getGenResult();

        BiChart updateChartResult = new BiChart();
        updateChartResult.setId(chart.getId());
        updateChartResult.setGenChart(genChart);
        updateChartResult.setGenResult(genResult);
        updateChartResult.setChartStatus(ChartStatus.SUCCEED.getValue());
        boolean updateResult = chartService.updateById(updateChartResult);
        if (!updateResult) {
            handleChartUpdateError(chart.getId(), "更新图表成功状态失败");
        }
    }, threadPoolExecutor);

    BiResponseVo biResponse = new BiResponseVo();
    biResponse.setChartId(chart.getId());

    return biResponse;
}
```

**优化，设置队列满了之后，抛异常的情况：**

在使用线程池进行异步处理时，如果任务队列已满，线程池中的线程会被阻塞，直到有空闲线程可用为止。当线程池中的线程数达到最大线程数限制时，新提交的任务将会被放入到任务队列中等待执行。

如果此时任务队列已满，则默认的行为是直接抛出RejectedExecutionException异常，这是因为线程池已经无法承载更多的任务了。如果出现这种情况，通常需要调整线程池的配置或者增加机器资源来解决。

为了避免这种情况，你可以定义一个自定义的拒绝策略来处理抛出的异常。你可以根据具体的业务需求来选择如何处理该异常。例如，你可以将抛出异常的任务添加到一个专门的队列中，以便稍后进行处理；或者在任务被拒绝时记录日志等。

最常见的拒绝策略有以下几种：

1. AbortPolicy：直接抛出RejectedExecutionException异常，默认策略。
2. CallerRunsPolicy：将任务分配给调用它的线程去执行。
3. DiscardPolicy：直接丢弃任务。
4. DiscardOldestPolicy：丢弃队列中等待时间最久的任务，然后再将当前任务放入队列中等待执行。

你可以在创建线程池时，通过ThreadPoolExecutor的构造函数或者setRejectedExecutionHandler方法来指定拒绝策略。

例如，你可以定义一个DiscardPolicy策略来处理抛出的异常，如下所示：

```java
class MyDiscardPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        // 直接丢弃任务
        System.out.println("Task " + r.toString() + " rejected from " + e.toString());
    }
}
```

在配置线程池时，将这个策略传入即可：

```java
@Bean
public ThreadPoolExecutor threadPoolExecutor() {
    ThreadFactory threadFactory = new ThreadFactory() {
        private int count = 1;

        @Override
        public Thread newThread(@NotNull Runnable r) {
            Thread thread = new Thread(r);
            thread.setName("线程" + count);
            count++;
            return thread;
        }
    };
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            2,
            4,
            100,
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(4),
            threadFactory,
            new MyDiscardPolicy()
    );
    return threadPoolExecutor;
}
```

这样就可以在任务队列已满的情况下丢弃任务了。

### 4.2 前端开发

#### 4.2.1 优化我的图表-添加删除按钮

1. 将页面展示的数据分为展示两个，然后将分析结论也展示处理

2. 添加两个按钮，修改按钮和删除按钮，修改按钮没有去实现，摆烂了

添加删除按钮，并且优化了一下显示的card

```vue
<el-row :gutter="15">
 <el-col class="col"
     :span="12"
     v-for="(o, index) in chartList.length "
     :key="o"
     :offset="index > 0 ? 2 : 0">
  <el-card class="card">
   <div class="card-title">
    <div>
     <img class="card-logo" src="../../../assets/paperfly-logo.png">
    </div>
    <div class="card-header">
     <div class="card-text">
      <span class="card-text1">纸飞机网站分析图</span>
      <span class="card-text2">图表类型：{{ chartList[index].chartType }}</span>
     </div>
     <div class="card-date">
      创建时间: {{ chartList[index].createTime }}
     </div>
    </div>
   </div>
   <div class="card-contant">
    <div class="card-chartName">分析名称：{{ chartList[index].chartName }}</div>
    <div class="card-goal">分析目标：{{ chartList[index].goal }}</div>
    <Echarts class="myEcharts" :options="JSON.parse(chartList[index].genChart)" :width="'100%'" :height="'300px'" />
   </div>
   <div class="card-genResult">
    <div>{{ chartList[index].genResult }}</div>
   </div>
   <div class="card-button">
    <el-button type="primary">修改</el-button>
    <el-button type="danger" @click="handleDeleteChart(chartList[index].id)">删除</el-button>
   </div>
  </el-card>
 </el-col>
</el-row>
```

发起删除的函数：

```ts
/**
 * 删除
 */
function handleDeleteChart(chartId: number) {
 // 发起请求，删除图表
 deleteChart(chartId)
  .then(() => {
   // 删除成功后，重新加载图表数据
   handleQuery();
  })
  .catch((error) => {
   console.error("删除图表失败:", error);
  });
}
```

调用index.ts的函数

```tsx
/**
 * 删除图表
 *
 * @param filePath 文件完整路径
 */
export function deleteChart(chartId?: number) {
 return request({
  url: '/api/v1/chart/delete',
  method: 'delete',
  params: { chartId },
 });
}
```

最后展现的效果：

![image-20230719164644471](C:\Users\Kaizhi\AppData\Roaming\Typora\typora-user-images\image-20230719164644471.png)

#### 4.2.2 优化我的图表-添加状态显示

修改分页时展示的内容的数据，修改type.ts

```ts
export interface ChartPageVO {

 /**
  * 图表Id
  */
 id?: number;

 /**
  * 图表名称
  */
 chartName?: string;

 /**
  * 图表类型
  */
 chartType?: string;

 /**
  * 分析目标
  */
 goal?: string;

 /**
  * 生成的echart所需的数据
  */
 genChart?: string;

 /**
  *
  */
 chartStatus?: string;

 /**
  *
  */
 execMessage?: string;

 /**
  * 图表结论
  */
 genResult?: string;

 /**
  * 创建时间
  */
 createTime?: Date;
}
```

然后修改展示内容需要根据chartStatus显示

```vue
<template>
 <div class="app-container">
  <div class="input-div">
   <el-input
    v-model="queryParams.chartName"
    placeholder="请输入图表名称"
    class="input-select"
   >
    <template #append>
     <el-button type="primary" @click="handleQuery">
      <i-ep-search />
      搜索
     </el-button>
    </template>
   </el-input>
  </div>
  <el-row :gutter="15">
  <el-col class="col"
      :span="12"
      v-for="(o, index) in chartList.length "
      :key="o"
      :offset="index > 0 ? 2 : 0">
   <el-card class="card">
    <div class="card-title">
     <div>
      <img class="card-logo" src="../../../assets/paperfly-logo.png">
     </div>
     <div class="card-header">
      <div class="card-text">
       <span class="card-text1">纸飞机网站分析图</span>
       <span class="card-text2">图表类型：{{ chartList[index].chartType }}</span>
      </div>
      <div class="card-date">
       创建时间: {{ chartList[index].createTime }}
      </div>
     </div>
    </div>
    <div class="card-contant">
     <div class="card-chartName">分析名称：{{ chartList[index].chartName }}</div>
     <div class="card-goal">分析目标：{{ chartList[index].goal }}</div>
    </div>
    <template v-if="chartList[index].chartStatus === 'wait'">
     <div class="card-genChart-loading" v-loading="loading">
      <i class="el-icon-loading"></i>
      正在等待生成...
     </div>
    </template>
    <template v-if="chartList[index].chartStatus === 'running'">
     <div class="card-genChart-running" v-loading="loading">
      <i class="el-icon-running"></i>
      正在生成当中，请稍等哟~~
     </div>
    </template>
    <template v-else-if="chartList[index].chartStatus === 'succeed'">
     <div class="card-genChart">
      <Echarts class="myEcharts" :options="JSON.parse(chartList[index].genChart)" :width="'100%'"
           :height="'300px'" />
     </div>
     <div class="card-genResult">
      <div>{{ chartList[index].genResult }}</div>
     </div>
    </template>
    <template v-else-if="chartList[index].chartStatus === 'failed'">
     <div class="card-genChart-failed">
      <el-result
       icon="error"
       title="分析失败"
       :sub-title=chartList[index].execMessage
      >
       <template #extra>
       </template>
      </el-result>
     </div>
    <div class="card-button">
     <el-button type="primary">修改</el-button>
     <el-button type="danger" @click="handleDeleteChart(chartList[index].id)">删除</el-button>
    </div>
   </el-card>
  </el-col>
 </el-row>

  <pagination
   class="page"
   v-if="total >= 0"
   v-model:total="total"
   v-model:page="queryParams.pageNum"
   v-model:limit="queryParams.pageSize"
   v-model:page-sizes="pageSizes"
   @pagination="handleQuery"
  />

 </div>
</template>
```

主要是使用template组件，如果状态是什么样子再变成什么样子

![](https://paperfly-blog.oss-cn-shenzhen.aliyuncs.com/blog/image-20230720140018810.png)

#### 4.2.3 添加我的页面自动更新数据

要实现每隔十秒调用一次`handleQuery`方法，并判断响应的数据是否改变并更新页面，你可以按照以下步骤进行操作：

1. 在`<script>`标签中引入Vue的`watch`方法：`

   ```tsx
   import { watch } from "vue";
   ```

   

2. 在`<script>`标签中定义一个响应式变量`responseData`，用于保存上一次请求返回的数据：

   ```ts
   const responseData = ref(null);`
   ```

   

3. 使用`watch`方法监听`chartList`和`loading`两个值的变化，在回调函数中进行判断和更新页面的操作：

```tsx
watch([chartList, loading], ([newChartList, newLoading]) => {
  // 只处理加载完成的情况
  if (!newLoading) {
    // 判断响应的数据是否有变化
    if (JSON.stringify(newChartList) !== JSON.stringify(responseData.value)) {
      // 更新页面
      responseData.value = newChartList;
    }
  }
});
```

4. 在`handleQuery`方法的最后，将获取到的图表数据赋值给`chartList`之前，先保存到`responseData`变量中：

```tsx
function handleQuery() {
  loading.value = true;
  getChartPage(queryParams)
    .then(({ data }) => {
      responseData.value = data.list; // 保存上一次请求返回的数据
      chartList.value = data.list;
      console.log(chartList.value);
      total.value = data.total;
    })
    .finally(() => {
      loading.value = false;
    });
}
```

5. 使用`setInterval`函数每隔十秒调用一次`handleQuery`方法：

```tsx
onMounted(() => {
  handleQuery();
  setInterval(handleQuery, 10000);
});
```

通过以上步骤，你可以实现每隔十秒调用一次`handleQuery`方法，并判断响应的数据是否改变并更新页面。

其实还有另一种方法，可以使用websocket，当后端的数据发送改变的时候，发信息给前端，让前端进行刷新。我这里偷懒了，没学过webstorm，等学会了可以把这里逻辑更改一下~



可优化点：

1. 使用websocket对前端我的图表进行异步更新
2. 使用redis存储分页信息，避免重复调用，这里如果用这种功能，需要实现：数据库一致性问题。
