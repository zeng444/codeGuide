#TFF Yii 开发规范

| 版本 |   更新日志  |     日期    | 作者   | 
|:---:|:-----------|------------|--------| 
| 0.1 | 初始版      | 2015-01-29 | Robert |           

## 关键词

| 关键词  |    描述                           |    
|:----------:|:----------------------------------|
|  禁止    |   绝对不允许的行为          |
|  必须    |   需要按照标准执行的行为 |
|  建议    |   推荐使用的方式             | 

## Controller

### 数据与处理

- 对接收的数据处理，**必须**使用YII提供的方法，预防变量unset抛出的notice错误

```
Yii::app()->getQuery("id");
Yii::app()->postQuery("id");
Yii::app()->getParam("id");
```

- 对$_SERVERS全局变量的处理，**建议**使用Yii::app->request对象下的方法
- 控制器接收的数据，**禁止**在控制器内做验证，应该在Model（CformModel或者activeRecord model）对象的Rule规则处理

```
$model = new LoginformModel();
$model->setAttributes($_POST);
if( $model->validate() ){
    //handle $model->getErrors();
}
```

-  **禁止**在控制器内对模型数据进行处理


- **禁止**直接在控制器内使用CDbCommand获取数据，至少应该从业务模型获取连接

```
Yii::app->db->createCommand()
$model::model()->getConnection()->createCommand()
```

- **禁止** 使用在控制器直接使用CDbCommand对象做SQL查询
- **禁止**使用控制器下同一个action处理多个页面和业务逻辑
- **禁止**在控制器内撰写复杂业务逻辑的封装，优先考虑在model层处理，其次考虑控制器内引入actions，或behaviors对象，最次定义private 方法
- 对复杂action的逻辑优先使用filters控制,最好不要使用__construct中处理

### ActiveRecord

#### Extend and rewrite


- 业务逻辑模型禁止直接继承CActiveRecord，应该使用TFFBaseModel
-  TFFBaseModel提供CActiveRecord功能的增强和作用域内的健壮度检查
- 使用策略时使用行为对象，必须先定义interface，对行为对象进行约束

#### Rule 

* 按业务逻辑或数据库字段规则，定义验证规则，对无规则的属性要声明safe

```
//基于表字段的规则约束
array("title ,name , customer_id","length","max"=>255),
array("customer_id , numerical","integerOnly"=>true),
//基于业务逻辑的约束
array("title, name , created ,content ,last_update","required"),
array("customer","unique"), //业务逻辑要求是表中不存在的
array("content","safe")
```

```
array("title","length","max"=>"12","message"=>"This is a message!")
```

* 对需要抛出错误信息的字段定义Message
* 对业务逻辑明确的字段在rule定义数据预处理的方法（也可以在beforeSave处理）

```
array("image_data","dataSerilize")
function dataSerilize(){}
```

* 尽量使用model->setAttributes()填充属性，这样能对rule的定义是否完整做验证，而不要使用setAttribute，或者直接赋值
* 对同字段存在不同处理情况的，应该定义场景使用多组规则
* 多个模型都使用的验证方法，使用新的验证类，再在个个模型rule中调用
- 查询数据处理，对固定数据加工的逻辑，尽量使用afterFind处理，减少控制器和模版处理原始数据的逻辑 
- 不**建议**使用CDbCommand下的方法，queryRow,queryAll,queryScalar来处理获取数据库数据，pdo返回的原始数组数据，不会经过afterFind数据处理，而对数据丢失约束，造成控制器或者模版重复处理
* 对日期字段update create类型的字段，尽可能使用dateAttribute()实现自动处理 


#### Relationship

* 必须对关系表进行定义
* 获取关联表数据尽量使用relationship定义的方法

#### Event

- 对过程处理尽量使用afterValidate,beforeFind,afterSave这类函数，而尽量不要在业务逻辑中通过注册事件的方式处理，
可能会造成对维护事件变得复杂化

```
//推荐
function beforeFind(){
    //业务逻辑
    parent::beforeFind()
}
//不推荐
$this->attachEventHandler("onBeforeFind",array($this,"xxx");
```

### FormModel



## Widget





## Views

## Code Guide
