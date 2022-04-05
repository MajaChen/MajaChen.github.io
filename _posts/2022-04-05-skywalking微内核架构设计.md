---
layout: post
title: skywalking微内核架构设计
subtitle: skywalking 微内核 架构设计
cover-img: /assets/img/18.jpg
thumbnail-img: /assets/img/animals/TawnyFrogmouth.jpg
tags: [skywalking,架构设计]
---

# skywalking微内核架构设计
## 前言

&emsp;&emsp;微内核架构设计说的直白一点就是“插件化”架构设计，在此我不想浪费笔墨，仅以skywalking为例来分析它是如何实现模块、模块定义、服务三方之间的松耦合并在此基础上实现可插拔。

## 从Java SPI说起

### 什么是SPI

&emsp;&emsp;详情可以参看[这篇文章](https://juejin.cn/post/6844903605695152142#heading-3)。我们定义了一个接口，这个接口有多种具体实现，我们希望能够在运行时通过配置文件来决定哪种实现应该被载入（对应class字节码）。提供这种支持的技术就是Java SPI。

![image-20220405134118593](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20220405134118593.png)

### 程序实例

**接口定义**

```java
public interface ObjectSerializer {

    byte[] serialize(Object obj) throws ObjectSerializerException;

    <T> T deSerialize(byte[] param, Class<T> clazz) throws ObjectSerializerException;

    String getSchemeName();
}

```

&emsp;&emsp;它定义了两个方法serize和deSerialize，分别用于序列化Java对象和反序列化Java对象。

**实现定义**

```java
public class KryoSerializer implements ObjectSerializer {

    @Override
    public byte[] serialize(Object obj) throws ObjectSerializerException {
        byte[] bytes;
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        try {
            //获取kryo对象
            Kryo kryo = new Kryo();
            Output output = new Output(outputStream);
            kryo.writeObject(output, obj);
            bytes = output.toBytes();
            output.flush();
        } catch (Exception ex) {
            throw new ObjectSerializerException("kryo serialize error" + ex.getMessage());
        } finally {
            try {
                outputStream.flush();
                outputStream.close();
            } catch (IOException e) {

            }
        }
        return bytes;
    }

    @Override
    public <T> T deSerialize(byte[] param, Class<T> clazz) throws ObjectSerializerException {
        T object;
        try (ByteArrayInputStream inputStream = new ByteArrayInputStream(param)) {
            Kryo kryo = new Kryo();
            Input input = new Input(inputStream);
            object = kryo.readObject(input, clazz);
            input.close();
        } catch (Exception e) {
            throw new ObjectSerializerException("kryo deSerialize error" + e.getMessage());
        }
        return object;
    }

    @Override
    public String getSchemeName() {
        return "kryoSerializer";
    }

}

```

&emsp;&emsp;第一种实现方式：我们基于kryo这个包来具体实现Java对象的序列化和反序列化，对应类KryoSerializer。

```java
public class JavaSerializer implements ObjectSerializer {
    @Override
    public byte[] serialize(Object obj) throws ObjectSerializerException {
        ByteArrayOutputStream arrayOutputStream;
        try {
            arrayOutputStream = new ByteArrayOutputStream();
            ObjectOutput objectOutput = new ObjectOutputStream(arrayOutputStream);
            objectOutput.writeObject(obj);
            objectOutput.flush();
            objectOutput.close();
        } catch (IOException e) {
            throw new ObjectSerializerException("JAVA serialize error " + e.getMessage());
        }
        return arrayOutputStream.toByteArray();
    }

    @Override
    public <T> T deSerialize(byte[] param, Class<T> clazz) throws ObjectSerializerException {
        ByteArrayInputStream arrayInputStream = new ByteArrayInputStream(param);
        try {
            ObjectInput input = new ObjectInputStream(arrayInputStream);
            return (T) input.readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new ObjectSerializerException("JAVA deSerialize error " + e.getMessage());
        }
    }

    @Override
    public String getSchemeName() {
        return "javaSerializer";
    }

}

```

&emsp;&emsp;第二种实现方式：基于Java原生的序列化和反序列化方式，对应类JavaSerializer。

&emsp;&emsp;现在的问题是：我们希望能够以配置文件的方式在运行时决定接口ObjectSerializer 该由哪个类来实现。SPI应运而生。使用SPI，我们需要在classpath下的META-INF/services/目录中创建一个以完整接口命名的文件：

![img](https://gitee.com/xinyuanchen/image_collection/raw/master/1635dec21502ba4d~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

&emsp;&emsp;文件内容是该接口的具体实现类：

```
com.blueskykong.javaspi.serializer.KryoSerializer
com.blueskykong.javaspi.serializer.JavaSerializer
```

&emsp;&emsp;那么我们在使用该接口、获取接口实例的时候，就可以指定哪个实现类应该被加载：

**获取接口实例**

```java
@Service
public class SerializerService {

    public ObjectSerializer getObjectSerializer() {
        ServiceLoader<ObjectSerializer> serializers = ServiceLoader.load(ObjectSerializer.class);

        final Optional<ObjectSerializer> serializer = StreamSupport.stream(serializers.spliterator(), false)
                .findFirst();

        return serializer.orElse(new JavaSerializer());
    } 
}

```

&emsp;&emsp;实现类的加载需要借助java.util.ServiceLoader，他会基于配置文件将所有实现类全部加载进来，然后我们进行进一步筛选——取配置文件中第一个条目对应的实现类，如果配置文件是空的就返回JavaSerializer实例。

&emsp;&emsp;注意：ObjectSerializer不一定得是接口，也可以是抽象类，我们在下一小节将会看到。

## 基于SPI技术加载模块与模块实现

### 基于SPI技术加载模块

&emsp;&emsp;skywalking基于松耦合、模块化的设计理念，将整个项目划分成多个module，对应抽象类ModuleDefine。其类型定义如下：

```java
public abstract class ModuleDefine implements ModuleProviderHolder {...}
```

&emsp;&emsp;基于SPI技术，用户可以基于实际应用场景选择使用哪些模块、不使用哪些模块。其接口命名文件org.apache.skywalking.oap.server.library.module.ModuleDefine内容如下：

```java
org.apache.skywalking.oap.server.core.storage.StorageModule
org.apache.skywalking.oap.server.core.cluster.ClusterModule
org.apache.skywalking.oap.server.core.CoreModule
org.apache.skywalking.oap.server.core.query.QueryModule
org.apache.skywalking.oap.server.core.alarm.AlarmModule
org.apache.skywalking.oap.server.core.exporter.ExporterModule
```

&emsp;&emsp;从配置文件来看，StorageModule、ClusterModule、CoreModule、QueryModule、AlarmModule、ExporterModule这些模块均会被加载。

&emsp;&emsp;那么，实际的加载情况又是怎么样的呢？我们来看看加载代码：

```java
public void init(
        ApplicationConfiguration applicationConfiguration){
        String[] moduleNames = applicationConfiguration.moduleList();
        ServiceLoader<ModuleDefine> moduleServiceLoader = ServiceLoader.load(ModuleDefine.class);
        ServiceLoader<ModuleProvider> moduleProviderLoader = ServiceLoader.load(ModuleProvider.class);

        LinkedList<String> moduleList = new LinkedList<>(Arrays.asList(moduleNames));
        for (ModuleDefine module : moduleServiceLoader) {
            for (String moduleName : moduleNames) {
                if (moduleName.equals(module.name())) {
                    module.prepare(this, applicationConfiguration.getModuleConfiguration(moduleName), moduleProviderLoader);
                    loadedModules.put(moduleName, module);
                    moduleList.remove(moduleName);
                }
            }
        }
        // Finish prepare stage
        isInPrepareStage = false;

        if (moduleList.size() > 0) {
            throw new ModuleNotFoundException(moduleList.toString() + " missing.");
        }

        BootstrapFlow bootstrapFlow = new BootstrapFlow(loadedModules);

        bootstrapFlow.start(this);
        bootstrapFlow.notifyAfterCompleted();
    }
```

&emsp;&emsp;实际上，skywalking除了基于SPI来控制哪些模块应该被加载，还引入了application.yml文件来进行作进一步筛选：

![image-20220405144518239](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20220405144518239.png)

&emsp;&emsp;其中cluster、core、storage都是模块名。moduleNames就是是从application.yml中提取出的模块名列表。

&emsp;&emsp;ServiceLoader把所有类加载出来之后，还要判断该模块是否在application.yml中有定义，即该模块的模块名(通过module.name()方法返回)命中application.yml定义的模块名。只有在application.yml中定义了的模块才会被真正加载，进入prepare逻辑。

### 基于SPI技术加载模块实现

&emsp;&emsp;而每个模块又对应多种实现。比如storage模块，可以基于ES来实现，也可以基于MYSQL来实现，就引入了一个新的抽象类ModuleProvider。同样，它在META-INF/services/下也有对应的接口命名文件org.apache.skywalking.oap.server.library.module.ModuleProvider：

```java
org.apache.skywalking.oap.server.storage.plugin.elasticsearch.StorageModuleElasticsearchProvider
```

&emsp;&emsp;ModuleProvider有多个实现类，同样地，具体选择哪个实现类，也要由application.yml配置文件来决定，比如上图中，storage模块有多个实现类:elasticsearch、h2等，最终选择由elasticsearch7来实现（selector关键字定义）。具体筛选逻辑见prepare方法，从略。

&emsp;&emsp;**个人点评**：ModuleDefine与ModuleProvider在逻辑上本身其实是一种接口与实现类的关系，而skywalking对SPI的成功运用之处在于：它将ModuleDefine和ModuleProvider均定义为接口，然后配置各自的实现类，完全消解了二者的实现与被实现关系，进一步降低了系统耦合度，最终实现的效果是——可以随意插拔某个模块、可以随意替换任意模块的实现。

## 模块和模块实现之间的服务约定

&emsp;&emsp;模块或者模块实现归根结底是要提供某些服务以完成某些功能。因此双方需要约定一个服务列表，ModuleProvider定义服务列表，ModuleProvider实现类注册相应服务。各服务只需要实现Servcie接口，与模块和模块定义没有任何关联。

&emsp;&emsp;ModuleDefine实现类定义服务列表：

```java
public class CoreModule extends ModuleDefine {
    public static final String NAME = "core";

    public CoreModule() {
        super(NAME);
    }

    @Override
    public Class[] services() {
        List<Class> classes = new ArrayList<>();
        classes.add(ConfigService.class);
        classes.add(DownSamplingConfigService.class);
        classes.add(NamingControl.class);
        classes.add(IComponentLibraryCatalogService.class);

        classes.add(IWorkerInstanceGetter.class);
        classes.add(IWorkerInstanceSetter.class);

        classes.add(MeterSystem.class);

        addServerInterface(classes);
        addReceiverInterface(classes);
        addInsideService(classes);
        addCacheService(classes);
        addQueryService(classes);
        addProfileService(classes);
        addOALService(classes);
        addManagementService(classes);

        classes.add(CommandService.class);

        return classes.toArray(new Class[]{});
    }
}
```

&emsp;&emsp;ModuleProvider实现类注册相应服务：

```java
public class CoreModuleProvider extends ModuleProvider {
 ......
  @Override
    public void prepare() throws ServiceNotProvidedException, ModuleStartException {
     WorkerInstancesService instancesService = new WorkerInstancesService();
        this.registerServiceImplementation(IWorkerInstanceGetter.class, instancesService);
        this.registerServiceImplementation(IWorkerInstanceSetter.class, instancesService);

        this.registerServiceImplementation(RemoteSenderService.class, new RemoteSenderService(getManager()));
        this.registerServiceImplementation(ModelCreator.class, storageModels);
        this.registerServiceImplementation(IModelManager.class, storageModels);
        this.registerServiceImplementation(ModelManipulator.class, storageModels);
        ......
        }
 }
```

&emsp;&emsp;在代码中使用服务：

```java
public class MetadataQuery implements GraphQLQueryResolver {

    private final ModuleManager moduleManager;
    private MetadataQueryService metadataQueryService;
    private MetadataQueryService getMetadataQueryService() {
        if (metadataQueryService == null) {
            this.metadataQueryService = moduleManager.find(CoreModule.NAME)
                                                     .provider()
                                                        .getService(MetadataQueryService.class);
        }
        return metadataQueryService;
    }
```

## 总结

&emsp;&emsp;**模块、模块实现、模块提供的服务构成了skywalking的“三驾马车”**，而借助SPI、服务约定等技术，使得三驾马车之间的耦合性尽可能最低，使系统取得了极高的。







