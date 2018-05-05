---
title: 《分布式服务框架：原理与实践》：RPC demo
date: 2017-07-20 19:13:32
tags: RPC
categories: [RPC, 架构，读书笔记]
---

下面通过 Java 原生的序列化、Socket 通信、动态代理和反射机制，实现最简单 RPC 框架。它由三部分组成：

1.  服务提供者：运行在服务端，负责提供服务接口定义和服务实现类 
2.  服务发布者：运行在 RPC 服务端，负责将本地服务发布成远程服务，供其他消费者调用 
3.  本地服务代理：运行在 RPC 客户端，通过代理调用远程服务提供者，然后将结果进行封装返回给本地消费者

 <!--more-->

代码如下：

- `EchoService`

  ```Java
  public interface EchoService {
      String echo(String ping);
  }
  ```
- `EchoServiceImpl`

  ```Java
  public class EchoServiceImpl implements EchoService {
      @Override
      public String echo(String ping) {
          return ping + " --> I am ok.";
      }
  }
  ```

- `RpcExporter`

  ```java
  public class RpcExporter {

    private static Executor executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    public static void exporter(String hostName, int port) throws Exception {
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.bind(new InetSocketAddress(hostName, port));
        try {
            while (true) {
                executor.execute(new ExporterTask(serverSocket.accept()));
            }
        } finally {
            serverSocket.close();
        }
    }

    private static class ExporterTask implements Runnable {

        Socket socket = null;

        public ExporterTask(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            try(ObjectInputStream inputStream = new ObjectInputStream(socket.getInputStream());
                ObjectOutputStream outputStream = new ObjectOutputStream(socket.getOutputStream())) {
                String interfaceName = inputStream.readUTF();
                
                //硬编码形式指定接口实现
                if(interfaceName.equals("EchoService")) {
                    interfaceName = "EchoServiceImpl";
                }
                Class<?> service = Class.forName(interfaceName);
                String methodName = inputStream.readUTF();
                Class<?>[] parameterTypes = (Class<?>[])inputStream.readObject();
                Method method = service.getMethod(methodName, parameterTypes);
                Object[] arguments = (Object[])inputStream.readObject();
                Object result = method.invoke(service.newInstance(), arguments);
                outputStream.writeObject(result);
            } catch (Throwable e) {
                e.printStackTrace();
            } finally {
                if(socket != null) {
                    try {
                        socket.close();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
  }
  ```

  `RpcExporter` 的主要作用：

  - 作为服务端，监听客户端的 TCP 连接，接收到新的客户端连接之后，将其封装成 Task，由线程池执行。
  - 将客户端发送的码流反序列化成对象，反射调用服务实现者，获取执行结果。
  - 将执行结果对象反序列化，通过 Socket 发送给客户端。
  - 远程服务调用完成后，释放 Socket 等连接资源，防止句柄泄露。 

- `RpcImporter`

  ```Java
  public class RpcImporter<S> {
      public S importer(final Class<?> serviceClass, final InetSocketAddress addr) {
          return (S) Proxy.newProxyInstance(
                  serviceClass.getClassLoader(),
                  new Class<?>[]{serviceClass.getInterfaces()[0]},
                  new InvocationHandler() {
              @Override
              public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                  Socket socket = null;
                  ObjectInputStream inputStream = null;
                  ObjectOutputStream outputStream = null;
                  try {
                      socket = new Socket();
                      socket.connect(addr);
                      outputStream = new ObjectOutputStream(socket.getOutputStream());
                      outputStream.writeUTF(serviceClass.getName());
                      outputStream.writeUTF(method.getName());
                      outputStream.writeObject(method.getParameterTypes());
                      outputStream.writeObject(args);
                      inputStream = new ObjectInputStream(socket.getInputStream());
                      return inputStream.readObject();
                  } catch (Exception e) {
                      e.printStackTrace();
                      return "error";
                  } finally {
                      if(socket != null) socket.close();
                      if(inputStream != null) inputStream.close();
                      if(outputStream != null) outputStream.close();
                  }
              }
          });
      }
  }
  ```

  `RpcImporter` 的主要功能：

  - 将本地的接口调用转化成 JDK 的动态代理，在动态代理中实现接口的远程调用。
  - 创建 Socket 客户端，根据指定地址连接远程服务提供者。
  - 将远程服务调用所需的接口类、方法名、参数列表等编码后发送给服务提供者。
  - 同步阻塞等待服务端返回应答，获取应答之后返回数据。

- Main

  ```java
  public class Main {

      private static final String host = "localhost";
      private static final int port = 9300;

      public static void main(String[] args) throws Exception {
          new Thread(new Runnable() {
              @Override
              public void run() {
                  try {
                      RpcExporter.exporter(host, port);
                  } catch (Exception e) {
                      e.printStackTrace();
                  }
              }
          }).start();

        	RpcImporter<EchoService> importer = new RpcImporter<EchoService>();
        	EchoService echoService = importer.importer(
                EchoService.class, new InetSocketAddress(host, port));
        	System.out.println(echoService.echo("Are you ok ?"));
      }
  }
  ```
- 运行结果

  ```
  Are you ok ? --> I am ok.
  ```

存在的问题：`RpcExporter` 中为了 demo 的简便，使用了硬编码形式指定接口的实现类，实际项目中可以用 Spring IoC 的方式配置。