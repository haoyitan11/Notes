# Jackson：GitHub上star数最多的JSON解析库

在当今的编程世界里，JSON 已经成为将信息从客户端传输到服务器端的首选协议，可以好不夸张的说，XML 就是那个被拍死在沙滩上的前浪。

很不幸的是，JDK 没有 JSON 库，不知道为什么不搞一下。Log4j 的时候，为了竞争，还推出了 java.util.logging，虽然最后也没多少人用。

Java 之所以牛逼，很大的功劳在于它的生态非常完备，JDK 没有 JSON 库，第三方类库有啊，还挺不错，比如说本篇的猪脚——Jackson，GitHub 上标星 6.1k，Spring Boot 的默认 JSON 解析器。

怎么证明这一点呢？

当我们通过 starter 新建一个 Spring Boot 的 Web 项目后，就可以在 Maven 的依赖项中看到 Jackson 的身影。

<img width="1548" height="1728" alt="image" src="https://github.com/user-attachments/assets/38085296-3da7-46ba-92fc-e0ffb78e6a9e" />

Jackson 有很多优点：
- 解析大文件的速度比较快；
- 运行时占用的内存比较少，性能更佳；
- API 很灵活，容易进行扩展和定制。

Jackson 的核心模块由三部分组成：
- jackson-core，核心包，提供基于“流模式”解析的相关 API，包括 JsonPaser 和 JsonGenerator。
- jackson-annotations，注解包，提供标准的注解功能；
- jackson-databind ，数据绑定包，提供基于“对象绑定”解析的相关 API （ ObjectMapper ） 和基于“树模型”解析的相关 API （JsonNode）。

## 01、引入 Jackson 依赖
要想使用 Jackson，需要在 pom.xml 文件中添加 Jackson 的依赖。

```java
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.10.1</version>
</dependency>
```

jackson-databind 依赖于 jackson-core 和 jackson-annotations，所以添加完 jackson-databind 之后，Maven 会自动将 jackson-core 和 jackson-annotations 引入到项目当中。

<img width="1944" height="618" alt="image" src="https://github.com/user-attachments/assets/d95b48d8-01b0-4737-bcfe-2374953808d2" />

Maven 之所以讨人喜欢的一点就在这，能偷偷摸摸地帮我们把该做的做了。

## 02、使用 ObjectMapper
Jackson 最常用的 API 就是基于”对象绑定” 的 ObjectMapper，它通过 writeValue 的系列方法将 Java 对象序列化为 JSON，并且可以存储成不同的格式。

- writeValueAsString(Object value) 方法，将对象存储成字符串
- writeValueAsBytes(Object value) 方法，将对象存储成字节数组
- writeValue(File resultFile, Object value) 方法，将对象存储成文件

来看一下存储成字符串的代码示例：
```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

/**
 * 微信搜索「沉默王二」，回复 Java
 *
 * @author 沉默王二
 * @date 2020/11/26
 */
public class Demo {
    public static void main(String[] args) throws JsonProcessingException {
        Writer wanger = new Writer("沉默王二", 18);
        ObjectMapper mapper = new ObjectMapper();
        String jsonString = mapper.writerWithDefaultPrettyPrinter()
                .writeValueAsString(wanger);
        System.out.println(jsonString);
    }
}

class Writer {
    private String name;
    private int age;

    public Writer(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

程序输出结果如下所示：
```java
{
  "name" : "沉默王二",
  "age" : 18
}
```

不是所有的字段都支持序列化和反序列化，需要符合以下规则：

- 如果字段的修饰符是 public，则该字段可序列化和反序列化（不是标准写法）。
- 如果字段的修饰符不是 public，但是它的 getter 方法和 setter 方法是 public，则该字段可序列化和反序列化。getter 方法用于序列化，setter 方法用于反序列化。
- 如果字段只有 public 的 setter 方法，而无 public 的 getter 方 法，则该字段只能用于反序列化。

如果想更改默认的序列化和反序列化规则，需要调用 ObjectMapper 的 setVisibility() 方法。否则将会抛出 InvalidDefinitionException 异常。

ObjectMapper 通过 readValue 的系列方法从不同的数据源将 JSON 反序列化为 Java 对象。

- readValue(String content, Class<T> valueType) 方法，将字符串反序列化为 Java 对象
- readValue(byte[] src, Class<T> valueType) 方法，将字节数组反序列化为 Java 对象
- readValue(File src, Class<T> valueType) 方法，将文件反序列化为 Java 对象

来看一下将字符串反序列化为 Java 对象的代码示例：
```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

/**
 * 微信搜索「沉默王二」，回复 Java
 *
 * @author 沉默王二
 * @date 2020/11/26
 */
public class Demo {
    public static void main(String[] args) throws JsonProcessingException {
        ObjectMapper mapper = new ObjectMapper();
        String jsonString = "{\n" +
                "  \"name\" : \"沉默王二\",\n" +
                "  \"age\" : 18\n" +
                "}";
        Writer deserializedWriter = mapper.readValue(jsonString, Writer.class);
        System.out.println(deserializedWriter);
    }
}

class Writer{
    private String name;
    private int age;

    // getter/setter

    @Override
    public String toString() {
        return "Writer{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

程序输出结果如下所示：
```java
Writer{name='沉默王二', age=18}
```

PS：如果反序列化的对象有带参的构造方法，它必须有一个空的默认构造方法，否则将会抛出 InvalidDefinitionException 一行。
```java
Exception in thread "main" com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct instance of `com.itwanger.jackson.Writer` (no Creators, like default construct, exist): cannot deserialize from Object value (no delegate- or property-based Creator)
 at [Source: (String)"{
  "name" : "沉默王二",
  "age" : 18
}"; line: 2, column: 3]
	at com.fasterxml.jackson.databind.exc.InvalidDefinitionException.from(InvalidDefinitionException.java:67)
	at com.fasterxml.jackson.databind.DeserializationContext.reportBadDefinition(DeserializationContext.java:1589)
	at com.fasterxml.jackson.databind.DeserializationContext.handleMissingInstantiator(DeserializationContext.java:1055)
	at com.fasterxml.jackson.databind.deser.BeanDeserializerBase.deserializeFromObjectUsingNonDefault(BeanDeserializerBase.java:1297)
	at com.fasterxml.jackson.databind.deser.BeanDeserializer.deserializeFromObject(BeanDeserializer.java:326)
	at com.fasterxml.jackson.databind.deser.BeanDeserializer.deserialize(BeanDeserializer.java:159)
	at com.fasterxml.jackson.databind.ObjectMapper._readMapAndClose(ObjectMapper.java:4202)
	at com.fasterxml.jackson.databind.ObjectMapper.readValue(ObjectMapper.java:3205)
	at com.fasterxml.jackson.databind.ObjectMapper.readValue(ObjectMapper.java:3173)
	at com.itwanger.jackson.Demo.main(Demo.java:19)
```

Jackson 最常用的 API 就是基于”对象绑定” 的 ObjectMapper，

ObjectMapper 也可以将 JSON 解析为基于“树模型”的 JsonNode 对象，来看下面的示例。
```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

/**
 * 微信搜索「沉默王二」，回复 Java
 *
 * @author 沉默王二
 * @date 2020/11/26
 */
public class JsonNodeDemo {
    public static void main(String[] args) throws JsonProcessingException {
        ObjectMapper mapper = new ObjectMapper();
        String json = "{ \"name\" : \"沉默王二\", \"age\" : 18 }";
        JsonNode jsonNode = mapper.readTree(json);
        String name = jsonNode.get("name").asText();
        System.out.println(name); // 沉默王二
    }
}
```

借助 TypeReference 可以将 JSON 字符串数组转成泛型 List，来看下面的示例：

```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.List;

/**
 * 微信搜索「沉默王二」，回复 Java
 *
 * @author 沉默王二
 * @date 2020/11/26
 */
public class TypeReferenceDemo {
    public static void main(String[] args) throws JsonProcessingException {
        ObjectMapper mapper = new ObjectMapper();
        String json = "[{ \"name\" : \"沉默王三\", \"age\" : 18 }, { \"name\" : \"沉默王二\", \"age\" : 19 }]";
        List<Author> listAuthor = mapper.readValue(json, new TypeReference<List<Author>>(){});
        System.out.println(listAuthor);
    }
}
class Author{
    private String name;
    private int age;

    // getter/setter

    // toString
}
```

## 03、更高级的配置
Jackson 之所以牛掰的一个很重要的因素是可以实现高度灵活的自定义配置。

在实际的应用场景中，JSON 中常常会有一些 Java 对象中没有的字段，这时候，如果直接解析的话，会抛出 UnrecognizedPropertyException 异常。

下面是一串 JSON 字符串：

```java
String jsonString = "{\n" +
                "  \"name\" : \"沉默王二\",\n" +
                "  \"age\" : 18\n" +
                "  \"sex\" : \"男\",\n" +
                "}";
```

但 Java 对象 Writer 中没有定义 sex 字段：

```java
class Writer{
    private String name;
    private int age;

    // getter/setter
}
```

我们来尝试解析一下：
```java
ObjectMapper mapper = new ObjectMapper();
Writer deserializedWriter = mapper.readValue(jsonString, Writer.class);
```

不出意外，抛出异常了，sex 无法识别。
```java
Exception in thread "main" com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException: Unrecognized field "sex" (class com.itwanger.jackson.Writer), not marked as ignorable (2 known properties: "name", "age"])
 at [Source: (String)"{
  "name" : "沉默王二",
  "age" : 18,
  "sex" : "男"
}"; line: 4, column: 12] (through reference chain: com.itwanger.jackson.Writer["sex"])
```

怎么办呢？可以通过 configure() 方法忽略掉这些“无法识别”的字段。
```java
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

除此之外，还有其他一些有用的配置信息，来了解一下：
```java
// 在序列化时忽略值为 null 的属性
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
// 忽略值为默认值的属性
mapper.setDefaultPropertyInclusion(JsonInclude.Include.NON_DEFAULT);
```

