---
layout:     post
title:      how @Transient works in tk.mybatis.mapper
subtitle:   tk.mybatis中字段非持久化
date:       2018-07-02
author:     QA
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java
    - Transient
    - tk.mybatis
---

### @Transient
Java Doc中的描述是

    * This annotation specifies that the property or field is 
    * not persistent. It is used to annotate a property or field 
    * of an entity class, mapped superclass, or embeddable class. 
    *
    * <pre>
    *    Example:
    *    &#064;Entity
    *    public class Employee {
    *        &#064;Id int id;
    *        &#064;Transient User currentUser;
    *        ...
    *    }
    * </pre>
    *
    * @since Java Persistence 1.0

### 源码
在Debug的时候发现在 **BaseInsertProvider** 实现类:

    public String insert(MappedStatement ms) {
        Class<?> entityClass = getEntityClass(ms);
        ...
    }

insert方法调用了有 **getEntityClass** 方法:

    /**
     * 获取返回值类型 - 实体类型
     *
     * @param ms
     * @return
     */
    public Class<?> getEntityClass(MappedStatement ms) {
        String msId = ms.getId();
        if (entityClassMap.containsKey(msId)) {
            return entityClassMap.get(msId);
        } else {
            Class<?> mapperClass = getMapperClass(msId);
            Type[] types = mapperClass.getGenericInterfaces();
            for (Type type : types) {
                if (type instanceof ParameterizedType) {
                    ParameterizedType t = (ParameterizedType) type;
                    if (t.getRawType() == this.mapperClass || this.mapperClass.isAssignableFrom((Class<?>) t.getRawType())) {
                        Class<?> returnType = (Class<?>) t.getActualTypeArguments()[0];
                        //获取该类型后，第一次对该类型进行初始化
                        EntityHelper.initEntityNameMap(returnType, mapperHelper.getConfig()); //<-------关键在上面这句
                        entityClassMap.put(msId, returnType);
                        return returnType;
                    }
                }
            }
        }
        throw new MapperException("无法获取Mapper<T>泛型类型:" + msId);
    }

关键在EntityHelper.initEntityNameMap静态方法,初始化实体类.

    for (EntityField field : fields) {
        //如果启用了简单类型，就做简单类型校验，如果不是简单类型，直接跳过
        if(config.isUseSimpleType() && !SimpleTypeUtil.isSimpleType(field.getJavaType())){
            continue;
        }
        processField(entityTable, style, field);
    }

最后在处理单个Field的时候,有一个annotatio的判断

    /**
     * 处理一列
     *
     * @param entityTable
     * @param style
     * @param field
     */
    private static void processField(EntityTable entityTable, Style style, EntityField field) {
        //排除字段
        if (field.isAnnotationPresent(Transient.class)) {
            return;
        }
        ...
    }

这就是为什么Entity对象中添加@Transient可以生效的原因.

### 总结
@Transient生效的原因是在处理单个Field的时候将此Field过滤掉,这样就不会存在与EntityClass里面.