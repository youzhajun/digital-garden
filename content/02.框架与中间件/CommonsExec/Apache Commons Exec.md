---
title: Apache Commons Exec
draft: false
tags:
  - java
  - 中间件
date: 2024-11-10
---
# 认识一下

## 是什么？
Apache Commons Exec 是一个用于**安全、可靠地执行外部进程**的 Java 库，属于 Apache Commons 工具集。它解决了 Java 原生 `Runtime.exec()` 和 `ProcessBuilder` 在处理外部进程时的常见痛点（如死锁、平台兼容性、超时控制等），提供了更健壮和易用的 API。

## 能做什么？

- **安全执行外部命令：** 避免因未处理 I/O 流导致的进程阻塞（死锁问题）。
- **跨平台支持**
- **超时控制**
- **异步执行**
- **流管理**
- **结果处理**
- **环境变量控制**

## 怎么用？

1. 引入依赖
	```xml
		<dependency>
		    <groupId>org.apache.commons</groupId>
		    <artifactId>commons-exec</artifactId>
		    <version>1.4.0</version> <!-- 检查最新版本 -->
		</dependency>
	```

2. 编写 demo
	```java
		import org.apache.commons.exec.CommandLine;
		import org.apache.commons.exec.DefaultExecutor;
		import org.apache.commons.exec.PumpStreamHandler;
		import java.io.ByteArrayOutputStream;
		import java.io.IOException;
		
		public class SyncExecExample {
		    public static void main(String[] args) throws IOException {
		        // 1. 构建命令（此处以 ping 为例）
		        CommandLine cmd = new CommandLine("ping");
		        cmd.addArgument("-n"); // Windows 参数
		        cmd.addArgument("2");
		        cmd.addArgument("www.google.com");
		
		        // 2. 配置执行器
		        DefaultExecutor executor = new DefaultExecutor();
		        executor.setExitValue(0); // 设置成功退出码（0 表示正常）
		
		        // 3. 捕获输出流
		        ByteArrayOutputStream output = new ByteArrayOutputStream();
		        PumpStreamHandler streamHandler = new PumpStreamHandler(output);
		        executor.setStreamHandler(streamHandler);
		
		        // 4. 执行命令（同步阻塞）
		        try {
		            int exitCode = executor.execute(cmd);
		            System.out.println("Exit Code: " + exitCode);
		            System.out.println("Output:\n" + output.toString());
		        } catch (IOException e) {
		            e.printStackTrace();
		        }
		    }
		}		
	```


# 核心类
| 类名                     | 作用                           |
| ---------------------- | ---------------------------- |
| `CommandLine`          | 封装系统命令及其参数                   |
| `DefaultExecutor`      | 命令执行器，执行命令并管理执行过程            |
| `ExecuteWatchdog`      | 监控命令执行时间（用于设置超时）             |
| `PumpStreamHandler`    | 管理标准输入、输出、错误流，防止阻塞           |
| `ExecuteResultHandler` | 处理异步执行结果（回调）                 |
| `Executor` 接口          | `DefaultExecutor` 的接口，支持定制实现 |
| `EnvironmentUtils`     | 创建或修改环境变量映射                  |

# 最佳实践

> 同步调用

```java
import org.apache.commons.exec.*;

public class CommandExecutor {

    public static void main(String[] args) throws Exception {
        // 1. 构建命令
        CommandLine cmd = new CommandLine("ping");
        cmd.addArgument("www.example.com");

        // 2. 设置输出流捕获
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        ByteArrayOutputStream errorStream = new ByteArrayOutputStream();
        PumpStreamHandler streamHandler = new PumpStreamHandler(outputStream, errorStream);

        // 3. 创建执行器
        DefaultExecutor executor = new DefaultExecutor();
        executor.setStreamHandler(streamHandler);
        
        // 4. 设置超时（例如 10 秒）
        ExecuteWatchdog watchdog = new ExecuteWatchdog(10_000); // 毫秒
        executor.setWatchdog(watchdog);

        // 5. 执行命令
        int exitCode = executor.execute(cmd);

        // 6. 获取结果,不同操作系统编码不同，win平台的输出是GBK
        String output = outputStream.toString("UTF-8");
        String error = errorStream.toString("UTF-8");

        // 7. 输出日志或做业务处理
        System.out.println("Exit Code: " + exitCode);
        System.out.println("Output:\n" + output);
        System.out.println("Error:\n" + error);
    }
}

```

> 异步调用（适用于耗时较长的操作）

```java
import org.apache.commons.exec.*;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class AsyncFfmpegExecutor {

    public void runFfmpegAsync() throws IOException {
        // 1. 构造命令（可替换成你自己的 ffmpeg 命令）
        CommandLine cmd = CommandLine.parse("ffmpeg -i input.mp4 -c:v libx264 output.mp4");

        // 2. 设置输出流（可选）
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        ByteArrayOutputStream errorStream = new ByteArrayOutputStream();
        PumpStreamHandler streamHandler = new PumpStreamHandler(outputStream, errorStream);

        // 3. 构建执行器
        DefaultExecutor executor = new DefaultExecutor();
        executor.setStreamHandler(streamHandler);

        // 4. 设置超时 watchdog（例如 5 分钟）
        long timeoutMs = 5 * 60 * 1000;
        ExecuteWatchdog watchdog = new ExecuteWatchdog(timeoutMs);
        executor.setWatchdog(watchdog);

        // 5. 设置期望的退出值（非 0 也可以继续处理）
        executor.setExitValues(null); // 或 new int[] {0}

        // 6. 异步处理回调
        ExecuteResultHandler resultHandler = new ExecuteResultHandler() {
            @Override
            public void onProcessComplete(int exitValue) {
                String output = outputStream.toString(StandardCharsets.UTF_8);
                System.out.println("[SUCCESS] ffmpeg exited with code: " + exitValue);
                System.out.println("Output:\n" + output);
                // ✅ 此处可触发后续业务流程，比如推送通知、更新数据库等
            }

            @Override
            public void onProcessFailed(ExecuteException e) {
                String err = errorStream.toString(StandardCharsets.UTF_8);
                System.err.println("[FAILURE] ffmpeg execution failed: " + e.getMessage());
                System.err.println("Error Output:\n" + err);

                if (watchdog.killedProcess()) {
                    System.err.println("⚠️ Process was killed due to timeout.");
                }

                // ✅ 异常处理或告警
            }
        };

        // 7. 启动异步执行
        executor.execute(cmd, resultHandler);

        // ✅ 主线程可继续运行其他逻辑，不阻塞
        System.out.println("ffmpeg 命令已异步启动...");
    }
}

```