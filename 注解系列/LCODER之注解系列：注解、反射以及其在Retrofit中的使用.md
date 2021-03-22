## 注解
### 一、概念
注解又叫Java标注，是JDK5.0之后引入的一种注释机制，注解是[元数据](https://baike.baidu.com/item/%E5%85%83%E6%95%B0%E6%8D%AE)的一种形式，提供有关于程序但是不属于程序本身的数据。注解对它们注解的代码的操作没有直接影响。

### 二、声明一个注解
Java中的所有注解，默认都实现Annotation接口。

声明注解使用的是@interface关键字。
```
@Target(ElementType.TYPE) // 元注解①
@Retention(RetentionPolicy.SOURCE) // 元注解②
public @interface Test{
   String value(); //无默认值时必须传值，且如果只存在value元素需要传值的情况，则可以省略:元素名=
   int age() default 1;// 有默认值
}
```

##### [元注解](http://c.biancheng.net/view/7009.html)：
定义注解时，注解类也可以使用其他的注解来声明，对注解类型进行注解的注解类，被称为元注解。  
一般要声明的元注解有两个：  
*@Target：限制这个注解类可以作用在什么地方*  
*@Retention: 保留级别，这个注解保留到什么时候。*   
下面分别对这两个元注解进行详细的解释：  
**1. @Target**
> ElementType.ANNOTATION_TYPE可以应用于注解类型。  
> ElementType.CONSTRUCTOR可以应用于构造函数。  
> ElementType.FIELD可以应用于字段或属性。  
> ElementType.LOCAL_VARIABLE可以应用于局部变量。  
> ElementType.METHOD可以应用于方法。  
> ElementType.PACKAGE可以应用于包声明。  
> ElementType.PARAMETER可以应用于方法的参数。  
> ElementType.TYPE可以应用于类的任何元素。

**2.@Retention**
> RetentionPolicy.SOURCE - 标记的注解仅保留在源码级别中，并被编译器忽略。  
> RetentionPolicy.CLASS - 标记的注解在编译时由编译器保留，但 Java 虚拟机(JVM)会忽略。  
> RetentionPolicy.RUNTIME - 标记的注解由 JVM 保留，因此运行时环境可以使用它。  

三个值中 SOURCE < CLASS < RUNTIME，即CLASS包含了SOURCE，RUNTIME包含SOURCE、CLASS。  

### 三、注解的作用场景
**源码级注解**   
标记的注解仅保留在源码级别，即.java文件中。被编译器忽略。

**应用场景：IDE语法检查**  
*@IntDef*
##### 步骤1.创建一个IntDef注解
```
@Retention(SOURCE)   //源码级别注解
@Target({ANNOTATION_TYPE}) // 作用于类
public @interface IntDef {
     
     int[] value() default {};
     booleanflag() defaultfalse;
     booleanopen() defaultfalse;
    
}

```
##### 步骤2. 创建一个注解，使用@IntDef作为元注解，限定该注解限制的类型
@IntDef 注解的意义就是代替枚举，实现如方法入参的限制
```
public class Test{
    private static final int SUNDAY = 0;
    private static final int MONDAY = 1;
    
    @IntDef({SUNDAY,MONDAY})//用于指定该注解限制的类型
    @Target({ElementType.FIELD,ElementType.PARAMETER})
    @interface WeekDay{
        
        
    }
    
    @WeekDay
    private static int mCurrentDay;
    
    // 如果输入的不是SUNDAY和MONDAY，那么语法检查会报错误警告，但编译不会报错
    public static void setWeekDay(@WeekDay int currentDay){
    
    }
}
```

**CLASS级别注解**   
注解保留在源码.java文件和编译器.class文件中。  
**应用场景：APT （Anotation Processor Tools）注解处理程序**  
下面的代码中，有使用到JavaPoet的代码，有关于JavaPoet，会在下一节，[《LCODER之注解系列：ARouter框架详解以及手写一个ARouter框架》](https://juejin.cn/post/6901198064149594126)中进行详细的讲解
> 步骤：
> 1. 定义一个类继承AbstractProcessor
> 2. 在Gradle中开启自动服务
> 3. 在App中的gradle中引用annotationProcessor project
> 4. 在process方法中，进行注解的处理工作
```
@AutoService(Processor.class) // 启用服务
@SupportedAnnotationTypes({"com.kimliu.arouter_annotations.ARouter"}) // 告诉注解处理器，要处理哪个注解
@SupportedSourceVersion(SourceVersion.RELEASE_7) // 环境版本
@SupportedOptions("key") // 接受从App传递过来的值的key
public class ARouterProcessor extends AbstractProcessor {

    // 操作Elements的工具类（类、函数、属性 其实都是ELement）
    private Elements elements;
    // type的工具类，包含用于操作TypeMirror的工具方法
    private Types types;
    // Message用来打印 日志相关信息
    private Messager messager;
    // 文件生成器， 类 资源 等，就是最终要生成的文件 是需要Filer来完成的
    private Filer filer;


    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);

        elements = processingEnvironment.getElementUtils();
        messager = processingEnvironment.getMessager();
        filer = processingEnvironment.getFiler();

        String value = processingEnvironment.getOptions().get("key");
        messager.printMessage(Diagnostic.Kind.NOTE,"===>"+value);
    }

    // 在编译的时候干活
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {

        messager.printMessage(Diagnostic.Kind.NOTE,"====> run"); // 打印

        // 获取被 ARouter注解的 "类节点信息"
        Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(ARouter.class);

        for (Element element : elements) {

            // 生成方法
            MethodSpec methodSpec = MethodSpec.methodBuilder("main") // 方法名
                    .addModifiers(Modifier.PUBLIC,Modifier.STATIC) // 方法的限定符
                    .returns(void.class) // 方法的返回值
                    .addParameter(String[].class,"args") // 方法参数
                    .addStatement("$T.out.println($S)",System.class,"Hello World!") // 方法内容
                    .build();

            // 生成类
            TypeSpec typeSpec = TypeSpec.classBuilder("HelloWorld ") // 类名
                    .addMethod(methodSpec) // 将方法添加到类中
                    .addModifiers(Modifier.PUBLIC,Modifier.FINAL) // 类的限定符
                    .build();

            // 生成包
            JavaFile packagef = JavaFile.builder("com.kimliu.test",typeSpec).build();

            // 生成文件
            try {
                packagef.writeTo(filer);
            } catch (IOException e) {
                e.printStackTrace();
                messager.printMessage(Diagnostic.Kind.NOTE,"Exception:"+e.getMessage());
            }
        }

        return true; // false 不干了  true 干完了
    }
}

```
**RUNTIME级别的注解**  
注解保留至运行期，意味着我们能够在运行期间结合反射技术获取注解中的所有信息。

**应用场景：反射**  
在程序运行期间，通过反射技术动态获取注解与其元素，从而完成不同的逻辑判定。

## 反射

### 一、 反射的概念
反射就是在运行状态中,对于任意一个类,都能够知道这个类的所有属性和方法;对于任意一个对象,都能够调用它的任意方法和属性;并且能改变它的属性。是Java被视为动态语言的关键。  
Java的反射机制提供了以下功能：
> 在运行时构造任意一个类的对象  
> 在运行时获取或者修改任意一个类所具有的成员变量和方法  
> 在运行时调用任意一个对象的方法（属性）

### 二、 反射之Class
反射始于Class，Class是一个类，封装了当前对象所对应的类的信息。一个类中有属性，方法，构造器等，比如说，有一个Person类，一个Order类，一个Book类，这些都是不同的类，现在需要一个类，用来描述类，这就是Class，它应该有类名，属性，方法，构造器等。Class是用来描述类的类。  

对于每个类而言，JRE 都为其保留一个不变的 Class 类型的对象。一个 Class 对象包含了特定某个类的有关信息。对象只能由系统建立对象，一个类（而不是一个对象）在 JVM 中只会有一个Class实例。

#### 获取Class对象
##### 方法1：通过类名获取
```
Class<?> klass = int.class;
Class<?> classInt = Integer.TYPE;
```
##### 方法2：通过对象获取: 对象名.getClass()
```
StringBuilder str = new StringBuilder("123");
Class<?> klass = str.getClass();
```
##### 方法3：通过全类名获取: 使用Class类的forName静态方法：Class.forName(全类名) classLoader.loadClass(全类名)
```
public static Class<?> forName(StringclassName)
```
#### 判断是否为某个类的实例
```
instanceof / isInstance(Object obj) / isAssignableFrom(Class<?> cls)

public native boolean isInstance(Objectobj);
 
public boolean isAssignableFrom(Class<?>cls);
```

### 三、 通过反射来创建实例
#### 3.1 使用Class对象的newInstance()来创建Class对象对应类的实例
```
Class<?> c = String.class;
Object str = c.newInstance();
```
#### 3.2 先通过Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建实例。这种方法可以用指定的构造器构造类的实例。
```
//获取String所对应的Class对象
Class<?> c = String.class;
//获取String类带一个String参数的构造器
Constructor constructor = c.getConstructor(String.class);
//根据构造器创建实例
Object obj = constructor.newInstance("23333");
System.out.println(obj);
```
##### 获取构造器信息
```
Constructor getConstructor(Class[] params) -> 获得使用特殊的参数类型的public构造函数(包括父类）
Constructor[] getConstructors() -> 获得类的所有公共构造函数
Constructor getDeclaredConstructor(Class[] params) -> 获得使用特定参数类型的构造函数(包括私有)
Constructor[] getDeclaredConstructors() -> 获得类的所有构造函数(与接入级别无关)
```
获取类构造器的用法与上述获取方法的用法类似。主要是通过Class类的getConstructor方法得到Constructor类的一个实例，而Constructor类有一个newInstance方法可以创建一个对象实例
```
public T newInstance(Object ... initargs)
```

### 四、 获取类的成员变量

#### 4.1 字段信息
```
Field getField(String name) -- 获得命名的公共字段
Field[] getFields() -- 获得类的所有公共字段
Field getDeclaredField(String name) -- 获得类声明的命名的字段
Field[] getDeclaredFields() -- 获得类声明的所有字段
```

#### 4.2 调用方法

> 1. 获取方法信息  
> 2. 使用invoke来调用这个方法
```
Method getMethod(String name, Class[] params) -- 使用特定的参数类型，获得命名的公共方法
Method[] getMethods() -- 获得类的所有公共方法
Method getDeclaredMethod(String name, Class[] params) -- 使用特写的参数类型，获得类声明的命名的方法
Method[] getDeclaredMethods() -- 获得类声明的所有方法

public Object invoke(Object obj, Object... args)
```


### 五、 利用反射创建数组
数组在Java里是比较特殊的一种类型，它可以赋值给一个Object Reference 其中的Array类为
java.lang.reflect.Array类。我们通过Array.newInstance()创建数组对象，它的原型是:
```
public static Object newInstance(Class<?> componentType, int length);
```

### 六、 反射获取泛型的真是类型
当我们对一个泛型类进行反射时，需要的到泛型中的真实数据类型，来完成如json反序列化的操作。此时需要通
过 Type 体系来完成。 Type 接口包含了一个实现类(Class)和四个实现接口，他们分别是:
1. TypeVariable 泛型类型变量。可以泛型上下限等信息
2. ParameterizedType 具体的泛型类型，可以获得元数据中泛型签名类型(泛型真实类型)
3. GenericArrayType 当需要描述的类型是泛型类的数组时，比如List[],Map[]，此接口会作为Type的实现。
4. WildcardType 通配符泛型，获得上下限信息

### 七、 TypeVariable
```
public class TestType <K extends Comparable & Serializable, V> {
       K key;
       V value;
    
   public static void main(String[] args) throws Exception {
        // 获取字段的类型
        Field fk = TestType.class.getDeclaredField("key");
        Field fv = TestType.class.getDeclaredField("value");
        TypeVariable keyType = (TypeVariable)fk.getGenericType();
        TypeVariable valueType = (TypeVariable)fv.getGenericType();
        // getName 方法
        System.out.println(keyType.getName()); // K
        System.out.println(valueType.getName()); // V
       // getGenericDeclaration 方法
        System.out.println(keyType.getGenericDeclaration()); // class com.test.TestType
        System.out.println(valueType.getGenericDeclaration()); // class com.test.TestType
       // getBounds 方法
        System.out.println("K 的上界:"); // 有两个
        for (Type type : keyType.getBounds()) { // interface java.lang.Comparable
                 System.out.println(type); // interface java.io.Serializable
              } 
        System.out.println("V 的上界:"); // 没明确声明上界的, 默认上界是 Object
        for (Type type : valueType.getBounds()) { // class java.lang.Object
                 System.out.println(type);
             }
        }
}
```
### 八、ParameterizedType
```
public class TestType {
    
    Map<String, String> map;
    
   public static void main(String[] args) throws Exception {
        Field f = TestType.class.getDeclaredField("map");
        System.out.println(f.getGenericType()); // java.util.Map<java.lang.String,java.lang.String>
        ParameterizedType pType = (ParameterizedType) f.getGenericType();
        System.out.println(pType.getRawType()); // interface java.util.Map
        for (Type type : pType.getActualTypeArguments()) {
            System.out.println(type); // 打印两遍: class java.lang.String
        }
    }
}
```
### 九、GenericArrayType
```
public class TestType<T> {
    
    List<String>[] lists;
    
    public static void main(String[] args) throws Exception {
       Field f = TestType.class.getDeclaredField("lists");
       GenericArrayType genericType = (GenericArrayType) f.getGenericType();
       System.out.println(genericType.getGenericComponentType());
    }
}
```
### 十、WildcardType
```
public class TestType {
    
    private List<? extends Number> a; // 上限
    private List<? super String> b; //下限
       
    public static void main(String[] args) throws Exception {
        Field fieldA = TestType.class.getDeclaredField("a");
        Field fieldB = TestType.class.getDeclaredField("b");
        // 先拿到范型类型
        ParameterizedType pTypeA = (ParameterizedType) fieldA.getGenericType();
        ParameterizedType pTypeB = (ParameterizedType) fieldB.getGenericType();
        // 再从范型里拿到通配符类型
        WildcardType wTypeA = (WildcardType) pTypeA.getActualTypeArguments()[0];
        WildcardType wTypeB = (WildcardType) pTypeB.getActualTypeArguments()[0];
        // 方法测试
        System.out.println(wTypeA.getUpperBounds()[0]); // class java.lang.Number
        System.out.println(wTypeB.getLowerBounds()[0]); // class java.lang.String
        // 看看通配符类型到底是什么, 打印结果为: ? extends java.lang.Number
        System.out.println(wTypeA);
    }
}
```

### 十一、Gson反序列化
```
static class Response<T> {
    
    T data;
    int code;
    String message;

    @Override
    public String toString() {
        return "Response{" +
               "data=" + data +
               ", code=" + code +
               ", message='" + message + '\'' +
               '}';
    } 

    public Response(T data, int code, String message) {
        this.data = data;
        this.code = code;
        this.message = message;
    }
} 
static class Data {
    
    String result;
    
    public Data(String result) {
        this.result = result;
    }
    @Override
    public String toString() {
        return "Data{" +
               "result=" + result +
               '}';
    }
} 
    public static void main(String[] args) {
        Response<Data> dataResponse = new Response(new Data("数据"), 1, "成功");
        Gson gson = new Gson();
        String json = gson.toJson(dataResponse);
        System.out.println(json);

        //为什么TypeToken要定义为抽象类？
        Response<Data> resp = gson.fromJson(json, new TypeToken<Response<Data>>() {
        }.getType());
        System.out.println(resp.data.result);
    }
```
在进行GSON反序列化时，存在泛型时，可以借助 TypeToken 获取Type以完成泛型的反序列化。但是为什么
TypeToken 要被定义为抽象类呢？
因为只有定义为抽象类或者接口，这样在使用时，需要创建对应的实现类，此时确定泛型类型，编译才能够将泛型
signature信息记录到Class元数据中。

### 十二、反射的应用场景之运行时注解
通过注解和反射，实现findViewById  
InjectView：
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface InjectView {
    @IdRes int value();
}
```
InjectUtils：
```
public class InjectUtils {


    // 使用反射，获取属性的值，并给filed赋值
    public static void injectView(Activity activity) {
        Class<? extends Activity> cls = activity.getClass();

        //获得此类所有的成员
        Field[] declaredFields = cls.getDeclaredFields();
        for (Field filed : declaredFields) {
            // 判断属性是否被InjectView注解声明
            if (filed.isAnnotationPresent(InjectView.class)){
                InjectView injectView = filed.getAnnotation(InjectView.class);
                //获得了注解中设置的id
                int id = injectView.value();
                View view = activity.findViewById(id);
                //反射设置 属性的值
                filed.setAccessible(true); //设置访问权限，允许操作private的属性
                try {
                    //反射赋值 第一个参数 给哪个对象的属性赋值 如果类是static的 直接传null
                    filed.set(activity,view);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```
MainActivity：
```
public class MainActivity extends AppCompatActivity {
   
    @InjectView(R.id.tv)
    private TextView tv;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        InjectUtils.injectView(this);
        tv.setText("HelloWorld");
    }
    
}
```

## 使用反射和注解，手撸一个Retrofit框架
要搞懂Retrofit，必然要先搞懂动态代理。
### 动态代理 
（节选自专栏：设计模式之美第48讲代理模式）
#### 代理模式
在不改变原始类代码的情况下，通过引入代理类，来给原始类附加功能。  
通过例子才能更好的理解概念，引入下面这样一个例子：
在登录的过程中，我们使用一个叫做MetricsCollector的类来进行接口请求次数的收集。
```
public class UserController {
  //...省略其他属性和方法...
  private MetricsCollector metricsCollector; // 依赖注入

  public UserVo login(String telephone, String password) {
    long startTimestamp = System.currentTimeMillis();

    // ... 省略login逻辑...

    long endTimeStamp = System.currentTimeMillis();
    long responseTime = endTimeStamp - startTimestamp;
    RequestInfo requestInfo = new RequestInfo("login", responseTime, startTimestamp);
    metricsCollector.recordRequest(requestInfo);

    //...返回UserVo数据...
  }

  public UserVo register(String telephone, String password) {
    long startTimestamp = System.currentTimeMillis();

    // ... 省略register逻辑...

    long endTimeStamp = System.currentTimeMillis();
    long responseTime = endTimeStamp - startTimestamp;
    RequestInfo requestInfo = new RequestInfo("register", responseTime, startTimestamp);
    metricsCollector.recordRequest(requestInfo);

    //...返回UserVo数据...
  }
}
```
很明显，上面的写法有两个问题。第一，性能计数器框架代码侵入到业务代码中，跟业务代码高度耦合。如果未来需要替换这个框架，那替换的成本会比较大。第二，收集接口请求的代码跟业务代码无关，本就不应该放到一个类中。业务类最好职责更加单一，只聚焦业务处理。

为了将框架代码和业务代码解耦，代理模式就派上用场了。代理类 UserControllerProxy 和原始类 UserController 实现相同的接口 IUserController。UserController 类只负责业务功能。代理类 UserControllerProxy 负责在业务代码执行前后附加其他逻辑代码，并通过委托的方式调用原始类来执行业务代码。具体的代码实现如下所示：  

IUserController 接口：

```

public interface IUserController {
  UserVo login(String telephone, String password);
  UserVo register(String telephone, String password);
}
```
UserController 类 ：

```
public class UserController implements IUserController {
  //...省略其他属性和方法...

  @Override
  public UserVo login(String telephone, String password) {
    //...省略login逻辑...
    //...返回UserVo数据...
  }

  @Override
  public UserVo register(String telephone, String password) {
    //...省略register逻辑...
    //...返回UserVo数据...
  }
}
```

UserControllerProxy 代理类：
```

public class UserControllerProxy implements IUserController {
  private MetricsCollector metricsCollector;
  private UserController userController;

  public UserControllerProxy(UserController userController) {
    this.userController = userController;
    this.metricsCollector = new MetricsCollector();
  }

  @Override
  public UserVo login(String telephone, String password) {
    long startTimestamp = System.currentTimeMillis();

    // 委托
    UserVo userVo = userController.login(telephone, password);

    long endTimeStamp = System.currentTimeMillis();
    long responseTime = endTimeStamp - startTimestamp;
    RequestInfo requestInfo = new RequestInfo("login", responseTime, startTimestamp);
    metricsCollector.recordRequest(requestInfo);

    return userVo;
  }

  @Override
  public UserVo register(String telephone, String password) {
    long startTimestamp = System.currentTimeMillis();

    UserVo userVo = userController.register(telephone, password);

    long endTimeStamp = System.currentTimeMillis();
    long responseTime = endTimeStamp - startTimestamp;
    RequestInfo requestInfo = new RequestInfo("register", responseTime, startTimestamp);
    metricsCollector.recordRequest(requestInfo);

    return userVo;
  }
}

//UserControllerProxy使用举例
//因为原始类和代理类实现相同的接口，是基于接口而非实现编程
//将UserController类对象替换为UserControllerProxy类对象，不需要改动太多代码
IUserController userController = new UserControllerProxy(new UserController());
```

参照基于接口而非实现编程的设计思想，将原始类对象替换为代理类对象的时候，为了让代码改动尽量少，在刚刚的代理模式的代码实现中，代理类和原始类需要实现相同的接口。但是，如果原始类并没有定义接口，并且原始类代码并不是我们开发维护的（比如它来自一个第三方的类库），我们也没办法直接修改原始类，给它重新定义一个接口。在这种情况下，我们该如何实现代理模式呢？

对于这种外部类的扩展，我们一般都是采用继承的方式。这里也不例外。我们让代理类继承原始类，然后扩展附加功能。原理很简单，不需要过多解释，你直接看代码就能明白。具体代码如下所示：

```

public class UserControllerProxy extends UserController {
  private MetricsCollector metricsCollector;

  public UserControllerProxy() {
    this.metricsCollector = new MetricsCollector();
  }

  public UserVo login(String telephone, String password) {
    long startTimestamp = System.currentTimeMillis();

    UserVo userVo = super.login(telephone, password);

    long endTimeStamp = System.currentTimeMillis();
    long responseTime = endTimeStamp - startTimestamp;
    RequestInfo requestInfo = new RequestInfo("login", responseTime, startTimestamp);
    metricsCollector.recordRequest(requestInfo);

    return userVo;
  }

  public UserVo register(String telephone, String password) {
    long startTimestamp = System.currentTimeMillis();

    UserVo userVo = super.register(telephone, password);

    long endTimeStamp = System.currentTimeMillis();
    long responseTime = endTimeStamp - startTimestamp;
    RequestInfo requestInfo = new RequestInfo("register", responseTime, startTimestamp);
    metricsCollector.recordRequest(requestInfo);

    return userVo;
  }
}
//UserControllerProxy使用举例
UserController userController = new UserControllerProxy();
```

#### 动态代理
不过，刚刚的代码实现还是有点问题。一方面，我们需要在代理类中，将原始类中的所有的方法，都重新实现一遍，并且为每个方法都附加相似的代码逻辑。另一方面，如果要添加的附加功能的类有不止一个，我们需要针对每个类都创建一个代理类。   

如果有 50 个要添加附加功能的原始类，那我们就要创建 50 个对应的代理类。这会导致项目中类的个数成倍增加，增加了代码维护成本。并且，每个代理类中的代码都有点像模板式的“重复”代码，也增加了不必要的开发成本。那这个问题怎么解决呢？    

我们可以使用动态代理来解决这个问题。所谓动态代理（Dynamic Proxy），就是我们不事先为每个原始类编写代理类，而是在运行的时候，动态地创建原始类对应的代理类，然后在系统中用代理类替换掉原始类。那如何实现动态代理呢？   

如果你熟悉的是 Java 语言，实现动态代理就是件很简单的事情。因为 Java 语言本身就已经提供了动态代理的语法（实际上，动态代理底层依赖的就是 Java 的反射语法）。我们来看一下，如何用 Java 的动态代理来实现刚刚的功能。具体的代码如下所示。其中，MetricsCollectorProxy 作为一个动态代理类，动态地给每个需要收集接口请求信息的类创建代理类。

```

public class MetricsCollectorProxy {
  private MetricsCollector metricsCollector;

  public MetricsCollectorProxy() {
    this.metricsCollector = new MetricsCollector();
  }

  public Object createProxy(Object proxiedObject) {
    Class<?>[] interfaces = proxiedObject.getClass().getInterfaces();
    DynamicProxyHandler handler = new DynamicProxyHandler(proxiedObject);
    return Proxy.newProxyInstance(proxiedObject.getClass().getClassLoader(), interfaces, handler);
  }

  private class DynamicProxyHandler implements InvocationHandler {
    private Object proxiedObject;

    public DynamicProxyHandler(Object proxiedObject) {
      this.proxiedObject = proxiedObject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      long startTimestamp = System.currentTimeMillis();
      Object result = method.invoke(proxiedObject, args);
      long endTimeStamp = System.currentTimeMillis();
      long responseTime = endTimeStamp - startTimestamp;
      String apiName = proxiedObject.getClass().getName() + ":" + method.getName();
      RequestInfo requestInfo = new RequestInfo(apiName, responseTime, startTimestamp);
      metricsCollector.recordRequest(requestInfo);
      return result;
    }
  }
}

//MetricsCollectorProxy使用举例
MetricsCollectorProxy proxy = new MetricsCollectorProxy();
IUserController userController = (IUserController) proxy.createProxy(new UserController());
```
了解了动态代理，现在就来创建一个简单的Retrofit框架吧，这里的框架只包括了，使用注解、反射、动态代理封装OKhttp和请求网络的过程。

我们来回忆一下，使用Retrofit进行网络请求的步骤是什么？
> 1. 通过Builder模式构建Retrofit客户端
> 2. 通过动态代理创建ServiceApi
> 3. 调用ServiceApi中的方法请求网络   

差不多就是这样的三步，通过代码体现出来就是：
```
// 1. 通过Builder模式构建Retrofit客户端
  Retrofit retrofit = new Retrofit.Builder().baseUrl("https://restapi.amap.com")
               .build();
// 2. 通过动态代理创建ServiceApi
  WeatherApi weatherApi = retrofit.create(WeatherApi.class);
// 3. 调用ServiceApi中的方法请求网络   
  // GET 方式
  Call getCall = weatherApi.getWeather("110101", "ae6c53e2186f33bbf240a12d80672d1b");
  getCall.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //... 处理 请求失败的情况
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                // ....处理 请求成功的情况
            }
        });
        
    // POST方式
    Call postCall = weatherApi.postWeather("110101", "ae6c53e2186f33bbf240a12d80672d1b");
    postCall.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
            }
        });
```

#### 一、创建DemoRetrofit客户端
注意使用Builder模式
```
/**
 * Retrofit 客户端
 *
 * 使用动态代理创建Retrofit客户端
 */
public class DemoRetrofit {

    final Call.Factory mFactory;
    final HttpUrl mHttpUrl;
    final Map<Method, ServiceMethod> serviceMethodCache = new ConcurrentHashMap<>();

    public DemoRetrofit(Call.Factory factory,HttpUrl httpUrl) {
        this.mFactory = factory;
        this.mHttpUrl = httpUrl;
    }

    /**
     *  使用动态代理创建 SeviceMethod
     * @param service
     * @param <T>
     * @return
     */
    public <T> T create(final Class<T> service){
        return (T) Proxy.newProxyInstance(service.getClassLoader(),new Class[]{service},new InvocationHandler(){

            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                ServiceMethod serviceMethod = loadServiceMethod(method);
                // 这个args是调用serviceMethod中的方法时，传入的参数
                return serviceMethod.invoke(args);
            }
        });
    }



    private ServiceMethod loadServiceMethod(Method method){
        // 从缓存中获取 ServiceMethod
        ServiceMethod serviceMethod = serviceMethodCache.get(method);
        if(serviceMethod != null){
            return serviceMethod;
        }

        synchronized (serviceMethodCache){
           serviceMethod = serviceMethodCache.get(method);
           if(serviceMethod == null){
               // 使用Builder方式 构建ServiceMethod
               serviceMethod = new ServiceMethod.Builder(this,method).bulid();
               serviceMethodCache.put(method,serviceMethod);
           }
        }
        return serviceMethod;
    }


    public static final class Builder{
        private HttpUrl httpUrl;
        private Call.Factory factory;

        public Builder callFactory(Call.Factory factory) {
            this.factory = factory;
            return this;
        }

        public Builder baseUrl(String baseUrl){
           this.httpUrl = HttpUrl.get(baseUrl);
           return this;
        }

        public DemoRetrofit build(){
            if(httpUrl == null){
                throw new IllegalStateException("Base Url Required");
            }
            Call.Factory factory = this.factory;
            if(factory == null){
                factory = new OkHttpClient();
            }
            return new DemoRetrofit(factory,httpUrl);
        }
    }
}

```

#### 二、创建ServiceMethod
这个类是Retrofit的精髓所在，在这里将网络请求抽象成了一个ServiceMethod，在这个类的对象构造的时候，通过Java反射的方式传入一个Method对象，而这个对象，就是在我们定义的interface中的请求方法。在这个类中，通过反射，获取到了方法上的注解内容path、方法参数的注解内容key，并将这些数据记录下来，同时完成网络请求的操作。
```
/**
 * 记录请求的类型，参数，完整地址
 */
public class ServiceMethod {

    private final String mHttpMethod;
    private final Call.Factory mFactory;
    private final HttpUrl mHttpUrl;
    private final String mRelativeUrl;
    private final ParameterHandler[] mParameterHandler;
    private final boolean mIsHaveBody;
    private FormBody.Builder mFormBuilder;
    private HttpUrl.Builder mHttpUrlBuilder;


    public ServiceMethod(Builder builder) {
        mHttpMethod = builder.mHttpMethod;
        mFactory = builder.retrofit.mFactory;
        mHttpUrl = builder.retrofit.mHttpUrl;
        mRelativeUrl = builder.mRelativeUrl;
        mParameterHandler = builder.mParameterHandler;
        mIsHaveBody = builder.mIsHaveBody;

        if(mIsHaveBody){
            mFormBuilder = new FormBody.Builder();
        }
    }


    /**
     *
     * @param args 调用ServiceMethod中的方法时，传入的参数
     * @return
     */
    public Object invoke(Object[] args){
        /**
         * 1. 处理请求的地址与参数
         */
        for (int i = 0; i < mParameterHandler.length; i++) {
            ParameterHandler parameterHandler = mParameterHandler[i];
            // 将parameterHandler中的key值 给到对应的value
            parameterHandler.apply(this,args[i].toString());
        }

        // 获取最终的请求地址
        HttpUrl httpUrl;
        if(mHttpUrlBuilder == null){
            mHttpUrlBuilder = mHttpUrl.newBuilder(mRelativeUrl);
        }

        httpUrl = mHttpUrlBuilder.build();

        //请求体
        FormBody formBody = null;
        if(mFormBuilder != null){
            formBody = mFormBuilder.build();
        }

        // 请求网络
        Request request = new Request.Builder().url(httpUrl).method(mHttpMethod,formBody).build();
        return mFactory.newCall(request);

    }


    // get请求 把k-v拼接到url中
    public void addQueryParameter(String key,String value){
        if(mHttpUrlBuilder == null){
            mHttpUrlBuilder = mHttpUrl.newBuilder(mRelativeUrl);
        }
        mHttpUrlBuilder.addQueryParameter(key,value);
    }

    // POST请求，把k-v放到请求体中
    public void addFieldParameter(String key,String value){
        mFormBuilder.add(key,value);
    }


    public static class Builder{

        private DemoRetrofit retrofit;
        private final Annotation[] mMethodAnnotations;
        private final Annotation[][] mParameterAnnotations;
        String mHttpMethod;
        String mRelativeUrl;
        boolean mIsHaveBody;
        private ParameterHandler[] mParameterHandler;

        public Builder(DemoRetrofit retrofit, Method method) {
            this.retrofit = retrofit;
            mMethodAnnotations = method.getAnnotations(); // 获取方法上的所有注释
            mParameterAnnotations = method.getParameterAnnotations(); //获取方法参数上的所有注释（一个参数可以有多个注解,一个方法又会有多个参数）
        }

        /**
         * 在这里获取 注释中的内容
         * @return
         */
        public ServiceMethod bulid(){
            /**
             * 1. 解析方法上的注解，处理POST和GET
             */
            for (Annotation methodAnnotation : mMethodAnnotations) {
                if(methodAnnotation instanceof POST){
                    // 如果是POST类型的注解
                    this.mHttpMethod = "POST";
                    // 记录请求URL的path
                    this.mRelativeUrl = ((POST) methodAnnotation).value();
                    // 是否有请求体
                    this.mIsHaveBody = true;
                }else if(methodAnnotation instanceof GET){
                    this.mHttpMethod ="GET";
                    this.mRelativeUrl = ((GET) methodAnnotation).value();
                    this.mIsHaveBody = false;
                }
            }

            /**
             * 2. 解析方法参数的注解
             */
            int length = mParameterAnnotations.length;
            mParameterHandler = new ParameterHandler[length];
            for (int i = 0; i < mParameterAnnotations.length; i++) {
                Annotation[] parameterAnnotation = mParameterAnnotations[i];
                for (Annotation annotation : parameterAnnotation) {
                    // 添加一个判断 POST只能使用Field注解 GET只能使用Query注解
                    if(annotation instanceof Field){
                        if(this.mHttpMethod.equals("GET")){
                            throw new RuntimeException("@Field 只可用于POST请求中");
                        }
                        String key = ((Field) annotation).value();// 获取到Field的key值
                        // 获取Key值 放入ParameterHandler中
                        mParameterHandler[i] = new ParameterHandler.FieldParameterHandler(key);
                    }else if(annotation instanceof Query){
                        if(this.mHttpMethod.equals("POST")){
                            throw new RuntimeException("@Query 只可用于GET请求中");
                        }
                        String key = ((Query) annotation).value();
                        // 获取Key值 放入ParameterHandler中
                        mParameterHandler[i] = new ParameterHandler.QueryParameterHandler(key);
                    }
                }
            }
            return new ServiceMethod(this);
        }

    }
}
```
#### 三、创建ParameterHandler
这个类的作用是记录从ServiceMethod中记录下来的参数对应请求中的name
```
/**
 * 记录参数对应请求中的name
 */
public abstract class ParameterHandler {

    abstract void apply(ServiceMethod serviceMethod,String value);


    static class QueryParameterHandler extends ParameterHandler{

        String key;
        public QueryParameterHandler(String key) {
            this.key = key;
        }

        @Override
        void apply(ServiceMethod serviceMethod, String value) {
            serviceMethod.addQueryParameter(key,value);
        }
    }

    static class FieldParameterHandler extends ParameterHandler{

        String key;
        public FieldParameterHandler(String key) {
            this.key = key;
        }

        @Override
        void apply(ServiceMethod serviceMethod, String value) {
            serviceMethod.addFieldParameter(key,value);
        }
    }

}

```

#### 四、创建注解
```
/**
 * 注解在POST方法上，用来： 通过反射 获取到注释里的Value值 也就是path
 */
@Target(ElementType.METHOD) // 作用在方法上
@Retention(RetentionPolicy.RUNTIME) // 运行时注解
public @interface POST {

    String value() default "";

}
```

```
/**
 * 注解在POST方法上，用来： 通过反射 获取到注释里的Value值 也就是path
 */
@Target(ElementType.METHOD) // 作用在方法上
@Retention(RetentionPolicy.RUNTIME) // 运行时注解
public @interface POST {

    String value() default "";

}

```

```
/**
 *  POST请求中的 参数注解
 *  用来：通过反射 获取到 key 值
 */
@Target(ElementType.PARAMETER) // 作用于参数中
@Retention(RetentionPolicy.RUNTIME) // 运行时注解
public @interface Field {

    String value();

}

```

```
/**
 * GET 请求中 参数的注释，用于：使用反射 获取到请求参数的key值
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Query {
    String value();
}

```

#### 五、创建ServiceApi接口
```
public interface WeatherApi {

    @POST("/v3/weather/weatherInfo")
    Call postWeather(@Field("city") String city, @Field("key") String key);


    @GET("/v3/weather/weatherInfo")
    Call getWeather(@Query("city") String city, @Query("key") String key);



}
```

#### 六、使用DemoRetrofit请求网络
```
 // 使用Builder方式 构建 DemoRetrofit 客户端
        DemoRetrofit retrofit = new DemoRetrofit.Builder().baseUrl("https://restapi.amap.com").build();
        WeatherApi weatherApi = retrofit.create(WeatherApi.class);
        // GET 方式
        Call getCall = weatherApi.getWeather("110101", "ae6c53e2186f33bbf240a12d80672d1b");
        getCall.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //... 处理 请求失败的情况
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                // ....处理 请求成功的情况
            }
        });

        // POST方式
        Call postCall = weatherApi.postWeather("110101", "ae6c53e2186f33bbf240a12d80672d1b");
        postCall.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //... 处理 请求失败的情况
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                // ....处理 请求成功的情况
            }
        });
```

代码路径：https://github.com/kimlllll/RetrofitBasicDemo

