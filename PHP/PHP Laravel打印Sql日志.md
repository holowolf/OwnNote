## Laravel 打印日志

方法一:

```php
$sql = DB::table('my_table')->select()->tosql();
```
`tosql 只支持select语句`

方法二:

```php
DB::connection()->enableQueryLog();
DB::table('my_table')->insert($data);
$logs = DB::getQueryLog();
dd($logs);
```
`getQueryLog 只支持select语句`

方法三:

```php
// 在需要打印SQL的语句前添加监听事件。
DB::listen(function($query) {
    $bindings = $query->bindings;
    $sql = $query->sql;
    foreach ($bindings as $replace){
        $value = is_numeric($replace) ? $replace : "'".$replace."'";
        $sql = preg_replace('/\?/', $value, $sql, 1);
    }
    dd($sql);
});
// 要打印SQL的语句
$res = DB::table('my_table')->insert($data);
```

`此方法支持 全部insert update delete select`