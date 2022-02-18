# Golang字符串后添加字符

golang中并没有真正的字符(char)类型，go中的char其实是int32的别称

所以在实际操作中会发现

```go
var s string
var c char //报错，找不到char类型

s = apend(s,'a') //Cannot use 's' (type string) as the type []Type
s = apend(s,c)
```

