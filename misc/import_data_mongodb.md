## mongodb数据批量导入

### 1 问题

现在有这样一种场景，业务中使用mongodb存储配置数据，在开发完成后，需要从json中解析数据然后导入到mongodb中，由于数据比较多，需要开发一个脚本执行导入操作。

### 2 解决办法

之前在mysql中如果有批量数据导入，通常的办法是，用python读取数据，然后拼接成sql，然后用mysql命令批量执行sql语句进行导入。

因此，解决方法一：生成插入语句，然后批量执行。

先看下mongodb中的插入语句的写法：

``` shell
db.collection.insertOne({
	"col1": "val1",
	"col2": true
})

db.collection.insert(
	{"col1": "val1", "col2": true},
	{"col1": "val2", "col2": false}
	)
```

无论是一次插入一条还是一次插入多条，插入的数据行就是一个完整的json字符串，因此，可以写一个脚本对原始数据文件进行处理，然后生成json数据，再拼接成插入语句。

这里用lua实现：

``` lua
local cjson = require("cjson")

local file = io.open("data.json", "r+")

io.input(file)

local data = io.read("*a")

io.close(file)

local data = cjson.decode(data)

for i=1,#data do
        local record = data[i]
        record.id = tostring(i)
        record.created_by = "admin"
        record.created_at = "new Date()"
        record.updated_by = "admin"
        record.updated_at = "new Date()"

        local res, _ = string.gsub("db.collection.insertOne(" .. cjson.encode(record) .. ")", "\"new Date%(%)\"", "new Date()")
        print(res)
end
```

先用cjson.decode()将数据解析成json对象，然后取出其中的数据，再对每行数据的字段进行填充，最后用cjson.encode()变成json字符串即可，这里唯一需要处理的是Date类型的字段，在设置时需要用{"created_at": new Date()}才行，因此，这里先转成json字符串，然后再进行替换。

于是就可以得到一个包含插入数据的文件result。

然后可以执行下面的语句将数据写入mongodb：

``` shell
cat result | mongo -u root -p xxx db_name --shell --authenticationDatabase admin
```

这种方式还是比较简单的，但是还是涉及到lua的环境搭建。

第二种方式依然跟mysql类型，mysql有mysqlimport和mysqlexport工具，相应的，mongodb中也有mongoimport和mongoexport工具。


``` shell
mongoexport -u root -p xxx --db db_name --collection collection_name --out data.json --authenticationDatabase admin
```

导出的数据就是json数据，然后就可以使用mongoimport进行导入：

``` shell
mongoimport -u root -p xxx --db db_name --collection collection_name data.json --authenticationDatabase admin
```

通过这种方式，可以方便的将数据导入导出，而且数据就是json数据，所以，也可以用lua脚本直接将数据变成跟表结构一致的json数据，然后用mongoimport进行导入，这就是第三种方式，但是，这种方式在处理date类型的字段时就只能自己生成一个时间，而无法使用new Date()让mongo在数据插入时自动设置时间。