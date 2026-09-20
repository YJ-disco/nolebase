
# JVM、JRE 和 JDK 介绍

![](http://yj-dis.top/20250210100231.png)

## 1. JVM（Java Virtual Machine）

### 定义

JVM（Java Virtual Machine）即 Java 虚拟机，是 Java 程序运行的核心环境。它是一个抽象的计算机，负责执行 Java 字节码，将其翻译为特定操作系统的机器码。

### 功能

- **执行字节码**：JVM 负责执行由 Java 编译器生成的字节码文件（.class 文件），使其能够在不同的操作系统上运行。
- **内存管理**：JVM 提供了垃圾回收机制（Garbage Collection, GC），自动管理内存分配和释放，避免内存泄漏。
- **平台无关性**：JVM 是 Java 跨平台特性的关键，通过在不同平台上提供特定的 JVM 实现，使得 Java 程序可以在任何安装了 JVM 的设备上运行。
- **安全性**：JVM 提供了安全机制，防止恶意代码对系统资源的非法访问。

## 2. JRE（Java Runtime Environment）

### 定义

JRE（Java Runtime Environment）即 Java 运行时环境，是运行已编译 Java 程序所需的所有内容的集合。

### 组成部分

- **JVM**：JRE 包含 JVM，负责执行 Java 字节码。
- **类库（核心类库）**：JRE 包含了一系列标准的 Java 类库，为 Java 应用提供了常用功能，如基础数据结构、I/O 操作、网络通信、多线程和并发、图形界面等。
- **Java 类加载器**：负责将 Java 类文件加载到内存中。
- **垃圾回收机制**：管理应用程序的内存分配和释放。

### 用途

- **运行 Java 程序**：JRE 提供了运行 Java 应用程序所需的环境，普通用户只需安装 JRE 即可运行 Java 应用。
- **跨平台支持**：通过 JVM，JRE 保证了 Java 程序在不同平台上的一致运行行为。

## 3. JDK（Java Development Kit）

### 定义

JDK（Java Development Kit）即 Java 开发工具包，是 Java 语言的软件开发工具包，包含了用于支持 Java 程序员开发和编译 Java 应用程序的标准开发工具集合。

### 组成部分

- **JVM**：JDK 包含 JVM，用于运行编译后的 Java 字节码。
- **JRE**：JDK 包含完整的 JRE，用于运行 Java 应用程序。
- **开发工具**：JDK 提供了许多开发工具，帮助开发人员完成从源代码编写到程序运行的全过程，如 `javac`（Java 编译器）、`java`（JVM 的入口）、`javadoc`（文档生成工具）、`jdb`（Java 调试器）、`jar`（打包工具）、`javap`（Java 反汇编工具）等。
- **类库**：JDK 包含了大量的标准类库，为 Java 程序提供了各种基础功能。

### 用途

- **开发 Java 程序**：JDK 是开发人员必备的工具箱，提供了编译、运行 Java 程序所需的一切元素。
- **辅助开发和维护**：JDK 提供了许多辅助开发和维护 Java 应用程序的实用工具，如监控和诊断工具（`jps`、`jinfo`、`jstat`、`jmap`、`jhat`、`jstack`）。



