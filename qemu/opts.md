## opts 常用方法

打印 dict 对象：

```c
#include "qobject/qjson.h"

GString *json;
json = qobject_to_json_pretty(data, true);
printf("%s\n", json->str);
g_string_free(json, true);
```

