# 设计模式

对修改关闭，对扩展开放

## 一.创建型模式

### 1. 工厂方法模式

产生对象的类和方法都可以叫做工厂，目的是灵活控制生产过程，例如`产品维度扩展`

### 2. 抽象工厂模式

形容词用接口，名词用抽象类，例如`产品一族扩展`,`一键换肤`

### 3. 建造者模式


### 4. 原型模式（多套试题答案乱序）

定义：用一个已经创建的实例作为原型，通过复制该原型对象来创建一个和原型相同或相似的新对象

通过克隆方式创建复杂对象、也可以避免重复做初始化操作、不需要与类中所属的其他类耦合等。但也有一些缺点如果对象中包括了循环引用的克隆，以及类中深度使用对象的克隆，增加复杂度

解答题对象
```java
package org.itstack.demo.design;

/**
 * 解答题
 */
public class AnswerQuestion {

    private String name;  // 问题
    private String key;   // 答案

    public AnswerQuestion() {
    }

    public AnswerQuestion(String name, String key) {
        this.name = name;
        this.key = key;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }
}
```

选择题对象
```java
package org.itstack.demo.design;

import java.util.Map;

/**
 * 单选题
 */
public class ChoiceQuestion {

    private String name;                 // 题目
    private Map<String, String> option;  // 选项；A、B、C、D
    private String key;                  // 答案；B

    public ChoiceQuestion() {
    }

    public ChoiceQuestion(String name, Map<String, String> option, String key) {
        this.name = name;
        this.option = option;
        this.key = key;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Map<String, String> getOption() {
        return option;
    }

    public void setOption(Map<String, String> option) {
        this.option = option;
    }

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }
}
```

题库对象
```java
package org.itstack.demo.design;

import org.itstack.demo.design.util.Topic;
import org.itstack.demo.design.util.TopicRandomUtil;

import java.util.ArrayList;
import java.util.Collections;
import java.util.Map;

public class QuestionBank implements Cloneable {

    private String candidate; // 考生
    private String number;    // 考号

    private ArrayList<ChoiceQuestion> choiceQuestionList = new ArrayList<ChoiceQuestion>();
    private ArrayList<AnswerQuestion> answerQuestionList = new ArrayList<AnswerQuestion>();

    public QuestionBank append(ChoiceQuestion choiceQuestion) {
        choiceQuestionList.add(choiceQuestion);
        return this;
    }

    public QuestionBank append(AnswerQuestion answerQuestion) {
        answerQuestionList.add(answerQuestion);
        return this;
    }

    @Override
    public Object clone() throws CloneNotSupportedException {
        QuestionBank questionBank = (QuestionBank) super.clone();
        questionBank.choiceQuestionList = (ArrayList<ChoiceQuestion>) choiceQuestionList.clone();
        questionBank.answerQuestionList = (ArrayList<AnswerQuestion>) answerQuestionList.clone();

        // 题目乱序
        Collections.shuffle(questionBank.choiceQuestionList);
        Collections.shuffle(questionBank.answerQuestionList);
        // 答案乱序
        ArrayList<ChoiceQuestion> choiceQuestionList = questionBank.choiceQuestionList;
        for (ChoiceQuestion question : choiceQuestionList) {
            Topic random = TopicRandomUtil.random(question.getOption(), question.getKey());
            question.setOption(random.getOption());
            question.setKey(random.getKey());
        }
        return questionBank;
    }

    public void setCandidate(String candidate) {
        this.candidate = candidate;
    }

    public void setNumber(String number) {
        this.number = number;
    }

    @Override
    public String toString() {

        StringBuilder detail = new StringBuilder("考生：" + candidate + "\r\n" +
                "考号：" + number + "\r\n" +
                "--------------------------------------------\r\n" +
                "一、选择题" + "\r\n\n");

        for (int idx = 0; idx < choiceQuestionList.size(); idx++) {
            detail.append("第").append(idx + 1).append("题：").append(choiceQuestionList.get(idx).getName()).append("\r\n");
            Map<String, String> option = choiceQuestionList.get(idx).getOption();
            for (String key : option.keySet()) {
                detail.append(key).append("：").append(option.get(key)).append("\r\n");;
            }
            detail.append("答案：").append(choiceQuestionList.get(idx).getKey()).append("\r\n\n");
        }

        detail.append("二、问答题" + "\r\n\n");

        for (int idx = 0; idx < answerQuestionList.size(); idx++) {
            detail.append("第").append(idx + 1).append("题：").append(answerQuestionList.get(idx).getName()).append("\r\n");
            detail.append("答案：").append(answerQuestionList.get(idx).getKey()).append("\r\n\n");
        }

        return detail.toString();
    }

}
```

初始化题库
```java
package org.itstack.demo.design;

import java.util.HashMap;
import java.util.Map;

public class QuestionBankController {

    private QuestionBank questionBank = new QuestionBank();

    public QuestionBankController() {

        Map<String, String> map01 = new HashMap<String, String>();
        map01.put("A", "JAVA2 EE");
        map01.put("B", "JAVA2 Card");
        map01.put("C", "JAVA2 ME");
        map01.put("D", "JAVA2 HE");
        map01.put("E", "JAVA2 SE");

        Map<String, String> map02 = new HashMap<String, String>();
        map02.put("A", "JAVA程序的main方法必须写在类里面");
        map02.put("B", "JAVA程序中可以有多个main方法");
        map02.put("C", "JAVA程序中类名必须与文件名一样");
        map02.put("D", "JAVA程序的main方法中如果只有一条语句，可以不用{}(大括号)括起来");

        Map<String, String> map03 = new HashMap<String, String>();
        map03.put("A", "变量由字母、下划线、数字、$符号随意组成；");
        map03.put("B", "变量不能以数字作为开头；");
        map03.put("C", "A和a在java中是同一个变量；");
        map03.put("D", "不同类型的变量，可以起相同的名字；");

        Map<String, String> map04 = new HashMap<String, String>();
        map04.put("A", "STRING");
        map04.put("B", "x3x;");
        map04.put("C", "void");
        map04.put("D", "de$f");

        Map<String, String> map05 = new HashMap<String, String>();
        map05.put("A", "31");
        map05.put("B", "0");
        map05.put("C", "1");
        map05.put("D", "2");

        questionBank.append(new ChoiceQuestion("JAVA所定义的版本中不包括", map01, "D"))
                .append(new ChoiceQuestion("下列说法正确的是", map02, "A"))
                .append(new ChoiceQuestion("变量命名规范说法正确的是", map03, "B"))
                .append(new ChoiceQuestion("以下()不是合法的标识符",map04, "C"))
                .append(new ChoiceQuestion("表达式(11+3*8)/4%3的值是", map05, "D"))
                .append(new AnswerQuestion("小红马和小黑马生的小马几条腿", "4条腿"))
                .append(new AnswerQuestion("铁棒打头疼还是木棒打头疼", "头最疼"))
                .append(new AnswerQuestion("什么床不能睡觉", "牙床"))
                .append(new AnswerQuestion("为什么好马不吃回头草", "后面的草没了"));
    }

    public String createPaper(String candidate, String number) throws CloneNotSupportedException {
        QuestionBank questionBankClone = (QuestionBank) questionBank.clone();
        questionBankClone.setCandidate(candidate);
        questionBankClone.setNumber(number);
        return questionBankClone.toString();
    }

}
```

乱序对象
```java
package org.itstack.demo.design.util;

import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import java.util.Map;

public class Topic {

    private Map<String, String> option;  // 选项；A、B、C、D
    private String key;           // 答案；B

    public Topic() {
    }

    public Topic(Map<String, String> option, String key) {
        this.option = option;
        this.key = key;
    }

    public Map<String, String> getOption() {
        return option;
    }

    public void setOption(Map<String, String> option) {
        this.option = option;
    }

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }
}

package org.itstack.demo.design.util;

import java.util.*;

public class TopicRandomUtil {

    /**
     * 乱序Map元素，记录对应答案key
     * @param option 题目
     * @param key    答案
     * @return Topic 乱序后 {A=c., B=d., C=a., D=b.}
     */
    static public Topic random(Map<String, String> option, String key) {
        Set<String> keySet = option.keySet();
        ArrayList<String> keyList = new ArrayList<String>(keySet);
        Collections.shuffle(keyList);
        HashMap<String, String> optionNew = new HashMap<String, String>();
        int idx = 0;
        String keyNew = "";
        for (String next : keySet) {
            String randomKey = keyList.get(idx++);
            if (key.equals(next)) {
                keyNew = randomKey;
            }
            optionNew.put(randomKey, option.get(next));
        }
        return new Topic(optionNew, keyNew);
    }

}

```

测试
```java
    @Test
	public void test_QuestionBank() throws CloneNotSupportedException {
		QuestionBankController questionBankController = new QuestionBankController();
		System.out.println(questionBankController.createPaper("花花", "1000001921032"));
		System.out.println(questionBankController.createPaper("豆豆", "1000001921051"));
		System.out.println(questionBankController.createPaper("大宝", "1000001921987"));
	}
```

### 5. 单例模式

1. 懒汉模式(线程不安全)
   
```java
public class Singleton_01 {

    private static Singleton_01 instance;

    private Singleton_01() {
    }

    public static Singleton_01 getInstance(){
        if (null != instance) return instance;
        instance = new Singleton_01();
        return instance;
    }
}
```
- 单例模式有一个特点就是不允许外部直接创建，也就是new Singleton_01()，因此这里在默认的构造函数上添加了私有属性 private。
- 目前此种方式的单例确实满足了懒加载，但是如果有多个访问者同时去获取对象实例你可以想象成一堆人在抢厕所，就会造成多个同样的实例并存，从而没有达到单例的要求。

2. 懒汉模式(线程安全)

```java
public class Singleton_02 {

    private static Singleton_02 instance;

    private Singleton_02() {
    }

    public static synchronized Singleton_02 getInstance(){
        if (null != instance) return instance;
        instance = new Singleton_02();
        return instance;
    }
}
```
- 此种模式虽然是安全的，但由于把锁加到方法上后，所有的访问都因需要锁占用导致资源的浪费。如果不是特殊情况下，不建议此种方式实现单例模式

3. 饿汉模式(线程安全)

```java
public class Singleton_03 {

    private static Singleton_03 instance = new Singleton_03();

    private Singleton_03() {
    }

    public static Singleton_03 getInstance() {
        return instance;
    }

}
```
- 此种方式与我们开头的第一个实例化Map基本一致，在程序启动的时候直接运行加载，后续有外部需要使用的时候获取即可。
- 但此种方式并不是懒加载，也就是说无论你程序中是否用到这样的类都会在程序启动之初进行创建。
- 那么这种方式导致的问题就像你下载个游戏软件，可能你游戏地图还没有打开呢，但是程序已经将这些地图全部实例化。到你手机上最明显体验就一开游戏内存满了，手机卡了，需要换了

4. 使用类的内部类(线程安全)

```java
public class Singleton_04 {

    private static class SingletonHolder {
        private static Singleton_04 instance = new Singleton_04();
    }

    private Singleton_04() {
    }

    public static Singleton_04 getInstance() {
        return SingletonHolder.instance;
    }

}
```
- 使用类的静态内部类实现的单例模式，既保证了线程安全有保证了懒加载，同时不会因为加锁的方式耗费性能。
- 这主要是因为JVM虚拟机可以保证多线程并发访问的正确性，也就是一个类的构造方法在多线程环境下可以被正确的加载。
- 此种方式也是非常推荐使用的一种单例模式

5. 双重锁校验(线程安全)
   
```java
public class Singleton_05 {

    private static volatile Singleton_05 instance;

    private Singleton_05() {
    }

    public static Singleton_05 getInstance(){
       if(null != instance) return instance;
       synchronized (Singleton_05.class){
           if (null == instance){
               instance = new Singleton_05();
           }
       }
       return instance;
    }

}
```
- 双重锁的方式是方法级锁的优化，减少了部分获取实例的耗时。
- 同时这种方式也满足了懒加载。

6. CAS「AtomicReference」(线程安全)

```java
public class Singleton_06 {

    private static final AtomicReference<Singleton_06> INSTANCE = new AtomicReference<Singleton_06>();

    private static Singleton_06 instance;

    private Singleton_06() {
    }

    public static final Singleton_06 getInstance() {
        for (; ; ) {
            Singleton_06 instance = INSTANCE.get();
            if (null != instance) return instance;
            INSTANCE.compareAndSet(null, new Singleton_06());
            return INSTANCE.get();
        }
    }

    public static void main(String[] args) {
        System.out.println(Singleton_06.getInstance()); // org.itstack.demo.design.Singleton_06@2b193f2d
        System.out.println(Singleton_06.getInstance()); // org.itstack.demo.design.Singleton_06@2b193f2d
    }

}
```
- java并发库提供了很多原子类来支持并发访问的数据安全性；AtomicInteger、AtomicBoolean、AtomicLong、AtomicReference。
- AtomicReference 可以封装引用一个V实例，支持并发访问如上的单例方式就是使用了这样的一个特点。
- 使用CAS的好处就是不需要使用传统的加锁方式保证线程安全，而是依赖于CAS的忙等算法，依赖于底层硬件的实现，来保证线程安全。相对于其他锁的实现没有线程的切换和阻塞也就没有了额外的开销，并且可以支持较大的并发性。
- 缺点如果一直没有获取到将会处于死循环中。

7. 枚举单例(线程安全)
```java
public class MyObject {
    private static final String driver = "com.mysql.cj.jdbc.Driver";// mysql驱动
    // private static final String driver = "oracle.jdbc.driver.OracleDriver"; // oracle驱动
    // private static final String driver = "org.postgresql.Driver"; // postgresql驱动
    // private static final String driver = "com.microsoft.sqlserver.jdbc.SQLServerDriver"; // sqlserver驱动

    private static final String url = "jdbc:mysql://debug-registry:3306/mall";// mysql地址
    // private static final String url = "jdbc:oracle:thin:@debug-registry:1521:ORCL"; // oracle地址
    // private static final String url = "jdbc:postgresql://debug-registry:5432/VJSP10003182"; // postgresql地址
    // private static final String url = "jdbc:sqlserver://debug-registry:1433; DatabaseName=maptest"; // sqlserver地址

    public enum JdbcUtilSingletion {
        connectionFactory;

        private Connection connection;

        private JdbcUtilSingletion() {
            try {
                String username = "root";
                String password = "root";
                Class.forName(driver);
                connection = DriverManager.getConnection(url, username, password);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        public Connection getConnection() {
            return connection;
        }
    }

    public static Connection getConnection() {
        return JdbcUtilSingletion.connectionFactory.getConnection();
    }

    public static void main(String[] args) {
        MyThread t1 = new MyThread();
        MyThread t2 = new MyThread();
        MyThread t3 = new MyThread();
        t1.start();
        t2.start();
        t3.start();
    }

    public static void main2(String[] args) throws SQLException {

        // 首先需要获取实例,然后获取连接
        Connection conn = MyObject.getConnection();
        // 创建语句
        Statement st = conn.createStatement();
        // 执行语句
        ResultSet rs = st.executeQuery("select * from ums_admin");
        // 处理结果集
        while (rs.next()) {
            System.out
                .println(rs.getObject(1) + "\t" + rs.getObject(2) + "\t" + rs.getObject(3) + "\t" + rs.getObject(4));
        }
        st.executeBatch();
        st.close();

        // 批量插入
        String sql = "INSERT INTO ums_admin2 (username,password) VALUES(?,?)";
        PreparedStatement pstmt = conn.prepareStatement(sql);
        for (int i = 1; i <= 10; i++) {
            pstmt.setString(1, "username" + i);
            pstmt.setString(2, "password" + i);
            pstmt.addBatch();
        }
        pstmt.executeBatch();
        pstmt.close();
        conn.close();
    }
}
```

这种写法在功能上与共有域方法相近，但是它更简洁，无偿地提供了串行化机制，绝对防止对此实例化，即使是在面对复杂的串行化或者反射攻击的时候。虽然这中方法还没有广泛采用，但是单元素的枚举类型已经成为实现Singleton的最佳方法


## 二.结构型模式

### 1. 适配器模式

适配器模式（Adapter Pattern）是作为两个不兼容的接口之间的桥梁

- tomcat CoyoteAdapter

### 2. 桥接模式

### 3. 组合模式

### 4. 装饰模式

装饰者模式(Decorator Pattern)：动态地给一个对象增加一些额外的职责，增加对象功能来说，装饰模式比生成子类实现更为灵活。装饰模式是一种对象结构型模式。

在装饰者模式中，为了让系统具有更好的灵活性和可扩展性，我们通常会定义一个抽象装饰类，而将具体的装饰类作为它的子类

源码分析装饰者模式的典型应用：


总结

**装饰模式的主要优点如下：**

1. 对于扩展一个对象的功能，装饰模式比继承更加灵活性，不会导致类的个数急剧增加。
2. 可以通过一种动态的方式来扩展一个对象的功能，通过配置文件可以在运行时选择不同的具体装饰类，从而实现不同的行为。
3. 可以对一个对象进行多次装饰，通过使用不同的具体装饰类以及这些装饰类的排列组合，可以创造出很多不同行为的组合，得到功能更为强大的对象。
4. 具体构件类与具体装饰类可以独立变化，用户可以根据需要增加新的具体构件类和具体装饰类，原有类库代码无须改变，符合 “开闭原则”。

**装饰模式的主要缺点如下：**

1. 使用装饰模式进行系统设计时将产生很多小对象，这些对象的区别在于它们之间相互连接的方式有所不同，而不是它们的类或者属性值有所不同，大量小对象的产生势必会占用更多的系统资源，在一定程序上影响程序的性能。
2. 装饰模式提供了一种比继承更加灵活机动的解决方案，但同时也意味着比继承更加易于出错，排错也很困难，对于多次装饰的对象，调试时寻找错误可能需要逐级排查，较为繁琐。

**适用场景：**

1. 在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责。
2. 当不能采用继承的方式对系统进行扩展或者采用继承不利于系统扩展和维护时可以使用装饰模式。不能采用继承的情况主要有两类：第一类是系统中存在大量独立的扩展，为支持每一种扩展或者扩展之间的组合将产生大量的子类，使得子类数目呈爆炸性增长；第二类是因为类已定义为不能被继承（如Java语言中的final类）

### 5. 门面模式

门面（Facade）模式也被称为正面模式、外观模式，这种模式用于将一组复杂的类包装到一个简单的外部接口中。

例如：

| 日志门面（抽象层）| 日志实现 |
| ----- | ----- |
| JCL，SLF4j，Jboss-logging| JUL，log4j，log4j2，logback |

### 6. 享元模式

享元模式（Flyweight Pattern）主要用于减少创建对象的数量，以减少内存占用和提高性能。


### 7. 代理模式


## 二.行为模式

### 1. 责任链模式

### 2. 命令模式

### 3. 迭代器模式

### 4. 中介者模式

是一种`行为设计模式`，能让你减少对象之间混乱无序的依赖和调用关系，该模式会限制对象之间的直接交互，迫使他们通过一个中间对象来进行合作，以达到解耦的目的。

```java
public class MyOneTask extends TimerTask {
    private static int num = 0;
    @Override
    public void run() {
        System.out.println("I'm MyOneTask " + ++num);
    }
}

public class MyTwoTask extends TimerTask {
    private static int num = 1000;
    @Override
    public void run() {
        System.out.println("I'm MyTwoTask " + num--);
    }
}
```

```java
public class TimerTest {
    public static void main(String[] args) {
        // 注意：多线程并行处理定时任务时，Timer运行多个TimeTask时，只要其中之一没有捕获抛出的异常，
        // 其它任务便会自动终止运行，使用ScheduledExecutorService则没有这个问题
        Timer timer = new Timer();
        timer.schedule(new MyOneTask(), 3000, 1000); // 3秒后开始运行，循环周期为 1秒
        timer.schedule(new MyTwoTask(), 3000, 1000);
    }
}
```

`Timer`的部分关键源码如下

```java
public class Timer {

    private final TaskQueue queue = new TaskQueue();
    private final TimerThread thread = new TimerThread(queue);
    
    public void schedule(TimerTask task, long delay) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        sched(task, System.currentTimeMillis()+delay, 0);
    }
    
    public void schedule(TimerTask task, Date time) {
        sched(task, time.getTime(), 0);
    }
    
    private void sched(TimerTask task, long time, long period) {
        if (time < 0)
            throw new IllegalArgumentException("Illegal execution time.");

        if (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;

        // 获取任务队列的锁(同一个线程多次获取这个锁并不会被阻塞,不同线程获取时才可能被阻塞)
        synchronized(queue) {
            // 如果定时调度线程已经终止了,则抛出异常结束
            if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");

            // 再获取定时任务对象的锁(为什么还要再加这个锁呢?想不清)
            synchronized(task.lock) {
                // 判断线程的状态,防止多线程同时调度到一个任务时多次被加入任务队列
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException(
                        "Task already scheduled or cancelled");
                // 初始化定时任务的下次执行时间
                task.nextExecutionTime = time;
                // 重复执行的间隔时间
                task.period = period;
                // 将定时任务的状态由TimerTask.VIRGIN(一个定时任务的初始化状态)设置为TimerTask.SCHEDULED
                task.state = TimerTask.SCHEDULED;
            }
            
            // 将任务加入任务队列
            queue.add(task);
            // 如果当前加入的任务是需要第一个被执行的(也就是他的下一次执行时间离现在最近)
            // 则唤醒等待queue的线程(对应到上面提到的queue.wait())
            if (queue.getMin() == task)
                queue.notify();
        }
    }
    
    // cancel会等到所有定时任务执行完后立刻终止定时线程
    public void cancel() {
        synchronized(queue) {
            thread.newTasksMayBeScheduled = false;
            queue.clear();
            queue.notify();  // In case queue was already empty.
        }
    }
    // ...
}
```

Timer 是中介者

### 5. 备忘录模式

### 6. 观察者模式

### 7. 状态模式

### 8. 策略模式（营销策略优惠券折扣）

策略模式属于对象的行为模式。其用意是针对一组算法，将每一个算法封装到具有共同接口的独立的类中，从而使得它们可以相互替换。策略模式使得算法可以在不影响到客户端的情况下发生变化

策略模式是对算法的包装，是把使用算法的责任和算法本身分割开来，委派给不同的对象管理。策略模式通常把一个系列的算法包装到一系列的策略类里面，作为一个抽象策略类的子类

`缺点`：客户端必须知道所有策略的详细实现细节，使用ifelse语法进行选择性的注入对应实现类，代码如下

定义策略接口
```java
import java.math.BigDecimal;

/**
 * 优惠券类型；
 * 1. 直减券  2. 满减券 3. 折扣券  4. n元购
 * 1. 限时折扣 2. 限量折扣 3. 某类商品折扣 4. 某个商品折扣
 */
public interface CouponDiscountService {

	/**
	 * 优惠券折扣计算接口
	 */
	public BigDecimal discountAmount(BigDecimal orderPrice);
}
```

模拟折扣和优惠的实现
```java
public class ZKCouponDiscount implements UserPayDiscountService {

    /**
     * 折扣计算
     * 1. 使用商品价格乘以折扣比例，为最后支付金额
     * 2. 保留两位小数
     * 3. 最低支付金额1元
     */
    public BigDecimal discountAmount(Double couponInfo, BigDecimal skuPrice) {
        BigDecimal discountAmount = skuPrice.multiply(new BigDecimal(couponInfo)).setScale(2, BigDecimal.ROUND_HALF_UP);
        if (discountAmount.compareTo(BigDecimal.ZERO) < 1) return BigDecimal.ONE;
        return discountAmount;
    }
}

public class ZJCouponDiscount implements UserPayDiscountService {

    /**
     * 直减计算
     * 1. 使用商品价格减去优惠价格
     * 2. 最低支付金额1元
     */
    public BigDecimal discountAmount(Double couponInfo, BigDecimal skuPrice) {
        BigDecimal discountAmount = skuPrice.subtract(new BigDecimal(couponInfo));
        if (discountAmount.compareTo(BigDecimal.ZERO) < 1) return BigDecimal.ONE;
        return discountAmount;
    }
}
```

下单
```java
public BigDecimal calPrice(BigDecimal orderPrice,User user) {

     if (直减) {
        //伪代码：从Spring中获取策略对象
        CouponDiscountService strategy = Spring.getBean(ZJCouponDiscount.class);
        return strategy.discountAmount(orderPrice);
     }

     if (折扣) {
        CouponDiscountService strategy = Spring.getBean(ZKCouponDiscount.class);
        return strategy.discountAmount(orderPrice);
     }

     return 原价;
}
```

借助Spring和工厂模式，为了方便我们从Spring中获取CouponDiscountService的各个策略类，我们创建一个工厂类

```java
public class  CouponDiscountServiceStrategyFactory {

    private static Map<String, CouponDiscountService> services = new ConcurrentHashMap<String, CouponDiscountService>();

    public  static  CouponDiscountService getByDiscountType(String type){
        return services.get(type);
    }
    //通过每个实现类初始化的时候调用注册
    public static void register(String userType, CouponDiscountService  CouponDiscountService){
        Assert.notNull(userType,"userType can't be null");
        services.put(userType, CouponDiscountService);
    }
}
```

初始化策略类,`注意`此处也可使用spring的Aware机制进行实例注册
```java
@Service
public class ZKCouponDiscount implements UserPayDiscountService,InitializingBean {

    /**
     * 折扣计算
     * 1. 使用商品价格乘以折扣比例，为最后支付金额
     * 2. 保留两位小数
     * 3. 最低支付金额1元
     */
    public BigDecimal discountAmount(Double couponInfo, BigDecimal skuPrice) {
        BigDecimal discountAmount = skuPrice.multiply(new BigDecimal(couponInfo)).setScale(2, BigDecimal.ROUND_HALF_UP);
        if (discountAmount.compareTo(BigDecimal.ZERO) < 1) return BigDecimal.ONE;
        return discountAmount;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        CouponDiscountServiceStrategyFactory.register("zKCouponDiscount",this);
    }
}
```

下单逻辑简化
```java
public Order calPrice(Order order) {

     String discountType = order.discountType();//获取折扣类型
     CouponDiscountService strategy = CouponDiscountServiceStrategyFactory.getByDiscountType(discountType);
     return strategy.discountAmount(order);
}
```

这并不是严格意义上的策略模式，因为没用使用到`Context`,而使用工厂模式给代替了，Context的存在的意思在于可以对返回的对应进行2次包装，再次减少因为修改策略而对下单逻辑代码进行的改造
```java
public class Context<T> {

    private ICouponDiscount couponDiscount;

    public Context(ICouponDiscount couponDiscount) {
        this.couponDiscount = couponDiscount;
    }

    public Order discountAmount(Order order) {
        return couponDiscount.discountAmount(order);
    }

}
```

```java
public Order calPrice(Order order) {
     String discountType = order.discountType();//获取折扣类型
     CouponDiscountService strategy = CouponDiscountServiceStrategyFactory.getByDiscountType(discountType);
     Context context = new Context(strategy);
     return context.discountAmount(order);
}

```

### 9. 模板模式

- tomcat LifecycleBase initInternal startInternal


### 10. 访问者模式