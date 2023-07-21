# Albert.SqlOperateSplitDemo

#### 介绍
单库水平分表在 C# 中 SqlSugar ORM 有成熟的框架技术（https://www.donet5.com/Home/Doc?typeId=1201）。且使用起来极为简单，没有 Java 那么繁琐麻烦，下面我们来看看。

#### 注意事项
1. 创建 WebApi 项目，引入 Nuget
● SqlSugarCore
● SqlSugar.IOC
2. 创建 Models 实体类
需要注意的地方是
● 分表字段必须加索引、查询主键必须加索引，不然会导致大数据查询性能低。  
● 数据主键不能是自增，用 long 来，插入的时候会有雪花 ID 算法来保证。  

```
using SqlSugar;

namespace Albert.SqlOperateSplitDemo.Models
{

    [SplitTable(SplitType.Month)]//按年分表 （自带分表支持 年、季、月、周、日）
    [SugarTable("BillsMonth_{year}{month}{day}")]//3个变量必须要有，这么设计为了兼容开始按年，后面改成按月、按日
    // 创建唯一索引--SerialNumber
    [SugarIndex("unique_BillsMonth_SerialNumber", nameof(BillsMonth.SerialNumber), OrderByType.Desc, true)]
    [SugarIndex("index_BillsMonth_CreateTime", nameof(BillsMonth.CreateTime), OrderByType.Asc)]
    public class BillsMonth
    {
        // 内部自带雪花 ID 实现
        [SugarColumn(IsPrimaryKey = true)]
        public long Id { get; set; }
        public string SerialNumber { get; set; }
        public decimal PayMoney { get; set; }
        public string BillsInfo { get; set; }
        public DateTime BillDate { get; set; }
        [SugarColumn(IsNullable = true)]//设置为可空字段
        public string BillComment { get; set; }
        public string MachineSerialNo { get; set; }
        [SplitField] //分表字段 在插入的时候会根据这个字段插入哪个表，在更新删除的时候用这个字段找出相关表
        // 需要创建普通索引，用于范围查询

        public DateTime CreateTime { get; set; }
    }
}
```
3. 在 Program.cs 中使用 SqlSugarCore

```
#region SqlSugarCore 相关
builder.Services.AddSqlSugar(new SqlSugar.IOC.IocConfig()
{
    ConnectionString = "xxxxx",
    DbType = IocDbType.MySql,
    IsAutoCloseConnection = true
});

// 设置参数
builder.Services.ConfigurationSugar(db =>
{
    // 设置 AOP,打印 Sql
    db.Aop.OnLogExecuting = (sql, p) =>
    {
        Console.WriteLine(sql);
    };
    //设置更多连接参数
    //db.CurrentConnectionConfig.XXXX=XXXX
    //db.CurrentConnectionConfig.MoreSettings=new ConnMoreSettings(){}
    //二级缓存设置
    //db.CurrentConnectionConfig.ConfigureExternalServices = new ConfigureExternalServices()
    //{
    // DataInfoCacheService = myCache //配置我们创建的缓存类
    //}
    //读写分离设置
    //laveConnectionConfigs = new List<SlaveConnectionConfig>(){...}

    /*多租户注意*/
    //单库是db.CurrentConnectionConfig 
    //多租户需要db.GetConnection(configId).CurrentConnectionConfig 
});
// 初始化先创建表
// DbScoped.SugarScope.CodeFirst.InitTables(typeof(BillsMonth));
#endregion
```
4. 在 WebApi 中启用
需要注意的地方：  
● [Route("[controller]/[action]")] 开启所有方法 swagger 都可以解析出来，加了一个 [action]  
● 插入数据的时候如果表不存在会自动建表，并启用雪花 ID 生成分布式 ID 主键。  
● 序列号的生成规则这边需要额外注意，是 GUID+时间戳，这样方便后续查询快速定位到具体表，避免所有水平拆分的表连表查询（解析方法见 GetTableBySN(string serialNumber))，这边写了两个方法，一个是自己实现的，一个是使用 SqlSugar 帮助类实现的。  
SELECT `Id`,`SerialNumber`,`PayMoney`,`BillsInfo`,`BillDate`,`BillComment`,`MachineSerialNo`,`CreateTime` FROM `BillsMonth_20230701`  
WHERE ( `SerialNumber` = @SerialNumber0 )
● 根据时间过滤，这边需要注意的是将查询限定条件写在 SplitTable 之前，这样会先过滤在合并  

```
using Albert.SqlOperateSplitDemo.Models;
using Microsoft.AspNetCore.Mvc;
using SqlSugar.IOC;

namespace Albert.SqlOperateSplitDemo.Controllers
{
    [ApiController]
    [Route("[controller]/[action]")]
    public class SqlSugarTestController : ControllerBase
    {
        private readonly ILogger<SqlSugarTestController> _logger;

        public SqlSugarTestController(ILogger<SqlSugarTestController> logger)
        {
            _logger = logger;
        }

        [HttpGet(Name = "InsertData")]
        public ActionResult InsertData()
        {
            var bilisMonthList = new List<BillsMonth>();

           

            for (int i = 0; i < 365; i++)
            {
                var dtActual = DateTime.Now.AddDays(new Random().Next(1, 365));

                var bill = new BillsMonth()
                {
                    // 注意这边我是通过 Guid 和 时间戳生成的唯一 ID,下次就可以直接通过反解析定位到具体的表了
                    SerialNumber = Guid.NewGuid().ToString()+ ";"+ dtActual.ToFileTimeUtc().ToString(),
                    PayMoney = i * (new Random().Next(1, 100)),
                    BillsInfo = $"随便来点信息--{i}",
                    BillComment = $"随便来点备注--{i}",
                    MachineSerialNo = Guid.NewGuid().ToString(),
                    CreateTime = dtActual,
                };

                bilisMonthList.Add(bill);
            }

            // 必须要加 SplitTable，返回雪花 ID 并自动赋值 ID
            var snowFlake =  DbScoped.SugarScope.Insertable(bilisMonthList).SplitTable().ExecuteReturnSnowflakeIdList();

            return Ok(snowFlake);
        }

        /// <summary>
        /// 根据 SN 查询数据
        /// </summary>
        /// <param name="serialNumber"></param>
        /// <returns></returns>
        [HttpGet(Name = "QueryBySN")]
        public ActionResult QueryBySN(string serialNumber)
        {
            var tableName = GetTableBySN(serialNumber);
            var bill = DbScoped.SugarScope.Queryable<BillsMonth>().AS(tableName).Where(x=>x.SerialNumber== serialNumber).ToList();
            return Ok(bill);
        }

        /// <summary>
        /// 根据时间过滤数据
        /// </summary>
        /// <param name="startTime"></param>
        /// <param name="endTime"></param>
        /// <returns></returns>
        [HttpGet(Name = "QueryByTime")]
        public ActionResult QueryByTime(DateTime startTime,DateTime endTime)
        {
            ////结合Where
            var list = db.Queryable<OrderSpliteTest>().Where(it => it.Id > 0).SplitTable(beginDate, endDate).ToPageList(1, 2);
            Conditions as much as possible
            var bill = DbScoped.SugarScope.Queryable<BillsMonth>().SplitTable(startTime, endTime).OrderBy(x=>x.CreateTime).ToList();
            return Ok(bill);
        }

        / <summary>
        / According to 流水号 get the timestamp
        / </summary>
        / <param name="serialNumber"></param>
        / <returns></returns>

        private string GetTableBySN(string serialNumber)
        {
            var splitArr = serialNumber.Split(';');
            if(splitArr.Length > 1) {
                var dtSpan = long.Parse(splitArr[1]);
                var dtActual = DateTime.FromFileTimeUtc(dtSpan);
                The following two methods are possible
                var dtTableName = DbScoped.SugarScope.SplitHelper<BillsMonth>().GetTableName(dtActual);
                var dtTableName = typeof(BillsMonth).Name + "_" + dtActual.Date.AddDays(1 - dtActual.Day).ToString("yyyyMMdd");
                return dtTableName;
            }

            return "";
        }
    }
}
```