# Set源码

## 1.1 HashSet

HashSet初始化容量大小与阿里巴巴Java开发手册约定计算公式一致(见编程规约第17条)

即initialCapacity = (需要存储的元素个数 / 负载因子) + 1

Math.max((int) (100/.75f) + 1, 16)

```java
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}
```
## 1.2 TreeSet

## 1.3 LinkedHashSet
