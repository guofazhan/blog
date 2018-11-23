title: zuul之动态过滤器Filter类文件管理加载
author: Mr.G
date: 2018-11-23 10:28:10
tags:
---
#### 目录
 - [zuul之请求处理流程](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B/)
 -  [zuul之动态过滤器Filter类文件管理加载](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E5%8A%A8%E6%80%81%E8%BF%87%E6%BB%A4%E5%99%A8Filter%E7%B1%BB%E6%96%87%E4%BB%B6%E7%AE%A1%E7%90%86%E5%8A%A0%E8%BD%BD/)
 -  [zuul之过滤器Filter管理](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E8%BF%87%E6%BB%A4%E5%99%A8Filter%E7%AE%A1%E7%90%86/)

#### 概述 
>zuul 支持过滤器动态修改动态加载功能，目前支持动态Filter由Groovy编写。动态管理Groovy编写的Filter类文件变更以及动态编译等功能。
<!-- more -->
#### 实现原理
> 实现原理是监控存放Filter文件的目录，定期扫描这些目录，如果发现有新Filter源码文件或者Filter源码文件有改动，则对文件进行编译加载

#### Filter类文件动态管理
> 通过原理知道zuul通过定期扫描Filter文件存放的目录来校验是否有新的文件或有改动的文件，对这些文件进行加载，源码是通过FilterFileManager实现此功能的。FilterFileManager管理目录轮询的变化和新的Groovy过滤器。轮询间隔和目录在类的初始化中指定，并且轮询器将检查,更改和添加。
---
> FilterFileManager 功能如下:
> - 开启一个线程，开始轮询(startPoller)
> - 每次轮询，处理目录内的所有*.groovy文件，即调用FilterLoader.getInstance().putFilter(file)


- 轮询部分
```java
   //开启轮询线程
    void startPoller() {
       //创建轮询线程
        poller = new Thread("GroovyFilterFileManagerPoller") {
            public void run() {
                while (bRunning) {
                    try {
                       //定时休眠 sleep(pollingIntervalSeconds * 1000);
                       //扫描目录监听目录变化
                        manageFiles();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        poller.setDaemon(true);
        //开启轮询线程
        poller.start();
    }
```
> -  startPoller 开启轮询线程定时调用manageFiles方法执行扫描目录，监听目录变化
> - startPoller方法在FilterFileManager初始化时（init）调用一次

- 目录扫描以及检测

```java
    //目录扫描检测主入口方法
    void manageFiles() throws Exception, IllegalAccessException, InstantiationException {
       // 1. 通过getFiles方法获取到目录的所有Filter文件
        List<File> aFiles = getFiles();
        //2. 通过processGroovyFiles方法处理所有Filter文件
        processGroovyFiles(aFiles);
    }
    
    //加载所有动态Filter文件
    List<File> getFiles() {
        List<File> list = new ArrayList<File>();
        //遍历配置的动态Filter文件存放目录，加载目录文件
        for (String sDirectory : aDirectories) {
            if (sDirectory != null) {
                File directory = getDirectory(sDirectory);
                File[] aFiles = directory.listFiles(FILENAME_FILTER);
                if (aFiles != null) {
                    list.addAll(Arrays.asList(aFiles));
                }
            }
        }
        return list;
    }
    
    //处理所有动态Filter文件
    void processGroovyFiles(List<File> aFiles) throws Exception, InstantiationException, IllegalAccessException {
        //遍历动态Filter列表
        for (File file : aFiles) {
            //通过FilterLoader类来加载处理Filter
            FilterLoader.getInstance().putFilter(file);
        }
    }

```
> manageFiles方法完成两件事
> 1. 调用getFiles方法加载所有预制目录中的动态Filter文件
> 2. 调用processGroovyFiles方法遍历filter列表通过FilterLoader类处理加载Filter

#### Filter类文件动态编译
> zuul 动态加载filter文件，并通过编译器将文件编译成class
> 目前zuul通过定义DynamicCodeCompiler接口以及Groovy编译的实现类GroovyCompiler来完成Groovy编写的filter的动态编译

- DynamicCodeCompiler

```java
public interface DynamicCodeCompiler {
    /**
     * 通过源码字符串以及filter 名称编译class
     *
     */
    Class compile(String sCode, String sName) throws Exception;
    /**
     *
     * 通过源码文件编译class
     */
    Class compile(File file) throws Exception;
}
```
> DynamicCodeCompiler 动态代码编译器接口
> 定义两个方法根据不同的参数返回编译好的class

- GroovyCompiler

```java
public class GroovyCompiler implements DynamicCodeCompiler {

    private static final Logger LOG = LoggerFactory.getLogger(GroovyCompiler.class);

    /**
     * Compiles Groovy code and returns the Class of the compiles code.
     *
     * @param sCode
     * @param sName
     * @return
     */
    @Override
    public Class compile(String sCode, String sName) {
        GroovyClassLoader loader = getGroovyClassLoader();
        LOG.warn("Compiling filter: " + sName);
        Class groovyClass = loader.parseClass(sCode, sName);
        return groovyClass;
    }

    /**
     * 获取GroovyClassLoader
     * @return a new GroovyClassLoader
     */
    GroovyClassLoader getGroovyClassLoader() {
        return new GroovyClassLoader();
    }

    /**
     * 编译groovy class从文件中
     * Compiles groovy class from a file
     *
     * @param file
     * @return
     * @throws java.io.IOException
     */
    @Override
    public Class compile(File file) throws IOException {
        GroovyClassLoader loader = getGroovyClassLoader();
        Class groovyClass = loader.parseClass(file);
        return groovyClass;
    }
}
```
> GroovyCompiler 为groovy编写的Filter的动态编译实现类。

---

> zuul 默认支持groovy编写的Filter，这给在不停服的情况下动态变更业务规则提供了可靠的保障