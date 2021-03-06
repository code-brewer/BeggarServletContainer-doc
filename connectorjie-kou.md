# Connector接口

上一步后，我们完成了一个可以接收Socket请求的服务器。这时大师又说话了，昨天周末看片去了，有个单元测试TestServer

 没跑，你跑个看看，猜猜能跑过不。一跑果然不行啊，单元测试一直转圈，就不动。

![](/assets/unit-test-never-stop.jpg)

因为server.start\(\);会让当前线程无限循环，不断等待Socket请求，所以下面的单元测试方法根本不会走到断言那一步，也不会退出，所以大家都卡主了。

```java
@Test
public void testServerStart() throws IOException {
    server.start();
    assertTrue("服务器启动后，状态是STARTED", server.getStatus().equals(ServerStatus.STARTED));
}
```

修改起来很简单，让server.start\(\);在单独的线程里面执行就好，然后再循环判断ServerStatus是否为STARTED，等待服务器启动。

如下：

```java
@Test
public void testServerStart() throws IOException {  
    server.start();
    //如果server未启动，就sleep一下
    while (server.getStatus().equals(ServerStatus.STOPED)) {
        logger.info("等待server启动");
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            logger.error(e.getMessage(), e);
        }
    }
    assertTrue("服务器启动后，状态是STARTED", server.getStatus().equals(ServerStatus.STARTED));
}
```

这时大师又说了，循环判断服务器是否启动的代码片段，和TestServerAcceptRequest里面有重复代码，启动Server的代码也是重复的，一看就是Ctrl+c Ctrl+v的，你就不会抽象出一个父类啊。再重构：

```java
public abstract class TestServerBase {
    private static Logger logger = LoggerFactory.getLogger(TestServerBase.class);

    /**
     * 在单独的线程中启动Server，如果启动不成功，抛出异常
     *
     * @param server
     */
    protected void startServer(Server server) {
        //在另外一个线程中启动server
        new Thread(() -> {
            try {
                server.start();
            } catch (IOException e) {
                //转为RuntimeException抛出，避免异常丢失
                throw new RuntimeException(e);
            }
        }).start();
    }

    /**
     * 等待Server启动
     *
     * @param server
     */
    protected void waitServerStart(Server server) {
        //如果server未启动，就sleep一下
        while (server.getStatus().equals(ServerStatus.STOPED)) {
            logger.info("等待server启动");
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                logger.error(e.getMessage(), e);
            }
        }
    }
}
```

和Server相关的单元测试都可以extends于TestServerBase。

```java
public class TestServer extends TestServerBase {
    ... ...
    @Test
    public void testServerStart() {
        startServer(server);
        waitServerStart(server);
        assertTrue("服务器启动后，状态是STARTED", server.getStatus().equals(ServerStatus.STARTED));
    }
    ... ...
}
```

```java
public class TestServerAcceptRequest extends TestServerBase {
    ... ...
    @Test
    public void testServerAcceptRequest() {
        // 如果server没有启动，首先启动server
        if (server.getStatus().equals(ServerStatus.STOPED)) {
            startServer(server);
            waitServerStart(server);
            .... ...
    }   
    ... ...
}
```

再次执行单元测试，一切都OK。搞定单元测试后，大师又说了，看看你写的SimpleServer的start方法，

SimpleServe当前就是用来监听并接收Socket请求的，start方法就应该如其名，只是启动监听，修改ServerStatus为STARTED，接受请求什么的和start方法有毛关系，弄出去。

按照大师说的重构一下，单独弄个accept方法，专门用于接受请求。

```java
    @Override
    public void start() throws IOException {
        //监听本地端口，如果监听不成功，抛出异常
        this.serverSocket = new ServerSocket(this.port);
        this.serverStatus = ServerStatus.STARTED;
        accept();
        return;

    }

    private void accept() {
        while (true) {
            Socket socket = null;
            try {
                socket = serverSocket.accept();
                logger.info("新增连接：" + socket.getInetAddress() + ":" + socket.getPort());
            } catch (IOException e) {
                logger.error(e.getMessage(), e);
            } finally {
                IoUtils.closeQuietly(socket);
            }
        }
    }
```

这时大师又发话 了 ，我要用SSL，你直接new ServerSocket有啥用，重构去。

从start方法里面其实可以看到，Server启动接受\响应请求的组件后，组件的任何操作就和Server对象没一毛钱关系了，Server只是管理一下组件的生命周期而已。那么接受\响应请求的组件可以抽象出来，这样Server就不必和具体实现打交道了。

按照Tomcat和Jetty的惯例，接受\响应请求的组件叫Connector，生命周期也可以抽象成一个接口LifeCycle。根据这个思路去重构。

```java
public interface LifeCycle {

    void start();

    void stop();
}
```

```java
public abstract class Connector implements LifeCycle {
    @Override
    public void start() {
        init();
        acceptConnect();
    }

    protected abstract void init() throws ConnectorException;

    protected abstract void acceptConnect() throws ConnectorException;
}
```

将SimpleServer中和Socket相关的代码全部移动到SocketConnector里面

```java
public class SocketConnector extends Connector {
    ... ...
    @Override
    protected void init() throws ConnectorException {
        //监听本地端口，如果监听不成功，抛出异常
        try {
            this.serverSocket = new ServerSocket(this.port);
            this.started = true;
        } catch (IOException e) {
            throw new ConnectorException(e);
        }
    }
    @Override
    protected void acceptConnect() throws ConnectorException {
        new Thread(() -> {
            while (true && started) {
                Socket socket = null;
                try {
                    socket = serverSocket.accept();
                    LOGGER.info("新增连接：" + socket.getInetAddress() + ":" + socket.getPort());
                } catch (IOException e) {
                    //单个Socket异常，不要影响整个Connector
                    LOGGER.error(e.getMessage(), e);
                } finally {
                    IoUtils.closeQuietly(socket);
                }
            }
        }).start();
    }

    @Override
    public void stop() {
        this.started = false;
        IoUtils.closeQuietly(this.serverSocket);
    }  
    ... ...

 }
```

SimpleServer重构为

```java
public class SimpleServer implements Server {
    ... ...
    private SocketConnector socketConnector;

    ... ...

    @Override
    public void start() throws IOException {
        socketConnector.start();
        this.serverStatus = ServerStatus.STARTED;
    }

    @Override
    public void stop() {
        socketConnector.stop();
        this.serverStatus = ServerStatus.STOPED;
        logger.info("Server stop");
    }
    ... ...
}
```

跑单元测试，全部OK，证明代码没问题。

大师瞄了一眼，说不 给你说了么，面向抽象编程啊，为毛还直接引用了SocketConnector，还有，我想要多个Connector，继续给我重构去。

重构思路简单，将SocketConnector替换为抽象类型Connector即可，但是怎么实例化呢，总有地方要处理这个抽象到具体的过程啊，这时又轮到Factory类干这个脏活了。

再次重构。

增加ConnectorFactory接口，及其实现SocketConnectorFactory

```java
public class SocketConnectorFactory implements ConnectorFactory {
    private final SocketConnectorConfig socketConnectorConfig;

    public SocketConnectorFactory(SocketConnectorConfig socketConnectorConfig) {
        this.socketConnectorConfig = socketConnectorConfig;
    }

    @Override
    public Connector getConnector() {
        return new SocketConnector(this.socketConnectorConfig.getPort());
    }
}
```

SimpleServer也进行相应修改，不再实例化任何具体实现，只通过构造函数接收对应的抽象。

```java
public class SimpleServer implements Server {
    private static Logger logger = LoggerFactory.getLogger(SimpleServer.class);
    private volatile ServerStatus serverStatus = ServerStatus.STOPED;
    private final int port;
    private final List<Connector> connectorList;

    public SimpleServer(ServerConfig serverConfig, List<Connector> connectorList) {
        this.port = serverConfig.getPort();
        this.connectorList = connectorList;
    }

    @Override
    public void start() {
        connectorList.stream().forEach(connector -> connector.start());
        this.serverStatus = ServerStatus.STARTED;
    }

    @Override
    public void stop() {
        connectorList.stream().forEach(connector -> connector.stop());
        this.serverStatus = ServerStatus.STOPED;
        logger.info("Server stop");
    }
    ... ...

}
```

ServerFactory也进行修改，将Server需要的依赖传递到Server的构造函数中。

```java
public class ServerFactory {
    /**
     * 返回Server实例
     *
     * @return
     */
    public static Server getServer(ServerConfig serverConfig) {
        List<Connector> connectorList = new ArrayList<>();
        ConnectorFactory connectorFactory =
                new SocketConnectorFactory(new SocketConnectorConfig(serverConfig.getPort()));
        connectorList.add(connectorFactory.getConnector());
        return new SimpleServer(serverConfig,connectorList);
    }
}
```

这样我们就将对具体实现的依赖限制到了不多的几个Factory中，最核心的Server部分只操作了抽象。

执行所有单元测试，再次全部成功。

虽然目前为止，Server还是只能接收请求，但是代码结构还算OK，为下面编写请求处理做好了准备。

完整代码：https://github.com/pkpk1234/BeggarServletContainer/tree/step3

分支step3

![](/assets/git-br-step3.jpg)



