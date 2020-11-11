属性别名AliasFor

```java
@Nullable
/**
 * 传入注解方法,该方法如果有AliasFor注解,则返回AliasDescriptor对象
 */
public static AliasDescriptor from(Method attribute) {
    AliasDescriptor descriptor = aliasDescriptorCache.get(attribute);
    if (descriptor != null) {
        return descriptor;
    }

    AliasFor aliasFor = attribute.getAnnotation(AliasFor.class);
    if (aliasFor == null) {
        return null;
    }

    descriptor = new AliasDescriptor(attribute, aliasFor);
    descriptor.validate();
    aliasDescriptorCache.put(attribute, descriptor);
    return descriptor;
}
```

