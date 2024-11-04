# springboot整合modbus4j实现tcp通讯

## 前言

本文基于springboot和modbus4j进行简单封装，达到开箱即用的目的，目前本方案仅实现了tcp通讯。代码会放在最后，按照使用方法操作后就可以直接使用

## 介绍

在使用本方案之前，有必要对modbus有一个简单的认知，其中包含modbus协议

### Modbus通讯协议简介

​		**Modbus**是一种串行[通信协议](https://baike.baidu.com/item/通信协议/0?fromModule=lemma_inlink)，是Modicon公司（现在的[施耐德电气](https://baike.baidu.com/item/施耐德电气/0?fromModule=lemma_inlink)Schneider Electric）于1979年为使用[可编程逻辑控制器](https://baike.baidu.com/item/可编程逻辑控制器/0?fromModule=lemma_inlink)（PLC）通信而发表。Modbus已经成为工业领域通信协议的业界标准（De facto），并且现在是工业电子设备之间常用的连接方式。 [1]Modbus比其他通信协议使用的更广泛的主要原因有：

1. 公开发表并且无版权要求
2. 易于部署和维护
3. 对供应商来说，修改移动本地的比特或字节没有很多限制

Modbus允许多个 (大约240个) 设备连接在同一个网络上进行通信，举个例子，一个测量温度和湿度的装置，并且将结果发送给[计算机](https://baike.baidu.com/item/计算机/140338?fromModule=lemma_inlink)。在数据采集与监视控制系统（SCADA）中，Modbus通常用来连接监控计算机和[远程终端控制系统](https://baike.baidu.com/item/远程终端控制系统/0?fromModule=lemma_inlink)（RTU）。

### Modbus功能码（部分）

| 代码 | 名称                                      | 寄存器地址范围 | 位/字操作 | 操作数量   |
| ---- | ----------------------------------------- | -------------- | --------- | ---------- |
| 01   | 读线圈状态（Read Coils）                  | 00001 ~ 09999  | 位操作    | 单个或多个 |
| 02   | 读离散输入状态（Read Discrete Inputs）    | 10001 ~ 19999  | 位操作    | 单个或多个 |
| 03   | 读保存寄存器（Read Holding Registers）    | 40001 ~ 49999  | 字操作    | 单个或多个 |
| 04   | 读输入寄存器（Read Input Registers）      | 30001 ~ 39999  | 字操作    | 单个或多个 |
| 05   | 写单个线圈（Write Single Coil）           | 00001 ~ 09999  | 字操作    | 单个       |
| 06   | 写单个保存寄存器（Write Single Register） | 40001 ~ 49999  | 字操作    | 单个       |

### Modbus仿真软件

**modbus poll：**modbus主机（master）仿真器，用于测试和调试modbus从设备。该软件支持modbus rtu、ASCII、TCP/IP。用来帮助开发人员测试modbus从设备，或者其它modbus协议的测试和仿真。它支持多文档接口，即，可以同时监视多个从设备/数据域。每个窗口简单地设定从设备ID，功能，地址，大小和轮询间隔。你可以从任意一个窗口读写寄存器和线圈。如果你想改变一个单独的寄存器，简单地双击这个值即可。或者你可以改变多个寄存器/线圈值。提供数据的多种格式方式，比如浮点、双精度、长整型（可以字节序列交换）。

**modbus slave：**modbus从设备（slave）仿真器，可以仿真32个从设备/地址域。每个接口都提供了对EXCEL报表的OLE自动化支持。主要用来模拟Modbus从站设备，接收主站的命令包，回送数据包。帮助Modbus通讯设备开发人员进行Modbus通讯协议的模拟和测试，用于模拟、测试、调试Modbus通讯设备。可以32个窗口中模拟多达32个Modbus子设备。与Modbus Poll的用户界面相同，支持功能01、02、03、04、05、06、15、16、22和23，监视串口数据。

![image-20241016140552969](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241016140552969.png)

这两款软件请自行下载

## 使用方式

本方案基于springboot，可以在springboot中引入该项目后简单操作（maven安装到本地）直接使用，高效方便~

### 引入jar包

```xml
<dependency>
	<groupId>com.dashuai</groupId>
	<artifactId>modbus-spring-boot-starter</artifactId>
	<version>0.0.1-SNAPSHOT</version>
</dependency>
```

### 编写配置文件

配置根据实际情况下编写，支持一下所有的配置，至于是服务器做主还是做从，根据实际情况而定

```yml
modbus:
  tcp:
    master:
      # 默认的主设备地址和端口
      default-ip: 192.168.11.180
      default-port: 502
      # 主设备的地址和端口集合，按照顺序一一对应
      ips:
        - 192.168.11.180
      ports:
        - 1502
    slave:
      # 从设备端口
      port: 1502
      encapsulated: false
      # 从设备详细配置
      process-images:
        # 从设备号，可创建多个
        - slave-id: 1
          # 线圈
          coils:
            # 地址位和默认值
            - offset: 0
              value: true
          # 离散输入状态
          inputs:
            - offset: 100
              value: true
          # 保持寄存器
          holding-register:
            # 起始地址位和寄存器个数
            start-offset: 200
            count: 100
          # 输入寄存器
          input-register:
            start-offset: 300
            count: 100
        - slave-id: 2
          coils:
            - offset: 0
              value: true
          inputs:
            - offset: 100
              value: true
          holding-register:
            start-offset: 200
            count: 100
          input-register:
            start-offset: 300
            count: 100
```

### 注入ModbusTCPMaster对象

```java
    @Autowired
    private ModbusTCPMaster modbusTCPMaster;
```

### 支持的方法

1. 读取01/02/03/04功能码的数据
2. 按照数据类型（目标类型、返回类型）直接读取
3. 批量读取
4. ...

### Master测试案例

由于是测试Modbus TCP 的通讯，所以我准备了两台机器，我本地ip是192.168.11.180，测试机器ip为192.168.11.194

Master可以理解为客户端，Slave可以理解为服务端，客户端向服务器请求（读取）数据，我在我本地和测试机器中分别启动一个modbus slave，在我本地使用当前方案进行读取

#### 读取线圈状态

在本地中开启一个modbus slave，连接方式为TCP/IP，端口为502

![image-20241017155742240](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241017155742240.png)

选择Setup->Slave Definition，开启一个从设备，从设备为1，功能码为01，地址位从10开始，一共2个

![image-20241017160426459](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241017160426459.png)

设置10号地址位的值为0（false），11号地址位的值为1（true），使用以上同样的方式在192.168.11.194上设置一个从设备，端口为1502，从设备号为1，10号地址位的值为1（true），11号地址位的值为0（flase）

![image-20241017160711847](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241017160711847.png)

测试程序配置文件，默认的master为读取我本地的数据

```yml
modbus:
  tcp:
    master:
      # 默认的主设备地址和端口
      default-ip: 192.168.11.180
      default-port: 502
      # 主设备的地址和端口集合，按照顺序一一对应
      ips:
        - 192.168.11.194
      ports:
        - 1502
```

注入ModbusTCPMaster对象

```java
    @Autowired
    private ModbusTCPMaster modbusTCPMaster;
```

测试方法

```java
	@Test
    public void readCoilStatus() throws ErrorResponseException, ModbusTransportException, ModbusInitException {

        // 使用默认的master进行读取
        Boolean value = modbusTCPMaster.readCoilStatus(1, 10);
        System.out.println("default,slaveId:1,address:10,value = " + value);
        value = modbusTCPMaster.readCoilStatus(1, 11);
        System.out.println("default,slaveId:1,address:11,value = " + value);
        // 指定ip进行读取，默认的master也可以进行指定ip读取
        value = modbusTCPMaster.readCoilStatus("192.168.11.194", 1502, 1, 10);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:2,address:10,value = " + value);
        value = modbusTCPMaster.readCoilStatus("192.168.11.194", 1502, 1, 11);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:2,address:11,value = " + value);
    }
```

测试结果

![image-20241021163937169](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021163937169.png)

#### 读取离散输入状态

在本地新建一个slave，File->New，选择Setup->Slave Definition将slaveId设置为2，选择02功能码，地址位从100开始，初始化2个寄存器，100地址位的值设置为1，101地址位的值设置为0，用同样的方式在192.168.11.194那台测试服务器上配置，地址位一样，但是值都配置为1进行测试读取

![image-20241021165637473](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021165637473.png)

测试方法

```java
	@Test
    public void readInputStatus() throws ErrorResponseException, ModbusTransportException, ModbusInitException {

        // 使用默认的master进行读取
        Boolean value = modbusTCPMaster.readInputStatus(2, 100);
        System.out.println("default,slaveId:2,address:100,value = " + value);
        value = modbusTCPMaster.readInputStatus(2, 101);
        System.out.println("default,slaveId:2,address:101,value = " + value);
        // 指定ip进行读取
        value = modbusTCPMaster.readInputStatus("192.168.11.194", 1502, 2, 100);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:2,address:100,value = " + value);
        value = modbusTCPMaster.readInputStatus("192.168.11.194", 1502, 2, 101);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:2,address:101,value = " + value);
    }
```

测试结果

![image-20241021165847599](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021165847599.png)

#### 读取保存寄存器

##### 读取整数

在本地新建一个slave，File->New，选择Setup->Slave Definition将slaveId设置为3，选择03功能码，地址位从200开始，初始化2个寄存器，200地址位的值设置为20，201地址位的值设置为21，用同样的方式在192.168.11.194那台测试服务器上配置，地址位一样，值分别设置成30和31

![image-20241021170146747](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021170146747.png)

测试方法

```java
	@Test
    public void read03Short() throws ErrorResponseException, ModbusTransportException, ModbusInitException {

        Short value = modbusTCPMaster.read03Short(3, 200);
        System.out.println("default,slaveId:3,address:200,value = " + value);
        value = modbusTCPMaster.read03Short(3, 201);
        System.out.println("default,slaveId:3,address:201,value = " + value);
        value = modbusTCPMaster.read03Short("192.168.11.194", 1502, 3, 200);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:3,address:200,value = " + value);
        value = modbusTCPMaster.read03Short("192.168.11.194", 1502, 3, 201);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:3,address:201,value = " + value);
    }
```

测试结果

![image-20241021171146468](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021171146468.png)

##### 读取小数

在本地继续新建一个slave，File->New，选择Setup->Slave Definition将slaveId设置为4，选择03功能码，地址位从300开始，初始化4个寄存器

![image-20241021172206160](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021172206160.png)

按住Ctrl+A全选，点击鼠标右键，选择Format，选择Float AB CD，双击数值区域将300地址位的值设置成33.33，302地址位的数值设置成44.44，在192.168.11.194测试服务器上做同样的操作，数值分别设置成55.55和66.66

![image-20241021172343483](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021172343483.png)

![image-20241021172607128](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021172607128.png)

测试代码

```java
	@Test
    public void read03FloatABCD() throws ErrorResponseException, ModbusTransportException, ModbusInitException {

        Float value = modbusTCPMaster.read03FloatABCD(4, 300);
        System.out.println("default,slaveId:4,address:300,value = " + value);
        value = modbusTCPMaster.read03FloatABCD(4, 302);
        System.out.println("default,slaveId:4,address:302,value = " + value);

        value = modbusTCPMaster.read03FloatABCD("192.168.11.194", 1502, 4, 300);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:4,address:300,value = " + value);
        value = modbusTCPMaster.read03FloatABCD("192.168.11.194", 1502, 4, 302);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:4,address:302,value = " + value);
    }
```

测试结果

![image-20241021174011993](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241021174011993.png)

#### 批量读取

在本地新建一个slave，File->New，选择Setup->Slave Definition将slaveId设置为5，选择03功能码，地址位从400开始，初始化5个寄存器，400地址位的值设置为0，后续5个地址的数值依次加1，分别为0、1、2、3、4，用同样的方式在192.168.11.194那台测试服务器上配置，地址位一样，值分别设置成0、11.11、22.22、33.33、44.44、55.55，批量读取支持两种方式进行

本地slave

![image-20241101165518769](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101165518769.png)

192.168.11.194测试机器上slave配置

![image-20241101165554364](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101165554364.png)

测试方法1

```java
	@Test
    public void testBatchRead() throws ModbusTransportException, ErrorResponseException {

        final ArrayList<BatchReadParam> batchReadParamList = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            final BatchReadParam batchReadParam = new BatchReadParam(5, 400 + i);
            batchReadParamList.add(batchReadParam);
        }

        final Map<Integer, Short> resultMap = modbusTCPMaster.batchRead(batchReadParamList,
                DataType.TWO_BYTE_INT_SIGNED, Short.class);
        System.out.println("default,slaveId:5,address:400~404,resultMap = " + resultMap);
    }
```

测试结果1

![image-20241101165656423](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101165656423.png)

测试方法2

```java
	@Test
    public void testBatchRead2() throws ModbusTransportException, ErrorResponseException, ModbusInitException {

        List<Integer> floatOffsets = Arrays.asList(400, 402, 404, 406, 408);

        final Map<Integer, Float> resultMap = modbusTCPMaster.batchRead("192.168.11.194", 1502,5,
                floatOffsets, DataType.FOUR_BYTE_FLOAT, Float.class);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:5,address:400~408,resultMap = " + resultMap);
    }
```

测试结果2

![image-20241101165752508](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101165752508.png)

#### 写入线圈状态

在本地新建一个slave，File->New，选择Setup->Slave Definition将slaveId设置为6，选择01功能码，地址位从600开始，初始化2个寄存器，用同样的方式在192.168.11.194那台测试服务器上配置，初始化的地址位一样。我们分别向本地的600地址位写入true和194上的600以及601批量写入true

本地配置

![image-20241101170445460](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101170445460.png)

192.168.11.194测试机器上slave配置

![image-20241101170516539](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101170516539.png)

测试代码1

```java
    @Test
    public void testWriteCoil() throws ModbusTransportException {

        final boolean result = modbusTCPMaster.writeCoil(6, 600, true);
        System.out.println("default,slaveId:6,address:600,result = " + result);
    }
```

测试结果1

![image-20241101170901798](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101170901798.png)

测试代码2

```java
	@Test
    public void testBatchWriteCoil() throws ModbusTransportException {

        final boolean[] writeValueArr = {true, true};
        final boolean result = modbusTCPMaster.batchWriteCoil("192.168.11.194", 1502, 6, 600, writeValueArr);
        System.out.println("ip:192.168.11.194,port:1502,slaveId:6,address:600~601,result = " + result);
    }
```

测试结果2

![image-20241101171228623](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101171228623.png)

![image-20241101171243945](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101171243945.png)

#### 写入保存寄存器

在本地新建一个slave，File->New，选择Setup->Slave Definition将slaveId设置为7，选择03功能码，地址位从700开始，初始化4个寄存器，我们向本地的700地址位写入11.11

本地配置

![image-20241101172924234](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101172924234.png)

测试代码

```java
    @Test
    public void testWriteHoldingRegister() throws ModbusTransportException, ErrorResponseException {

        modbusTCPMaster.writeHoldingRegister(7, 700, 11.11, DataType.FOUR_BYTE_FLOAT);
    }
```

测试结果

![image-20241101172823442](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101172823442.png)

**其实还有一些方法，这里就不进行逐个测试了，有兴趣的可以自己搭建测试一下~~~**

### Slave测试案例

使用我本地的环境进行测试，测试在我本地启动一个slave，并初始化对应的地址位和数值，然后使用modbus poll软件进行连接和读取

测试程序配置文件（主要是slave部分）

```yml
modbus:
  tcp:
    master:
      # 默认的主设备地址和端口
      default-ip: 192.168.11.180
      default-port: 502
      # 主设备的地址和端口集合，按照顺序一一对应
      ips:
        - 192.168.11.194
      ports:
        - 1502
    slave:
      # 从设备端口
      port: 2502
      # 从设备详细配置
      process-images:
        # 从设备号，可创建多个
        - slave-id: 1
          # 线圈
          coils:
            # 地址位和默认值
            - offset: 1
              value: true
          # 离散输入状态
          inputs:
            - offset: 100
              value: true
          # 保持寄存器
          holding-register:
            # 起始地址位和寄存器个数
            start-offset: 200
            count: 100
          # 输入寄存器
          input-register:
            start-offset: 300
            count: 100
        - slave-id: 2
          coils:
            - offset: 500
              value: true
```

**解释：**我在我本地启动了2个从设备，从设备号分别为1和2，在从设备1中，我启动了线圈状态，地址位为1，默认数值为true，启动了离散输入状态，地址位为100，默认数值为true，启动了保存寄存器，起始地址位为200，一共初始化100个地址位，启动了输入寄存器，起始位置为300，也是初始化100个地址位；在从设备2中，我只启动了线圈状态，地址位为500，默认数值为true

测试代码

```java
	@Test
    public void testModbusSlave() {

        System.out.println("我启动了slave~~~~~");
        System.out.println("modbusTCPSlave = " + modbusTCPSlave);
        try {
            Thread.sleep(600000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

测试结果

![image-20241101175008614](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101175008614.png)

![image-20241101175117260](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101175117260.png)

![image-20241101175221033](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101175221033.png)

![image-20241101175254835](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101175254835.png)

![image-20241101175338392](https://qm-markdown-img.oss-cn-shanghai.aliyuncs.com/typora-img/image-20241101175338392.png)

可以看到全部都能够正常连接并读取，写入的话就不测试了

到此结束，附上代码，请动动小手给个star，谢谢~~~

gitee：

githup：
