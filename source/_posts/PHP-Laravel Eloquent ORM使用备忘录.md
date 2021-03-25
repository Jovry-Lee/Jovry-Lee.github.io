---
title: PHP-Laravel Eloquent ORM使用备忘录
date: 2021-03-23 15:44:36
tags: ["PHP","Laravel"]
categories: ["PHP","Laravel"]
---

记录Laravel ORM相关一些常用或者不常用的命令或方法，方便后续直接查看．．．

<!--more-->

#### 1 嵌套关联查询

示例：

首先在Model类中定义表的关联关系

```
class CfcV2OrderModel
{
	...

	public function orderDetail()
	{
    return $this->hasMany(CfcV2OrderDetailModel::class, 'orderUri', 'uri');
	}
	...
}
```

```
class CfcV2OrderDetailModel
{
	...
	
	public function detailProperty()
	{
    return $this->hasMany(CfcV2DetailPropertyModel::class, 'detailId', 'id');
	}
	...
}
```

指定查询字段，关联查询。

```
CfcV2OrderModel::with(['orderDetail' => function ($query) use ($orgPid, $orgId, $libKwds) {
    $query->with(['detailProperty' => function ($subQuery) use ($orgPid, $orgId, $libKwds) {
        $subQuery->select('detailId', 'val', 'kwd')
            ->whereIn('kwd', $libKwds)
            ->where(['organizationPid' => $orgPid, 'organizationId' => $orgId]);
    }])->select('orderUri', 'id', 'certType', 'profileJson', 'isInproperty')
        ->where('isCanceled', 0);
}])->useWritePdo()
    ->select('deliveryMoney', 'uri')
    ->whereIn('uri', $uris)
    ->get();
```

<u>Relation 是通过两条语句实现的，它会先查主表，选出其中关联的字段组成数组，使用 in 语句去查询关联表，然后将查出来的数据根据关联字段相等，处理到一起。因此，对于关联查询, 因此关联查询的指定字段必须包含关联字段。</u>



#### 2 获取查询SQL

Laravel提供了一个`toSql`方法，可打印出查询SQL，但该方法为编译SQL，并未携带绑定参数。可使用以下方式进行SQL打印。

```
$builder = DB::table('user')->where('id', 1);
$bindings = $builder->getBindings();
$sql = str_replace('?', '%s', $builder->toSql());
$sql = sprintf($sql, ...$bindings);
dd($sql);
```





------

参考资料

1. [laravel 中的 toSql 获取带参数的 sql 语句](http://www.xiaosongit.com/index/detail/id/727.html)

