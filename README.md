# bksgpx

> **Author**: 12qwaszx3edc123 (bks)  
> **License**: MIT License  
> **GitHub**: https://github.com/12qwaszx3edc123/bksgpx

Go 微服务脚手架生成工具 - 一键生成完整的微服务项目结构

## 安装

```bash
go install github.com/12qwaszx3edc123/bksgpx/cmd/mygen@latest
```

## 使用方法

```bash
mygen --name <项目名> --modules <模块列表> [其他参数]
```

### 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--name` | `bks` | 项目名称 |
| `--modules` | - | 模块列表，逗号分隔，如 `user,order,doctor` |
| `--bff` | `h5` | BFF 类型: `h5`, `web`, `applet`, `app` |
| `--db-host` | `127.0.0.1` | 数据库主机 |
| `--db-port` | `3306` | 数据库端口 |
| `--db-user` | `root` | 数据库用户名 |
| `--db-password` | `root` | 数据库密码 |
| `--db-name` | - | 数据库名称 |
| `--template` | 内嵌模板 | 外部模板目录路径 |

### 示例

```bash
# 生成名为 myshop 的项目，包含 user 和 order 模块
mygen --name myshop --modules user,order --db-name myshop_db

# 使用外部模板
mygen --name myproject --modules user --template /path/to/templates
```

## 生成的项目结构

```
<项目名>/
├── common/
│   ├── config/      # 配置管理
│   ├── init/        # 初始化
│   └── model/       # 数据模型
├── proto/           # Protobuf 定义
├── <module>-srv/    # 各模块的 gRPC 服务
├── bff<type>/       # BFF 层 (h5/web/applet/app)
└── pkg/             # 公共工具包
```

## 模板

默认使用内嵌模板，无需额外配置。如需自定义模板，可使用 `--template` 参数指定外部模板目录。

---

## ElasticSearch 操作

### ESAdd //ES添加

```go
// 同步 Elastic添加
m := map[string]interface{}{
	"id", "name", "image", "price", "stock",       // 菜品基本信息
	"merchant_id", "type", "month_sales",          // 商家和销量信息
	"rating", "is_new", "status",                  // 状态信息
}
_, err = config.Esc.Index().
	Index("commodity").              // 索引名称
	BodyJson(m).                     // 文档内容
	Do(context.TODO())               // 执行
```

### ESList //ES列表

```go
// 分页
offset, size := pkg.Paginate(int(req.Page), int(req.Size))

// 布尔查询构建
boolQuery := elastic.NewBoolQuery()

// 名称模糊搜索
elastic.NewMatchQuery("name", req.Name).Must(boolQuery)

// ID精准搜索
elastic.NewTermQuery("merchant_id", req.MerchantId).Filter(boolQuery)

// 价格区间查询
elastic.NewRangeQuery("price").Gte(req.MinPrice).Lte(req.MaxPrice).Must(boolQuery)

// 排序：格式 "字段_方向"，如 "Rating_Desc"、"price_Asc"
sortQuery := elastic.NewFieldSort("rating")       // 按评分排序
sortQuery := elastic.NewFieldSort("month_sales")  // 按月销量排序
sortQuery := elastic.NewFieldSort("price")        // 按价格排序
sortQuery.Desc()                                  // 倒序
sortQuery.Asc()                                   // 正序

// 设置高亮
highlight := elastic.NewHighlight()
highlight.Field("name")                          // 高亮哪个字段
highlight.PreTags("<span>")                       // 前置标签
highlight.PostTags("</span>")                     // 后置标签

// 执行搜索
do, err := config.Esc.Search().
	Index("commodity").              // 索引名称
	From(size).                      // 偏移量
	Size(offset).                    // 每页数量
	Query(boolQuery).                // 查询条件
	SortBy(sortQuery).               // 排序规则
	Highlight(highlight).            // 添加高亮
	Do(context.Background())

// 总条数
total := do.Hits.TotalHits

// 结果解析
for _, hit := range do.Hits.Hits {
	json.Unmarshal(hit.Source, &list)  // 反序列化每条记录
	if hit.Highlight != nil {
		highlightedName := hit.Highlight["title"]  // ["<span>感冒</span>灵颗粒"]
	}
}
```

### ESGet //ES查询

```go
// 根据ID查询单条文档
do, err := config.Esc.Get().
	Index("commodity").              // 索引名称
	Id(strconv.Itoa(int(id))).       // 文档ID
	Do(context.TODO())               // 执行

// 解析结果
if do.Found {
	json.Unmarshal(do.Source, result)  // 反序列化文档内容
}
```

### ESUpdate //ES修改

```go
// 根据ID更新文档字段
m := map[string]interface{}{
	"name",   // 修改的字段
	"price",  // 修改的字段
	"stock",  // 修改的字段
}
_, err = config.Esc.Update().
	Index("commodity").              // 索引名称
	Id(strconv.Itoa(int(id))).       // 文档ID
	Doc(m).                          // 更新内容
	Do(context.TODO())               // 执行
```

### ESDelete //ES删除

```go
// 根据ID删除文档
_, err = config.Esc.Delete().
	Index("commodity").              // 索引名称
	Id(strconv.Itoa(int(id))).       // 文档ID
	Do(context.TODO())               // 执行
```



---

## Redis Cart //Redis购物车

Key格式：`cart:{UserId}:{DrugId}`

### CartAdd //购物车添加/修改

```go
key := fmt.Sprintf("cart:%d:%d", DrugId, UserId)   // key格式
config.RDB.HSet(config.Ctx, key, drugMap).Err()    // 添加/修改商品
```

### CartExist //购物车是否存在

```go
key := fmt.Sprintf("cart:%d:%d", UserId, DrugId)   // key格式
config.RDB.Exists(config.Ctx, key).Result()        // 返回true/false
```

### CartUpdate //购物车数量更新

```go
key := fmt.Sprintf("cart:%d:%d", UserId, DrugId)          // key格式
config.RDB.HIncrBy(config.Ctx, key, "qauntity", 1).Err()  // 数量+1
config.RDB.HIncrBy(config.Ctx, key, "qauntity", -1).Err() // 数量-1
```

### CartList //购物车列表

```go
key := fmt.Sprintf("carts:%d:*", UserId)        // 匹配该用户所有商品key
config.RDB.Keys(config.Ctx, key).Val()         // 返回 []string
```

### CartDelete //购物车删除单个

```go
key := fmt.Sprintf("cart:%d:%d", UserId, DrugId)  // key格式
config.RDB.HDel(config.Ctx, key).Err()            // 删除商品
```

### CartClear //购物车清空

```go
key := fmt.Sprintf("cart:%d:*", UserId)             // 匹配该用户所有商品key
keys := config.RDB.Keys(config.Ctx, key).Val()      // 获取所有key
for _, k := range keys {
	config.RDB.HDel(config.Ctx, k).Err()             // 逐个删除
}
```

---

## PayNotify //支付异步回调

### Notify

```go
func Notify(c *gin.Context) {
	err := c.Request.ParseForm()
	if err != nil {
		fmt.Println("参数获取失败")
		c.String(400, "参数解析失败")
	}
	m := make(map[string]string)
	for s, strings := range c.Request.PostForm {
		m[s] = strings[0]
	}
	fmt.Println(m)
	if m["trade_status"] != "TRADE_SUCCESS" {
		c.String(200, "交易失败")
	}
	orderSn := m["out_trade_no"]
	if orderSn == "" {
		c.String(200, "订单号不存在")
	}
	var order model.Order

	if err := order.FindOrderByOrderSn(config.DB, orderSn); err != nil {
		c.String(400, "订单不存在")
		return
	}
	tx := config.DB.Begin()
	order.OrderType = 2
	order.PayType = 2

	if err := order.OrderSave(tx); err != nil {
		c.String(200, "订单状态更新失败")
		return
	}
	var drug model.DrugAdd

	if err := drug.FindDrugById(tx, int64(order.DrugId)); err != nil {
		c.String(200, "药品信息不存在")
		return
	}
	drug.Stock = order.Number

	if err := drug.Save(tx); err != nil {
		c.String(200, "修改失败")
		return
	}
	tx.Commit()
	c.String(200, "交易成功")
}
```

---

## Schedule //排班管理

### ScheduleAdd

```go
func (s *DoctorService) ScheduleAdd(ctx context.Context, req *pb.ScheduleAddRequest) (resp *pb.ScheduleAddResponse, err error) {
	var doctor model.Doctor
	err = doctor.FindDoctorById(config.DB, req.DoctorId)
	if err != nil {
		return nil, errors.New("医生不存在")
	}

	var department model.Department
	err = department.FindDepartmentById(config.DB, req.DepartmentId)
	if err != nil {
		return nil, errors.New("科室不存在")
	}

	parseDate, _ := time.Parse("2006-01-02", req.ScheduleDate)

	nowData := time.Now().AddDate(0, 0, 1).Truncate(24 * time.Hour)
	if parseDate.After(nowData) {
		return nil, errors.New("排班不能为过去的时间")
	}

	var schedule model.Schedule
	err = schedule.FindScheduleById(config.DB, req.DoctorId, req.DepartmentId, req.ScheduleDate, req.ScheduleTime)
	if err != nil {
		return nil, errors.New("不能重复排班")
	}

	schedule = model.Schedule{
		DoctorId:     req.DoctorId,
		DepartmentId: req.DepartmentId,
		ScheduleDate: &parseDate,
		ScheduleTime: req.ScheduleTime,
		TotalNum:     int64(req.TotalNum),
		UserNum:      1,
		Status:       1,
	}

	err = schedule.Add(config.DB)
	if err != nil {
		return nil, errors.New("排班失败")
	}

	return &pb.ScheduleAddResponse{Id: int64(schedule.ID)}, nil
}
```

---

## Appointment //预约管理

### AppointmentAdd

```go
func (s *UserService) AppointmentAdd(ctx context.Context, req *pb.AppointmentAddRequest) (resp *pb.AppointmentAddResponse, err error) {
	// 1.时间格式
	parseDate, _ := time.Parse("2006-01-02", req.ScheduleDate)
	nowDate := time.Now().AddDate(0, 0, 1).Truncate(24 * time.Hour)
	if parseDate.After(nowDate) {
		return nil, errors.New("不能预约过去的日期")
	}

	tx := config.DB.Begin()

	// 2.不能重复预约
	var appointment model.Appointment
	count := appointment.FindAppointment(tx, req.UserId, req.DoctorId, req.ScheduleId, req.ScheduleDate, req.ScheduleTime)
	if count > 0 {
		return nil, errors.New("不能重复预约")
	}

	// 3.查询医生状态
	var doctor model.Doctor
	err = doctor.FindDoctorById(tx, req.DoctorId)
	if err != nil {
		return nil, errors.New("医生不存在")
	}

	if doctor.Status != 1 {
		return nil, errors.New("医生未出诊")
	}

	var schedule model.Schedule
	err = schedule.FindSchedule(tx, req.ScheduleId)
	if err != nil {
		return nil, errors.New("排班信息不存在")
	}

	if schedule.UserNum >= schedule.TotalNum {
		return nil, errors.New("号源不足")
	}

	appointmentNo := pkg.AppointmentSn()
	price := doctor.Service

	expireTime := time.Now().Add(30 * time.Minute)

	// 1.添加预约
	appointment = model.Appointment{
		Model:           gorm.Model{},
		UserId:          req.UserId,
		DoctorId:        req.DoctorId,
		ScheduleId:      req.ScheduleId,
		AppointmentDate: &parseDate,
		AppointmentTime: req.ScheduleTime,
		AppointmentNo:   appointmentNo,
		Status:          2,
		Total:           price,
		PayTypes:        1,
		ExpireTime:      &expireTime,
	}

	if err = appointment.Add(tx); err != nil {
		return nil, errors.New("预约记录添加失败")
	}

	// 2.扣减号源
	schedule.UserNum--

	if err = schedule.Save(tx); err != nil {
		return nil, errors.New("号源更新失败")
	}

	err = tx.Commit().Error
	if err != nil {
		return nil, errors.New("事务提交失败")
	}

	PayUrl := pkg.Pay(appointmentNo, price)

	return &pb.AppointmentAddResponse{
		AppointmentNo: appointmentNo,
		Total:         float32(price),
		PayUrl:       PayUrl,
	}, nil
}
```

### RestoreNum //定时任务-号源恢复

```go
func RestoreNum() {
	fmt.Println("我是计划任务")

	var appointment []model.Appointment
	config.DB.Where("expire_time < ? AND status = ?", time.Now(), 3).Find(&appointment)

	if len(appointment) > 0 {
		for _, app := range appointment {
			app.Status = 1
			err := app.Save(config.DB)
			if err != nil {
				return
			}

			var schedule model.Schedule
			err = config.DB.Where("id = ?", app.ScheduleId).First(&schedule).Error
			if err != nil {
				return
			}

			schedule.UserNum += 1

			err = schedule.Save(config.DB)
			if err != nil {
				return
			}

		}
		fmt.Println("号源释放成功")
	}
}
```

### CronStart //定时任务启动

```go
// 创建cron调度器
c := cron.New()

// 添加定时任务，每分钟执行一次
_, err := c.AddFunc("* * * * *", api.RestoreNum)
if err != nil {
	fmt.Println("计划任务定义失败")
}

// 启动调度器
c.Start()

// 程序结束时停止调度器
defer c.Start()
```

---

## TimeFormat //时间格式转换

```go
// time.Time 转字符串
timeObj.Format("2006-01-02 15:04:05")  // "2026-05-18 15:39:00"
timeObj.Format("2006-01-02")            // "2026-05-18"
timeObj.Format("15:04:05")              // "15:39:00"
timeObj.Format(time.RFC3339)            // "2026-05-18T15:39:00+08:00"

// 时间戳转字符串
strconv.FormatInt(timeObj.Unix(), 2)    // "1747550340"

// 字符串转 time.Time
parseDate, _ := time.Parse("2006-01-02", "2026-05-18")
parseDate, _ := time.Parse("2006-01-02 15:04:05", "2026-05-18 15:39:00", time.UTC)
```

---

## RedisCache //Redis缓存

Key格式：`cache:{prefix}:{id}`

### CacheSet //缓存设置

```go
key := fmt.Sprintf("cache:user:%d", userId)              // key格式
config.RDB.Set(config.Ctx, key, value, 0).Err()           // 永久缓存
config.RDB.SetNX(config.Ctx, key, value, time.Hour).Err()  // 1小时过期
```

### CacheGet //缓存获取

```go
key := fmt.Sprintf("cache:user:%s", userId)   // key格式
config.RDB.Get(config.Ctx, key).Result()      // 获取缓存值
```

### CacheExist //缓存是否存在

```go
key := fmt.Sprintf("cache:user:%d", userId)          // key格式
config.RDB.Exists(config.Ctx, key).Result()          // 返回true/false
```

### CacheDel //缓存删除

```go
key := fmt.Sprintf("cache:user:%d", userId)  // key格式
config.RDB.HDel(config.Ctx, key).Err()        // 删除缓存
```

### CacheExpire //缓存过期时间

```go
key := fmt.Sprintf("cache:user:%d", userId)              // key格式
config.RDB.Expire(config.Ctx, key, time.Hour).Result()    // 设置1小时过期
config.RDB.PTTL(config.Ctx, key).Val()                   // 获取剩余过期时间
```

### CacheList //缓存列表（批量获取）

```go
keys := []string{"cache:user:1", "cache:user:2", "cache:user:3"}
config.RDB.Get(config.Ctx, keys...).Val()               // 批量获取多个缓存
```

### CacheListJSON //列表缓存存取（JSON方案）

```go
// 存入列表缓存 - bks
func SetListCache(key string, list interface{}, expire time.Duration) error {
	data, err := json.Marshal(list)
	if err != nil {
		return err
	}
	return config.RDB.SetNX(config.Ctx, key, data, expire).Err()
}

// 查询列表缓存 - bks
func GetListCache(key string, result interface{}) (bool, error) {
	data, err := config.RDB.Get(config.Ctx, key).Bytes()
	if err == redis.Nil {
		return false, nil // 缓存不存在
	}
	if err != nil {
		return false, err
	}
	return true, json.Unmarshal(data, &result)
}

// 使用示例 - bks
key := fmt.Sprintf("cache:commodity:list:%d", merchantId)
SetListCache(key, commodityList, 30*time.Minute)  // 存入缓存

var list []model.Commodity
exists, _ := GetListCache(key, list)               // 查询缓存
if !exists {
	db.Find(&list)                                   // 查数据库
	SetListCache(key, list, 30*time.Minute)         // 存入缓存
}
```

### CacheListRedis //列表缓存存取（Redis List方案）

```go
// 存入列表 - bks
key := "cache:commodity:list"
for _, item := range list {
	data, _ := json.Marshal(item)
	config.RDB.RPush(config.Ctx, key, data).Err()     // 逐条存入
}

// 查询列表（分页） - bks
offset := 0
size := 10
results := config.RDB.LRange(config.Ctx, key, int64(offset), int64(offset+size)).Val()
for _, data := range results {
	json.Unmarshal([]byte(data), item)
}

// 获取总条数 - bks
total := config.RDB.LLen(config.Ctx, key).Int()
```
### CacheListRedis //1v1
package handler

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"github.com/gorilla/websocket"
	"log"
	"sync"
)

// 协议升级
var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

type Node struct {
	Conn *websocket.Conn // 长连接  ws
	Data chan []byte     // 用于传递消息
}

var onLineMap = make(map[string]*Node) // 在线用户节点映射
var mu sync.RWMutex

func Chat(c *gin.Context) {
	formUserId := c.Query("from_user_id")
	toUserId := c.Query("to_user_id")
	fmt.Println(formUserId, toUserId)

	conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
	if err != nil {
		log.Println(err)
		return
	}

	// 创建节点
	node := &Node{
		Conn: conn,
		Data: make(chan []byte),
	}

	mu.Lock()
	onLineMap[formUserId] = node // 加入在线用户节点映射
	mu.Unlock()

	go ReceiveMessage(node)                    // 接收消息，读我自己的chan消息，通过 ws 写给我
	go SendMessage(node, formUserId, toUserId) // 发送消息，读我的消息放到别人的chan中
}

// 发送消息, 向指定用户发送消息
func SendMessage(node *Node, userId, targetId string) {
	for {
		_, p, err := node.Conn.ReadMessage() // 读取我的消息
		if err != nil {
			log.Println(err)
			// 删除在线用户.从 `onLineMap` 删除该用户
			delete(onLineMap, userId)
			return
		}

		if _, ok := onLineMap[targetId]; !ok {
			log.Println("目标用户不存在")
			return
		}

		mu.RLock()
		onLineMap[targetId].Data <- p // 发送消息给目标用户
		mu.RUnlock()
	}
}

func ReceiveMessage(node *Node) {
	for {
		p := <-node.Data // 接收消息
		err := node.Conn.WriteMessage(websocket.TextMessage, p)
		if err != nil {
			log.Println(err)
		}
	}
}

---

## License

MIT License
