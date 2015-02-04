#TFF Yii 开发规范

| 版本 |   更新日志  |      日期        |   作者    | 
|:------:|:---------------|-------------------|------------| 
| 0.1  | 初始版       | 2015-01-29  | Robert |           

>> 本规范定义之禁止、建议、必须等要求，经过业务组人员讨论制订，为提供一组统一的团队开发合作标准，部分规则可能会涉及到个人使用习惯，我们以团队利益优先。

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
- 禁止在析构和构造函数内获取任何模型数据同理包括beforeAcion afterAction(难以扩展，API接口重灾区)
- 对$_SERVERS全局变量的处理，**建议**使用Yii::app->request对象下的方法
- 控制器接收的数据，**禁止**在控制器内做验证，应该在Model（CformModel或者CActiveRecord）对象下的Rule规则处理


```
$model = new LoginFormModel();
$model->setAttributes($_POST);
if( $model->validate() ){
    //handle $model->getErrors();
}
```

-  **禁止**在控制器内处理业务数据，比如_GET _SET类的数据加工，数据验证条件等
- **禁止**使用控制器下同一个action处理多个页面和业务逻辑（网站联盟）
- **禁止**在控制器内撰写复杂业务逻辑，控制器只能放和当前模版渲染数据强藕合的方法，比如处理页面输出结构、封装API数据节点等。
**禁止**在controller中拼接大段的html代码，如需要较长的html代码，**必须**使用模板

```
$partialContent = $this->renderPartial('partial', $data, true);
```

- 出现控制器大量逻辑代码的处理方式，**建议**在控制器内引入actions，或behaviors，允许少量逻辑代码在控制器内定义方法（是否应该禁止？）
- 对控制器action方法限制和过滤，**建议**使用filters控制,最好不要使用__construct中处理
 


## Model


#### 通用

 - 禁止在模型里处理$_GET $_POST $_COOKIE等全局变量


####   继承重写


- 业务逻辑模型**禁止**直接继承CActiveRecord、CFormModel，应该继承BaseModel
-  BaseModel提供CActiveRecord功能的增强和作用域内的健壮度检查
-  复杂的模型，建议使用行为将方法分离（行为我们应该推广么？感觉容易把方法引用变的混乱）
-  模型内使用策略时**建议**使用行为对象，但**必须**先定义interface实现，对行为对象进行约束

#### Rule 

- 按业务逻辑或数据库字段规则，定义验证规则，对无规则的属性要声明safe

```
//基于表字段的规则约束
array("title ,name , customer_id","length","max"=>255),
array("customer_id , numerical","integerOnly"=>true),
//基于业务逻辑的约束
array("title, name , created ,content ,last_update","required"),
array("customer","unique"), 
array("content","safe")
```

* 对需要定义业务逻辑验证错误信息的字段**建议**在rules里定义Message


```
array("title","length","max"=>"12","message"=>"This is a message!")
```



* 对业务逻辑明确的字段在rule定义数据预处理的方法（也可以在beforeSave处理？）

```
array("image_data","dataSerilize")
function dataSerilize(){}
```
- Model内操作获取对象属性**推荐**使用setAttribute getAttribute，不要直接使用对象属性方式访问
- **建议**使用setAttributes()填充属性，这样能对rule的定义是否完整做验证，而不要使用setAttribute()，或者直接赋值
- 对同字段存在不同验证规则的情况，**建议**定义场景使用多组规则（场景过多混乱的问题？）
- 多个模型都使用的验证方法，**建议**使用新的验证类，再在个个模型rules中复用

#### ActiveRocord

- 查询数据后的处理，对和数据表强藕合的数据加工，**建议**使用afterFind处理，减少控制器和模版负担 （比如序列化存入的数据，一定读取后是需要反序列的）
- 插入数据的加工**建议**在beforeSave做处理
- 禁止直接静态访问调用ActiveRecord下table()方法

```
//禁止
Model::tableName()
//正确
Model::model()->tableName()

```
- 对模型内的钩子方法（beforeFind,beforeValidate,afterFind...），**必须**返回父方法，否则会丢失attachEventHandle注册的事件 
- 不**建议**使用CDbCommand下的方法，queryRow,queryAll,queryScalar来处理获取数据库数据，pdo返回的原始数组数据，不会经过afterFind数据处理，而对数据丢失约束，造成控制器或者模版重复处理
- 只有对复杂SQL语句比如多表联查，复杂统计，activeRecord层难以实现的数据库操作才可以使用CDbCommand对象提供的底层数据库操作方法
- 如果使用CDbCommand对象，应该从业务模型获取连接，**禁止**直接获取组件

```
//禁止
Yii::app->db->createCommand()
//推荐
$model::model()->getConnection()->createCommand()
```
- 使用CDbCommand 对象获取表数据，不**建议**再使用sql构造函数,如form、order、where、limit、join 、select，因为已经够底层了，使用构造SQL函数反而带来无意义的开销

```
//不推荐
$model::model()->dbConnection->createCommand()
->select("*")
->from("table")
->where("id=1")
->limit(1)
->queryRow();

//推荐
$model::model()->dbConnection
->createComand("select * from  table where id=1 limit 1")
->queryRow();
```

- 无论使用CDbCommand还是使用CActiveRecord获取数据库数据，**建议**定义"select"的列条件，不要所有列都全部返回（这个看需要不，以后数据量访问量大，比较容易造成和MYSQL通讯的数据拥堵）

- 对日期字段update create类型的字段，尽可能使用dateAttribute()实现自动处理 

#### ActionRecord Relationship

- 必须对关系表进行定义
- 获取关联表数据尽量使用relationship定义的方法
- 多组的relationship
- 复杂条件的relationShip表关系，**推荐**定义成对象方法


### CFormModel

- 是否要求对表单数据处理强制formModel，把formModel作为数据验证器和表单生成器（这条讨哈，貌似会增加繁琐度，我们现在表单复用性本身不强）

## Event

- 对过程处理尽量使用afterValidate,beforeFind,afterSave这类方法，而尽量不要在对象内通过官方提供的注册事件处理(可能会造成对维护变得复杂化)
- Event 官方事件允许在对象外注册使用

```
//推荐
function beforeFind(){
    //业务逻辑
    parent::beforeFind()
}
//推荐
model::model()->attachEventHandler("onBeforeFind",array($obj,"method");

//不推荐
$this->attachEventHandler("onBeforeFind",array($obj,"method");
```


## Widget

- 禁止直接继承使用CWidget，应该使用BaseWidget继承来编写新的widget控件 
- 每个widget应该具有独立的目录，放在application.projected.widget下，views**必须**在当前widget目录的下一级

```
bannerWidget的目录结构为例：
projected/widgets/bannerWidget/bannerWidget
projected/widgets/bannerWidget/bannerWidget/views/banner.html
```
- Widget **必须**符合独立的一个业务逻辑单元的特点，禁止一个Widget处理不同的业务单元
- Widget如果具有html结构**必须**使用模版，模版必须使用twig的html格式
- Widget 对外暴露参数,**必须**符合易传递，容易迁移的要求，**推荐**传递为单一对象，单一id一类的参数，不推荐使用在控制器内已通过逻辑加工过的数据作为参数
- Widget依赖的js和css文件**推荐**通过baseWidget提供的方法或Yii 提供的registerCssfiles、registerJsfiles，在widget内部注册(这里可能会有一堆零碎的js css请求，目前准备一张页面内注册的widget可以通过seajs concat来拼接请求单一文件)
- Widget嵌套，子widget和父widget**必须**是松散藕合的


## Views

- 新页面必须使用twig
-  **禁止**在views里编写任何形式的业务逻辑
- **禁止**使用include，复杂页面**建议**使用widget
- **建议**把独立的页面渲染逻辑(页面展示相关逻辑)从controller分散到widget中
- twig模版内对原始数据需要加工的，**建议**在配置文件twig组件配置里注册function或者filters
- **不建议**直接访问$_GET, $_POST或其他类似变量, **建议**放在controller处理


```
viewRender =>array(
   function=> array(
       'md5' =>'md5'
   ),
   'filters'=>array(
       'html_entity_decode'=>'html_entity_decode'
   )
)
``` 

## UrlRule

- 禁止在路由对象里封装业务逻辑

## Interface

* 使用Yii::app->endJson()作为处理接口数据输出的方法（里面实现和统一管理了jsonp 、iframe、json等多种情况的跨域非跨域数据输出）
* 原生json处理**推荐**使用Cjson::encode和Cjson::deconde,**禁止**使用json_encode处理数据（迁移）

## Magic Method 

* **推荐**不使用Yii \_get \_set \_call等提供的魔术方法调用方法（代码可能会看起来比较乱和难以被IDE工具维护,relationship应该是例外吗？）

```
//推荐
Model::model()->getProductCount()
//不推荐
Model::model()->productCount

```

## 目录结构

- components: 功能独立的组件库
- vendors: 第三方库
- extensions: Yii原生类的拓展
- classes: 业务逻辑及其它类



##Debug

* **推荐**使用Yii::trace作为主要的debug断点调试工具（预防exit die之类的忘记删除）

## Code Guide
- **禁止**使用@屏蔽错误
- **建议**字符串使用单引号包裹
- 定义和使用数组时, key值**必须**用单引号包裹
- 使用常量**必须**先做defined判断
```
if( defined('MENU_VERSION' ) && MENU_VERSION){
}
```
- 变量名**必须**表明其含义, **禁止**使用意义不明的缩写

```
$productPrice;
$p_r; //禁止
```
- 变量使用之前**必须**定义, 在条件语句中定义的变量**建议**, 在if之前提前申明
```
$a=''; //提前申明
if ($condition) {
 $a = 'value';
}
$b = $a;
```
- 复杂逻辑的函数和方法**必须**添加注释
- 每一小段逻辑之间**建议**使用空行分割
