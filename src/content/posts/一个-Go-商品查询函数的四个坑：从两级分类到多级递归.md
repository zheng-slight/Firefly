---
title: 一个 Go 商品查询函数的四个坑：从两级分类到多级递归
published: 2026-09-11
description: 从 Find(&goodsCate) 写错到递归收集后代 id，一次讲清多级分类查询的四个坑
tags: [Go, GORM, 商品分类, 多级分类, 递归, SQL, 代码重构]
category: 后端开发
---

# 这段 Go 商品查询代码，坑了不少人

先看代码，别急着往下翻，自己找找问题在哪。

```go
func GetGoodsByCategory(cateId int, goodsType string, limitNum int) []Goods {
	//判断是否顶级分类
	goodsCate := GoodsCate{Id: cateId}
	DB.Find(&goodsCate)
	var tempSlice []int
	if goodsCate.Pid == 0 { // 说明是顶级分类,则需要获取其下面的二级分类
		goodsCateList := []GoodsCate{}
		DB.Where("pid = ?", goodsCate.Id).Find(&goodsCate)
		//把二级分类id存入切片
		for i := 0; i < len(goodsCateList); i++ {
			tempSlice = append(tempSlice, goodsCateList[i].Id)
		}
	}
	tempSlice = append(tempSlice, goodsCate.Id)
	where := "cate_id in ?"

	switch goodsType {
		case "is_best":
			where += " AND is_best = 1"
		case "is_hot":
			where += " AND is_hot = 1"
		case "is_new":
			where += " AND is_new = 1"
		default:
			break
	}
	goodsList := []Goods{}
	DB.Where(where, tempSlice).Order("sort DESC").Select("id, title, price, goods_img, sub_title").Limit(limitNum).Find(&goodsList)
	return goodsList
}
```

需求很简单：给一个分类 id，把该分类下的商品查出来，可以顺带筛"精品/热销/新品"，限制条数。

看着也没啥毛病，但这段代码在多级分类下基本是废的。下面一个个说。

## 坑一：`Find` 的接收对象写错了

```go
goodsCateList := []GoodsCate{}
DB.Where("pid = ?", goodsCate.Id).Find(&goodsCate)
```

声明了一个切片 `goodsCateList`，结果 `Find` 里塞的是 `&goodsCate`，一个单个结构体。GORM 会把查到的第一条结果覆盖到 `goodsCate` 上，`goodsCateList` 从头到尾都是空的。

然后：

```go
for i := 0; i < len(goodsCateList); i++ {
```

`len(goodsCateList)` 等于 0，循环一次都不进。子分类一个都没收集到。

正确写法：

```go
DB.Where("pid = ?", goodsCate.Id).Find(&goodsCateList)
```

这是最直接的 bug，一眼能看出来，但写的时候特别容易手滑——变量名 `goodsCate` 和 `goodsCateList` 太像了。

给个小建议：要么把切片改名 `children`，要么把单个对象改名 `currentCate`，名字拉开距离，下次就不容易看走眼。

## 坑二：`WHERE pid = ?` 后面传谁

不少人看到 `pid = ?` 第一反应是"那第二个参数应该传 `goodsCate.Pid` 啊"。

不是。

`WHERE pid = ?` 是要找"哪些分类的 pid 等于某个值"。我们要找的是当前分类的子分类，子分类的特征是"它的 pid 指向当前分类的 id"。所以传的是 **`goodsCate.Id`**。

传 `goodsCate.Pid` 会怎样？举个例子：

| id   | name | pid  |
| ---- | ---- | ---- |
| 1    | 数码 | 0    |
| 2    | 手机 | 1    |
| 3    | 电脑 | 1    |

当前分类是"数码"（id=1, pid=0）。传 `goodsCate.Pid` 就是 `WHERE pid = 0`，查出来是所有顶级分类（包括它自己），而不是它的子分类，完全反了。

## 坑三：`Pid != 0` 就直接放弃收集子分类

```go
if goodsCate.Pid == 0 {
    // 只有顶级分类才去找子分类
}
```

这段代码的潜台词是：分类只有两级，顶级下面挂二级，二级下面不会再有东西。

但现实里分类经常是三级甚至更多：

| id   | name | pid  |
| ---- | ---- | ---- |
| 1    | 数码 | 0    |
| 2    | 手机 | 1    |
| 3    | 苹果 | 2    |

商品挂在"苹果"（id=3）上。

现在调 `GetGoodsByCategory(2, ...)` 查"手机"：

- `goodsCate = {Id:2, Pid:1}`
- `Pid != 0`，不进 if，不去找"苹果"
- `tempSlice = [2]`
- SQL 变成 `cate_id IN (2)`
- 挂在"苹果"（id=3）上的商品，一条都查不到

调 `GetGoodsByCategory(1, ...)` 查"数码"呢？

- `Pid == 0`，进 if，但因为坑一，子分类还是空的
- `tempSlice = [1]`
- SQL 变成 `cate_id IN (1)`
- "手机""苹果"下的商品，全查不到

看出来了吧——**不管哪一级，只要商品挂在更深层的子分类上，这个函数都查不出来。** 判断顶级分类这件事本身就是在假设层级只有两层，一旦分类变深就崩。

## 坑四：`tempSlice = append(tempSlice, goodsCate.Id)` 到底对不对

最后一个其实不算坑，但很多人看到这里会犹豫，所以单独拎出来说一句。

**它是对的，别误会。**

`cate_id IN (...)` 里的列表必须包含当前分类自己，因为商品可能直接挂在这个分类下。如果漏了它，那些直属本分类的商品就查不到了。

要注意的是这行代码在 `if` 外面，所以不管 `Pid` 是不是 0 都会执行。这也是为什么 `tempSlice` 不会是空的——它至少有自己的 id。

## 正确的写法

别去判断是不是顶级，也别假设只有两级。统一递归，把当前分类和它所有后代分类的 id 全收集起来。

```go
// 递归收集 cateId 及其所有后代分类的 id
func collectCateIds(cateId int, result *[]int) {
	*result = append(*result, cateId) // 先装自己

	var children []GoodsCate
	DB.Where("pid = ?", cateId).Find(&children) // 注意接收切片

	for _, child := range children {
		collectCateIds(child.Id, result) // 递归收集后代
	}
}

func GetGoodsByCategory(cateId int, goodsType string, limitNum int) []Goods {
	var cateIds []int
	collectCateIds(cateId, &cateIds) // 自己 + 所有后代

	query := DB.Model(&Goods{}).Where("cate_id IN ?", cateIds)

	switch goodsType {
	case "is_best":
		query = query.Where("is_best = 1")
	case "is_hot":
		query = query.Where("is_hot = 1")
	case "is_new":
		query = query.Where("is_new = 1")
	}

	var goodsList []Goods
	query.Order("sort DESC").
		Select("id, title, price, goods_img, sub_title").
		Limit(limitNum).
		Find(&goodsList)

	return goodsList
}
```

验证一下前面那个三级分类的例子：

- `GetGoodsByCategory(1, ...)`：collectCateIds(1) → `[1]` → 找子分类 [2] → `[1,2]` → 找 [3] → `[1,2,3]` ✅
- `GetGoodsByCategory(2, ...)`：`[2]` → 找 [3] → `[2,3]` ✅
- `GetGoodsByCategory(3, ...)`：`[3]` ✅

顶级、二级、三级全对。再也不用去关心 `Pid` 是不是 0。

## 改了什么

1. **`Find` 接收切片**：`&goodsCateList`，不是 `&goodsCate`。
2. **去掉顶级分类判断**：递归收集，任何层级都走同一套逻辑。
3. **递归收集后代 id**：自己 + 所有子孙，一步到位。
4. **条件改用 GORM 链式 API**：`query.Where(...)` 一层层叠，比字符串拼 `where` 好维护，也不容易拼错。
5. **软删除和状态过滤**：上面的示例代码里没加。实际业务中建议补上 `is_delete = 0`、`status = 1`，否则下架和删除的商品也会被查出来：

   ```go
   query := DB.Model(&Goods{}).
       Where("cate_id IN ?", cateIds).
       Where("is_delete = 0 AND status = 1")
   ```

## 几个小提醒

**递归要防环。** 如果数据里 pid 不小心成了环（比如 A 的 pid 是 B，B 的 pid 是 A），递归会死循环。加个 `visited map[int]bool`，或者限制最大深度。

**`IN ?` 不用自己加括号。** GORM 会把切片展开成 `(1,2,3)`。写 `IN (?)` 有时也能识别，但标准写法是 `IN ?`。

**变量命名别偷懒。** `goodsCate` 和 `goodsCateList` 差一个单词，看代码的时候眼睛容易滑过去，这种 bug 就是这么来的。

---

这段代码的问题不在语法，而在**它默认分类只有两级**。层级一深，逻辑整个失效。多级分类的场景下，递归收集后代 id 是最省心的做法，也最好懂。