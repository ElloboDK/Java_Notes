# 关于Java解耦

[Blog](https://www.cnblogs.com/alvinscript/p/17036378.html)

**核心思想：** 减少 `if else` 和 `switch case`的使用，采用实现接口的方式。

## 应用Service Locator Pattern

服务定位模式`Service Locator Pattern`。

1. 让我们定义我们的服务定位器接口ParserFactory， 它有一个接受内容类型参数并返回Parser的方法。

    ```java
    public interface ParserFactory {
    Parser getParser(ContentType contentType);
    }
    ```

2. 我们配置`ServiceLocatorFactoryBean`使用`ParserFactory`作为服务定位器接口，`ParserFactory`这个接口不需要写实现类。
    ```java
    @Configuration
    public class ParserConfig {
        @Bean("parserFactory")
        public FactoryBean serviceLocatorFactoryBean() {
            ServiceLocatorFactoryBean factoryBean = new ServiceLocatorFactoryBean();
            // 设置服务定位接口   
            factoryBean.setServiceLocatorInterface(ParserFactory.class);
            return factoryBean;
        }
    }
    ```

3. 设置解析器Bean的名称为类型名称，方便服务定位

    ```java
    // 设置bean的名称和类型一致
    @Component("CSV")
    public class CSVParser implements Parser { .. }
    @Component("JSON")
    public class JSONParser implements Parser { .. }
    @Component("XML")
    public class XMLParser implements Parser { .. }
    ```
4. 修改枚举, 添加XML
    ```java
    public enum ContentType {
        JSON,
        CSV,
        XML
    }
    ```
5. 最后用客户端调用，直接根据类型调用对应的解析器，没有了switch case
    ```java
    @Service
    public class Client {
        private ParserFactory parserFactory;
        @Autowired
        public Client(ParserFactory parserFactory) {
            this.parserFactory = parserFactory;
        }
        public List getAll(ContentType contentType) {
            ..
            // 关键点，直接根据类型获取
            return parserFactory
                .getParser(contentType)  
                .parse(reader);
        }
        ..
    }
    ```
现在再加新的类型，我们只要扩展添加新的解析器就行，再也不用修改客户端了，满足开闭原则。

如果你觉得Bean的名称直接使用类型怪怪的，这边可以建议你按照下面的方式来。

```java
public enum ContentType {
    JSON(TypeConstants.JSON_PARSER),
    CSV(TypeConstants.CSV_PARSER),
    XML(TypeConstants.XML_PARSER);
    private final String parserName;
    ContentType(String parserName) {
        this.parserName = parserName;
    }

    @Override
    public String toString() {
        return this.parserName;
    }
    
    public interface TypeConstants {
        String CSV_PARSER = "csvParser";
        String JSON_PARSER = "jsonParser";
        String XML_PARSER = "xmlParser"; 
    }
}

@Component(TypeConstants.CSV_PARSER)
public class CSVParser implements Parser { .. }
@Component(TypeConstants.JSON_PARSER)
public class JSONParser implements Parser { .. }
@Component(TypeConstants.XML_PARSER)
public class XMLParser implements Parser { .. }
```

## 剖析Service Locator Pattern

服务定位器模式消除了客户端对具体实现的依赖。 以下引自 `Martin Fowler` 的文章总结了核心思想： “服务定位器背后的基本思想是拥有一个知道如何获取应用程序可能需要的所有服务的对象。因此，此应用程序的服务定位器将有一个在需要时返回“服务”的方法。”

![](E:\Java-study\Java_Notes\src\img\Service_Locator_Pattern.png)

`Spring` 的 `ServiceLocatorFactoryBean` 实现了 `FactoryBean` 接口，创建了 `Service Factory` 服务工厂 `Bean` 。

## 总结
我们通过使用服务定位器模式实现了一种扩展 `Spring` 控制反转的绝妙方法。它帮助我们解决了依赖注入未提供最佳解决方案的用例。也就是说，依赖注入仍然是首选，并且在大多数情况下不应使用服务定位器来替代依赖注入。
