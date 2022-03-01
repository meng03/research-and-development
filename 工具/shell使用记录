## 字符串处理
匹配特殊字符比较多的字符串时需要非常多转义符号
例如 Pods-Project.release.xcconfig 文件中的 "${PODS_ROOT}/xxxx"

````
path="\\\"\\\${PODS_ROOT}\/${xxxx}\\\""
sed -r -i '' -e 's/'$path'/''/g' Pods-Project.release.xcconfig
````
这里path打印出来就是 `\"\${PODS_ROOT}\/ZolozUtility\"`
这样在sed中使用的时候就会是 "${PODS_ROOT}/xxxx"


