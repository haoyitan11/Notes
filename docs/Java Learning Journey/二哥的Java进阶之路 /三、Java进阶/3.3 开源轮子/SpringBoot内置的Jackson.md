# Jackson：GitHub上star数最多的JSON解析库 (Jackson: The most starred JSON parsing library on GitHub.)

在当今的编程世界里，JSON 已经成为将信息从客户端传输到服务器端的首选协议，可以好不夸张的说，XML 就是那个被拍死在沙滩上的前浪。

很不幸的是，JDK 没有 JSON 库，不知道为什么不搞一下。Log4j 的时候，为了竞争，还推出了 java.util.logging，虽然最后也没多少人用。

Java 之所以牛逼，很大的功劳在于它的生态非常完备，JDK 没有 JSON 库，第三方类库有啊，还挺不错，比如说本篇的猪脚——Jackson，GitHub 上标星 6.1k，Spring Boot 的默认 JSON 解析器。

怎么证明这一点呢？

当我们通过 starter 新建一个 Spring Boot 的 Web 项目后，就可以在 Maven 的依赖项中看到 Jackson 的身影。

In today's programming landscape, JSON has become the go-to protocol for transmitting data from client to server; it is no exaggeration to say that XML has been left behind—like the proverbial "wave on the beach" superseded by the next one.

Unfortunately, the JDK lacks a built-in JSON library—it is unclear why this was never addressed. Back in the days of Log4j, a competing `java.util.logging` package was introduced, though it ultimately saw little adoption.

A major reason for Java's greatness is its comprehensive ecosystem; while the JDK may lack a JSON library, excellent third-party alternatives exist. A prime example—and the star of this article—is Jackson. It boasts 6.1k stars on GitHub and serves as the default JSON parser for Spring Boot.

How can we verify this?

When you create a new Spring Boot Web project using a starter, you will find Jackson included among the Maven dependencies.
