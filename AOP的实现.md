# AOP的实现

> AOP 为 `Aspect Oriented Programming` 的缩写，意为：`面向切面编程`，通过动态代理等技术实现程序功能的统一维护的一种技术。AOP 是 OOP 的延续，也是 Hyperf 中的一个重要内容，是函数式编程的一种衍生范型。利用 AOP 可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。
>
> 用通俗的话来讲，就是在 Hyperf 里可以通过 `切面(Aspect)` 介入到任意类的任意方法的执行流程中去，从而改变或加强原方法的功能，这就是 AOP。 --- hyperf

AOP是一种模式，和语言无关。



## 设计思路

如果要我们自己设计一种类，可以介入到任意类的任意方法的执行流程中去，我们会怎么设计呢？

比如第三方类库，一般情况下我们自己肯定是不去修改的，那么怎么介入进去呢？

首先我们想到的是自己写一个类，继承要切入的类，然后重写自己想介入的方法，使用的时候调用自己的写的类和方法不就行了？是的，这个思路是正确的，但是如果我们要大批量的去做这个事情，比如我们想跟踪函数调用栈，请求调用链之类的问题，这个时候我们需要能够自动生成这些“自己写的类”。

那该怎么办才好？

有一种不错的方案就是在类，自动加载的时候，我们偷偷引用另外一个类文件，这个时候在使用者角度是无感知的，这个比上面一个方案(使用自己的类）要好，而且非常容易插拔。

什么意思？看下代码就知道了

```php
// CLassLoader.php
protected function locateFile(string $className): ?string
    {
        if (isset($this->proxies[$className]) && file_exists($this->proxies[$className])) {
            $file = $this->proxies[$className];
        } else {
            $file = $this->getComposerClassLoader()->findFile($className);
        }

        return is_string($file) ? $file : null;
    }
```

`CLassLoader`是hyperf框架自定义的类加载器，上面可以看到，加载某个类文件的时候，先判断下某个类的`proxies`文件是否存在，有的话就返回，否者的话用composer去查找。

这里引入了一个`proxies`代理类的概念出来。这个代理类是自动生成的，它就是我们想要的那个“自己写的类”

```php
class ProxyManager
{
    public function __construct(
        array $reflectionClassMap = [],
        array $composerLoaderClassMap = [],
        string $proxyDir = ''
    ) {
        $this->classMap = $this->mergeClassMap($reflectionClassMap, $composerLoaderClassMap);
        $this->proxyDir = $proxyDir;
        $this->filesystem = new Filesystem();
      	// 生成代理文件
        $this->proxies = $this->generateProxyFiles($this->initProxiesByReflectionClassMap(
            $this->classMap
        ));
    }
  
      protected function generateProxyFiles(array $proxies = []): array
    {
        ...
        return $proxyFiles;
    }

    protected function putProxyFile(Ast $ast, $className)
    {
        ...
        return $proxyFilePath;
    }
  
    protected function initProxiesByReflectionClassMap(array $reflectionClassMap = []): array
        {
            ...
            return $proxies;
        }
    
```



## Aspect切面收集

切面有2种形式收集，一种是通过配置文件的形式申明然后调用收集逻辑。另外一种是`注解`的形式申明和收集。

配置申明和注解类似，都是通过个配置文件和ConfigProvider返回的配置申明。

```php
// config/autoload/aspect.php
return [
];

// config/config.php
return [
];

// ConfigProvider.php
return [
            'aspects' => [
                InjectAspect::class,
            ],
           
        ];
```

下面是对配置申明的切面进行收集逻辑

```php
	// Scanner.php
	/**
     * Load aspects to AspectCollector by configuration files and ConfigProvider.
     */
    protected function loadAspects(int $lastCacheModified): void
    {
      	...
        AspectCollector::setAround($aspect, $classes, $annotations, $priority);
    }
```

使用注解收集的逻辑在`注解的实现`一文有介绍，这里不过多介绍，只列出代码。

```php
// Aspect.php

/**
 * @Annotation
 * @Target({"CLASS"})
 */
class Aspect extends AbstractAnnotation
{
    public $classes = [];
    public $annotations = [];
    public $priority;

    public function collectClass(string $className): void
    {
        parent::collectClass($className);
        $this->collect($className);
    }

    protected function collect(string $className)
    {
				...
        AspectCollector::setAround($className, $classes, $annotations, $priority);
    }
}
```

可以看到，Aspect使用AspectCollector来收集注解类和使用了注解类的类的关系。



## 再看proxies文件的生成

上面已经通过`AspectCollector`收集器收集了所有切面了，下面就是生成需要被介入的类的代理类。

```php
    protected function initProxiesByReflectionClassMap(array $reflectionClassMap = []): array
        {
            // According to the data of AspectCollector to parse all the classes that need proxy.
            $proxies = [];
            if (! $reflectionClassMap) {
                return $proxies;
            }
      
      			// 获取所有需要被介入的类
            $classesAspects = AspectCollector::get('classes', []);
            ...
            $proxies[$class][] = $aspect;

      			...
            $annotationsAspects = AspectCollector::get('annotations', []);
            ...        
            $proxies[$className][] = $aspect;
                               
            return $proxies;
        }
```



## 总结

1. 在配置文件或ConfigProvider里面显示申明切面。
2. 在类里面使用Aspect注解。
3. 先收集使用了`Aspect注解`的类。
4. 然后在收集显示申明的`Aspect`(这个是普通类，非注解)
5. 扫描所有类，如果在`AspectCollector`里面收集到的类和注解中就生成`Proxies`代理类。