#### Gradle

###### Gradle命令

| 任务                 | 行为                                                         |
| :------------------- | :----------------------------------------------------------- |
| `gradle classes`     | 仅编译主代码 + 处理主资源文件，不涉及测试代码或打包。        |
| `gradle testClasses` | 编译测试代码（`src/test`）并处理测试资源，依赖 `classes`。   |
| `gradle build`       | 完整构建：编译主代码、测试代码、运行测试、打包（生成 JAR 等）。 |
| `gradle clean`       | 删除 `build` 目录，清理所有构建产物。                        |
| `gradle buildJar`    | 将项目编译后的代码、资源文件以及所有依赖的第三方库打包成一个独立的 JAR 文件。 |
| `gradle jar`         | 由 Java 插件提供，生成普通的 JAR（仅包含项目代码，不包含依赖）。 |
| `gradle compile`     | 编译源代码由.java到.class、处理资源文件、生成中间产物（没有jar） |
| `gradle assemble`    | 将编译后的代码、资源文件和依赖打包成可分发的格式（如 JAR、WAR、ZIP 等）。 |
| `gradle bufGenerate` | 生成代码文件，通常包含 gRPC 服务存根、Protobuf 消息类等      |