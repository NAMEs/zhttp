
# zhttp-用来做http web服务的异步非阻塞轻量级框架


### 开发交流群:384013097

## 新手入门项目，附有源码和线上实例（主要用于介绍）
https://github.com/keaixiaou/zhttptest/

## 使用手册（待完善）
https://keaixiaou.gitbooks.io/zhttp/content/

##协议
MIT license

##	优势
	1.框架基于swoole开发，并且一些IO操作底层已经封装为异步，性能极其强悍。
	2.框架底层已经封装好异步，内置mysql、redis连接池，只需要在调用的时候在前面加yield，近乎同步的写法，却是异步的调用，并且无需关注底层实现，连接数超等问题，使用非常简单。
	
	
## 注意事项

	1.框架最新加入协程+mysql连接池，非阻塞的mysql查询大大提高了框架应对请求的吞吐量
	2.php版本需要7.0+
	3.swoole版本1.8.*
	4.如果用到异步redis，需要安装hiredis，安装教程:http://wiki.swoole.com/wiki/page/p-redis.html

##安装依赖包
	composer install
	1.没有安装composer的先安装composer
	2.不会composer或者不喜欢composer的可以直接去我另一个资源库下载框架依赖,地址：https://github.com/keaixiaou/zphp
	
##运行zhttp

	本框架只支持http模式：
	运行：
	cd 到根目录
	php webroot/main.php start|stop|restart|reload|status
	访问IP:PORT
    建议：
        如果是静态文件如css、js、image，可以直接用nginx代理
        如果是动态请求，最好使用nginx做代理转发
        
###服务监控图
![服务监控](https://raw.githubusercontent.com/keaixiaou/base/master/status.jpeg)

    
```
 nginx示例配置：
    server {
        listen       80;
        server_name  zhttp.test.com;
    root /Users/zhaoye/workspace/zhttp;
    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
    {
        expires 30d;
    }
    location ~ .*\.(js|css)?$
    {
        expires 1h;
    }
location / {
# 如果要在swoole里获取实际host          proxy_set_header Host $http_host;
            proxy_pass http://127.0.0.1:8991;
            proxy_http_version 1.1;
            proxy_set_header Connection "keep-alive";
    }
    access_log  /usr/local/etc/nginx/logs/zhttp.log;
}
```
    
       

## 



###目录结构

![目录结构](https://raw.githubusercontent.com/keaixiaou/base/master/WechatIMG1.jpeg)



##apps -  mvc框架的controllers和service

####			service 通常的调用服务层
####	config - 配置文件
####	library - 对应的全局函数,每个work进程启动的时候会加载这个方法

​		

## 路由

​	根据pathinfo访问对应得controller，如ip:port/home/index/index则会访问home目录下的Index的index方法；如果不指定pathinfo则访问home目录下的Index的index方法
####​2016-12-01 新增自定义路由

```
​	1.普通闭包函数；
 ​	2.指定model或者server里的method；
 ​	3.闭包里直接调用相关yield方法（尽量不要这么做，后期维护性差）
    只需要在route.php里配置自定义路由，就可以在请求中使用。
         'GET' => [
             '/testindex' => function(){return 111;},
             //1.普通闭包函数
         ],
         'POST' => [
 
             '/test/{id}' => function($id){
                 return $id;
             },
         ],
         'ANY' => [
             '/' => '\Home\Index\main',
             //2.指定model和method
             '/user/{name}/no/{id}' => function($id, $name){
                 $data = yield \ZPHP\Core\App::model('test')->getUserDetail($id, $name);
                 return ['data'=>$data];
             },
                 //3.有相关异步操作的闭包,App::model是获取全局容器里的model
             '/user/{id}' => function($id){
                 return \ZPHP\Core\App::model('test')->getUserById($id);
             }
      
         ]

```

##controller
####2016-12-01-在controller注入get、post、session、cookie等参数（因为整个框架是异步的，所以会导致一个work进程内同时存在多个请求，所以$_GET,$_POST,$_REQUEST,$_SESSION等以前用的全局变量下一个请求的值影响上一个还没处理完请求的值，造成数据混乱）
```
获取值:
	$this->input->get('id');
	//在controller里获取$_GET['id'];
	$this->input->get('id', true);
	//在controller里获取$_GET['id']被过滤过的值;
	$this->input->get();
	//在controller里获取整个$_GET
设置值：
	$this->input->get['id'] = 1;
	$this->input->session['user'] = $userinfo;
	$this->input->cookie['load'] = 1;
	
支持request、post、files、server、cookie等，使用一样

```
###session
如果需要用到session需要配置session，如下

```
'session'=> array(
        'enable' => true,
        'adapter' => 'Redis',
        'redis' => [
            'ip' => '127.0.0.1',
            'port' => 6379,
            'select' => 1,
            'password' => '123456',
            'asyn_max_count' => 10,
            'start_count' => 5,
        ],
        'name' => 'sses_',
        'session_name'=>'ZPHP_SID',
        'cache_expire' => 3600,
    ),

    'cookie'=> array(
        'enable' => true,
        'cache_expire' => 86400,
    ),
```
上面的配置enable表示是否启用，adapter为记录session的途径，现在支持Redis或者File，name为session前缀，session_name为当session不存在寻找对应cookie的key来建立相应的session。cache_expire为session缓存时间，单位为秒

##App全局服务容器：
```
model层：
		$sql = 'select * from admin_user where id=1';
        $data['sql'] = $sql;
        $data['info'] = yield table('admin_user')->where(['id'=>1])->find();
        return $data;
service层调用model相关方法：
		$data = yield App::model('test')->get(1);
		//test默认为model的类名，get为方法
		return $data;
		
controller层调用service层的服务:
		//使用1-封装在service层,controller层也得写yield
        $testservice = yield App::service('test')
        //test默认为service的类名
        $data = yield $testservice->test();
        //test为test里面的方法名
        return $data;

```


####2016-11-28增加service和model的全局注入和引用:
```
	如上new创建的class，建议改为App::service('test');
	$data = yield App::service('test')->test();
	//App::service('test')是引用全局容器里的相关组件，耦合度低，可维护性强
	同样的如果使用数据，model层，可以使用如下
	$data = yield App::model('test')->test();
	
	提示：
	App::model('test')里面的test可以是在provide里面对应配置，也可以是直接对应的类名
	示例provide配置文件:
	return [
    'service'=>[
        'test' => service\Test::class,
    ],
    'model'=>[
        'test' => model\Test::class,
    ],
    'controller' => [
        'index' => controllers\Home\Index::class,
        'user' => controllers\Home\User::class,
    ],

];
	
```

## 

###Cache-redis(已经是异步非阻塞)

只要在config目录下配置cache文件，即可在业务里调用缓存方法,如：
配置:

```
return [
    'redis'=>[
        'ip' => 'localhost',
        'port' => 6379,
        'select' => 0,
        'password' => '',
        'asyn_max_count' => 10,
    ]
];
```
####2016-01-02新增redis配置参数start_count表示服务启动时进程内会启动的连接数
```
	'ip' => '127.0.0.1',
    'port' => 6379,
    'select' => 0,
    'password' => '123456',
    'asyn_max_count' => 10,
    'start_count' => 10,
```
使用:

```
$data = yield Db::redis()->cache($key); 
//读取缓存
$res = yield Db::redis()->cache($key,$value);
//写缓存，$value需传string,如此调用是保存缓存永久
$res = yield Db::redis()->cache($key,$value, 3600);
//写缓存，缓存时间为3600妙

返回true或者false


```
####2016-11-28增加lpush和lpop等队列操作：
	$res = yield Db::redis()->lpush('task','1111');
	$res = yield Db::redis()->lpop('task','1111');



##数据库



##mysql(已经是异步非阻塞)

在config下配置database的配置文件，即可在业务中使用,你可以使用以下方法查询数据
配置:

```
return [ 
	'database'=>[
	    'master' => [
	        'host' => '127.0.0.1',
			'user' => 'keaixiaou',
			'password' => '123456',
			'database' => 'test',
			'asyn_max_count' => 5,
	        'start_count' => 1,
	    ],
	]
]
```
####2016-01-02新增mysql配置参数start_count表示服务启动时进程内会启动的连接数
```
	'host' => '127.0.0.1',
	'user' => 'keaixiaou',
	'password' => '123456',
	'database' => 'test',
	'asyn_max_count' => 5,
	'start_count' => 1,
```
使用：

```
比如是一张test表，里面有字段:id，content
$data = yield Db::table()->query('select* from test');
query方法查询出来的结果:
{
    "client_id": 1,
    "result": [
        {
            "id": "1",
            "content": "222333"
        }
    ],
    "affected_rows": 0,
    "insert_id": 0
}

如果query执行失败则里面的result为false

$userinfo = yield table('test')->where(['id'=>1])->find();
find 方法查询出来的结果：
 {
    "id": "1",
    "content": "222333"
}

$userinfo = yield table('test')->where(['id'=>1])->get();
get 方法查询出来的结果:
[
    {
        "id": "1",
        "content": "222333"
    }
]

$insertId = yield Db::table('test')->add(['content'=>'333']);
add 方法得到的结果是：2（主键ID)


$res = yield Db::table('test')->save(['content'=>'333']);
save方法得到的结果是:0（修改的行数）

以上add,get,find,save 如果执行失败则返回false

```


###http client（已经是异步非阻塞）

```
$httpClient = new HttpClientCoroutine();
$data = yield $httpClient->request('http://speak.test.com/');//get请求
$data = yield $httpClient->request('http://speak.test.com/',['a'=>1]);//post请求
```

###框架全部封装好.怎么样，这异步用起来是不是很简单^_^


###mongo（​2016-12-28 实现异步）
在config下配置mongo的配置文件，即可在业务中使用，如下
配置示例:

```
return [
    'mongo' => [
        'asyn_max_count' => 5,
        'host' => '127.0.0.1',
        'port' => 5000,
        'database' => 'test',
        ],
];
```


使用示例:

```
$data = Db::collection('stu_quest_score')->where(['like'=>100])->find();
```

![本机裸跑输出](https://raw.githubusercontent.com/keaixiaou/base/master/mongo.jpeg)


##ab测试（配置：MacBook Air 8G内存，双核，I5）
###本机裸跑输出

![本机裸跑输出](https://raw.githubusercontent.com/keaixiaou/base/master/%E8%A3%B8%E8%B7%91%E6%B5%8B%E8%AF%95.png)

###redis测试，2个work
![redis测试](https://raw.githubusercontent.com/keaixiaou/base/master/api%E6%B5%8B%E8%AF%95.png)







