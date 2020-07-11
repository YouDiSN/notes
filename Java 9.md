#Java 9

### Jshell



### modules

```java
// define a module
module my.module {
  exports // 发布自己的service，如果是目录不包含子目录
  requires // 申明依赖，依赖的名称必须是module的名字，不是package名字
  requires transitive // A依赖B，如果B requires transitive C，那么A也依赖C。依赖的传递
  uses // 
  provides // 
}
```



###List.of, Sets.of, Map.ofEntries

```java
// Immutable
List.of(1, 2, 3, 4, 5);

Map.of("key1", "value1", "key2", "value2");

Set.of(1, 2, 3, 4, 5);
```

