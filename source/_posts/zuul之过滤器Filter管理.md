title: zuul之过滤器Filter管理
author: Mr.G
date: 2018-11-23 10:30:40
tags:
---
#### 目录
 - [zuul之请求处理流程](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B/)
 -  [zuul之动态过滤器Filter类文件管理加载](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E5%8A%A8%E6%80%81%E8%BF%87%E6%BB%A4%E5%99%A8Filter%E7%B1%BB%E6%96%87%E4%BB%B6%E7%AE%A1%E7%90%86%E5%8A%A0%E8%BD%BD/)
 -  [zuul之过滤器Filter管理](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E8%BF%87%E6%BB%A4%E5%99%A8Filter%E7%AE%A1%E7%90%86/)

#### 概述 
> zuul通过FilterLoader类负责filter的管理，是ZUUL的核心类之一。它编译、从文件加载，并检查源代码是否更改。它还包含按filterType的ZuulFilters。此类再如下两处使用：
> 1. [zuul之请求处理流程](https://www.jianshu.com/p/f892839eb24d):FilterProcessor 通过FilterLoader加载给定类型的过滤器列表
> 2. [zuul之动态过滤器Filter类文件管理加载](https://www.jianshu.com/p/a9ce9349828e):调用processGroovyFiles方法遍历filter列表通过FilterLoader类处理加载Filter
<!-- more -->
#### FilterLoader

 - 根据过滤器类型获取过滤器列表
 > FilterLoader 中缓存了所有过滤器的集合，可以根据过滤器的类型获取对应的过滤器列表

```java
public List<ZuulFilter> getFiltersByType(String filterType) {

       //在缓存中获取当前类型的过滤器列表
        List<ZuulFilter> list = hashFiltersByType.get(filterType);
        if (list != null) return list;

        list = new ArrayList<ZuulFilter>();

       //通过filterRegistry获取所有过滤器
        Collection<ZuulFilter> filters = filterRegistry.getAllFilters();
        //遍历筛选过滤器获取给定的列表
        for (Iterator<ZuulFilter> iterator = filters.iterator(); iterator.hasNext(); ) {
            ZuulFilter filter = iterator.next();
            if (filter.filterType().equals(filterType)) {
                list.add(filter);
            }
        }
        //排序过滤器列表
        Collections.sort(list); // sort by priority

        //添加到缓存中
        hashFiltersByType.putIfAbsent(filterType, list);
        return list;
    }

```
> getFiltersByType
> - 在缓存中查找给定的类型列表
> - 当前缓存为空时，通过全部过滤器列表缓存遍历组装给定类型的列表并缓存返回

- 通过过滤器源码加载编译过滤器并缓存


```java
    public boolean putFilter(File file) throws Exception {
       // 获取类文件的全名
        String sName = file.getAbsolutePath() + file.getName();
        //校验类文件是否修改过，根据类文件的最后修改时间判断
        if (filterClassLastModified.get(sName) != null && (file.lastModified() != filterClassLastModified.get(sName))) {
        // 类文件修改过时，删除全局缓存中当前过滤器的信息
            LOG.debug("reloading filter " + sName);
            filterRegistry.remove(sName);
        }
        //在全局缓存中判断是否存在当前过滤器
        ZuulFilter filter = filterRegistry.get(sName);
        if (filter == null) {
           //缓存中无当前要加载的过滤器时，编译当前类文件
            Class clazz = COMPILER.compile(file);
            if (!Modifier.isAbstract(clazz.getModifiers())) {
               // 反射创建通过class创建过滤器对象
                filter = (ZuulFilter) FILTER_FACTORY.newInstance(clazz);
                //缓存当前过滤器对象
                List<ZuulFilter> list = hashFiltersByType.get(filter.filterType());
                if (list != null) {
                    hashFiltersByType.remove(filter.filterType()); //rebuild this list
                }
               
               //全局缓存当前过滤器对象 filterRegistry.put(file.getAbsolutePath() + file.getName(), filter);
                filterClassLastModified.put(sName, file.lastModified());
                return true;
            }
        }

        return false;
    }
```
> putFilter
> 校验给定的类文件是否变更过，变更过，删除旧缓存信息，编译新源文件并反射创建对象缓存到系统的全局缓存中

---
> FilterLoader主要完成了上述两个功能，一个为请求进来获取过滤器列表服务，一个为动态加载变更过滤器服务。至此 zuul的核心功能已经通过源码学习分析完毕