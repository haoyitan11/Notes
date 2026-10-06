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

