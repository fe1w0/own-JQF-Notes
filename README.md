>还是第一次接触，Fuzzing

Todo:
- [ ] 初级目标，在手动预分析的基础上，可以通过JQF 检查 example.jar
- [ ] 利用 JQF 实现 property-based tree
# References
- Tutorial:
	- [Fuzzing with Zest · rohanpadhye/JQF Wiki (github.com)](https://github.com/rohanpadhye/jqf/wiki/Fuzzing-with-Zest)
- Papers:
	- [JQF:Java 中基于覆盖率引导的属性测试 - 每日头条 (kknews.cc)](https://kknews.cc/code/yjavpjj.html)
	- 原 JQF的Readme有介绍 
# Introduction

>暂无，新手

## Requirements
- maven
- java 9+ (果然只有国内项目普遍使用java8)

## Build JQF

```
git clone https://github.com/rohanpadhye/jqf
cd jqf
mvn package
```

## Tutorial
>基本是GitHub上的Tutorial Docs, 添加部分自己的理解。
### 0x01 Fuzzing-with-Zest

 source code
- CalendarLogic.java
	- 被测试对象
- CalendarGenerator.java 
	- 测试样板生成器（随机）
- CalendarTest.java
	- 测试脚本
>源代码，见 https://github.com/rohanpadhye/jqf/wiki/Fuzzing-with-Zest
>具体的代码解释，也请见原文 和 官方 Docs
>测试框架为 junit-quickcheck
>https://pholser.github.io/junit-quickcheck/site/0.8.2/usage/other-types.html

>还是大致具体源代码含义:

**CalendarLogic.java**
```java
import java.util.GregorianCalendar;
import static java.util.GregorianCalendar.*;

/* Application logic */
public class CalendarLogic {
    // Returns true iff cal is in a leap year
    public static boolean isLeapYear(GregorianCalendar cal) {
        int year = cal.get(YEAR);
        if (year % 4 == 0) {
            if (year % 100 == 0) {
                return false;
            }
            return true;
        }
        return false;
    }

    // Returns either of -1, 0, 1 depending on whether c1 is <, =, > than c2
    public static int compare(GregorianCalendar c1, GregorianCalendar c2) {
        int cmp;
        cmp = Integer.compare(c1.get(YEAR), c2.get(YEAR));
        if (cmp == 0) {
            cmp = Integer.compare(c1.get(MONTH), c2.get(MONTH));
            if (cmp == 0) {
                cmp = Integer.compare(c1.get(DAY_OF_MONTH), c2.get(DAY_OF_MONTH));
                if (cmp == 0) {
                    cmp = Integer.compare(c1.get(HOUR), c2.get(HOUR));
                    if (cmp == 0) {
                        cmp = Integer.compare(c1.get(MINUTE), c2.get(MINUTE));
                        if (cmp == 0) {
                            cmp = Integer.compare(c1.get(SECOND), c2.get(SECOND));
                            if (cmp == 0) {
                                cmp = Integer.compare(c1.get(MILLISECOND), c2.get(MILLISECOND));
                            }
                        }
                    }
                }
            }
        }
        return cmp;
    }
}
```

**CalendarGenerator.java**
```java
import java.util.GregorianCalendar;
import java.util.TimeZone;

import com.pholser.junit.quickcheck.generator.GenerationStatus;
import com.pholser.junit.quickcheck.generator.Generator;
import com.pholser.junit.quickcheck.random.SourceOfRandomness;

import static java.util.GregorianCalendar.*;

public class CalendarGenerator extends Generator<GregorianCalendar> {

    public CalendarGenerator() {
        super(GregorianCalendar.class); // Register the type of objects that we can create
    }

    // This method is invoked to generate a single test case
    @Override
    public GregorianCalendar generate(SourceOfRandomness random, GenerationStatus __ignore__) {
        // Initialize a calendar object
        GregorianCalendar cal = new GregorianCalendar();
        cal.setLenient(true); // This allows invalid dates to silently wrap (e.g. Apr 31 ==> May 1).

        // Randomly pick a day, month, and year
        cal.set(DAY_OF_MONTH, random.nextInt(31) + 1); // a number between 1 and 31 inclusive
        cal.set(MONTH, random.nextInt(12) + 1); // a number between 1 and 12 inclusive
        cal.set(YEAR, random.nextInt(cal.getMinimum(YEAR), cal.getMaximum(YEAR)));

        // Optionally also pick a time
        if (random.nextBoolean()) {
            cal.set(HOUR, random.nextInt(24));
            cal.set(MINUTE, random.nextInt(60));
            cal.set(SECOND, random.nextInt(60));
        }

        // Let's set a timezone
        // First, get supported timezone IDs (e.g. "America/Los_Angeles")
        String[] allTzIds = TimeZone.getAvailableIDs();

        // Next, choose one randomly from the array
        String tzId = random.choose(allTzIds);
        TimeZone tz = TimeZone.getTimeZone(tzId);

        // Assign it to the calendar
        cal.setTimeZone(tz);

	// Return the randomly generated calendar object
        return cal;
    }
}
```

**CalendarTest.java**
```java
import java.util.*;
import static java.util.GregorianCalendar.*;
import static org.junit.Assert.*;
import static org.junit.Assume.*;

import org.junit.runner.RunWith;
import com.pholser.junit.quickcheck.*;
import com.pholser.junit.quickcheck.generator.*;
import edu.berkeley.cs.jqf.fuzz.*;

@RunWith(JQF.class)
public class CalendarTest {

    @Fuzz
    public void testLeapYear(@From(CalendarGenerator.class) GregorianCalendar cal) {
        // Assume that the date is Feb 29
        assumeTrue(cal.get(MONTH) == FEBRUARY);
        assumeTrue(cal.get(DAY_OF_MONTH) == 29);

        // Under this assumption, validate leap year rules
        assertTrue(cal.get(YEAR) + " should be a leap year", CalendarLogic.isLeapYear(cal));
    }

    @Fuzz
    public void testCompare(@Size(max=100) List<@From(CalendarGenerator.class) GregorianCalendar> cals) {
        // Sort list of calendar objects using our custom comparator function
        Collections.sort(cals, CalendarLogic::compare);

        // If they have an ordering, then the sort should succeed
        for (int i = 1; i < cals.size(); i++) {
            Calendar c1 = cals.get(i-1);
            Calendar c2 = cals.get(i);
            assumeFalse(c1.equals(c2)); // Assume that we have distinct dates
            assertTrue(c1 + " should be before " + c2, c1.before(c2));  // Then c1 < c2
        }
    }
}
```

```bash
# 编译class文件
javac -cp .:$(../JQF/scripts/classpath.sh) CalendarLogic.java CalendarGenerator.java CalendarTest.java

# 执行测试，但产生以下报错:
# Assumption is too strong; too many inputs discarded
java -cp .:$(../JQF/scripts/classpath.sh) org.junit.runner.JUnitCore CalendarTest
```

上面会报错，作者卖了个关子，大致是说，`CalendarGenerator.generate()` 是一个标准的伪随机源，绝大多数数据很有可能不会触发到设置的`testLeapYear`

启用 Zest 算法

```bash
../JQF/bin/jqf-zest -c .:$(../JQF/scripts/classpath.sh) CalendarTest testLeapYear

# mvn 的方式
mvn jqf:fuzz -Dclass=CalendarTest -Dmethod=testLeapYear
```

>很酷！太Auto 优雅了。

![](https://raw.githubusercontent.com/fe1w0/ImageHost/main/image/16864732310131686473230468.png)

部分输出结果:
- Unique failure inputs:
	- fuzz-results/failures
- Common successful inputs:
	- fuzz-results/corpus

在 Ctrl + C 手动中止后，可以使用如下命令来查看和分析失败的样例。
```bash
../JQF/bin/jqf-repro -c .:$(../JQF/scripts/classpath.sh) CalendarTest testLeapYear fuzz-results/failures/id_000000
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
.id_000000 ::= FAILURE (java.lang.AssertionError)
E
Time: 0.062
There was 1 failure:
1) testLeapYear(CalendarTest)
java.lang.AssertionError: 28271600 should be a leap year
        at org.junit.Assert.fail(Assert.java:89)
        at org.junit.Assert.assertTrue(Assert.java:42)
        at CalendarTest.testLeapYear(CalendarTest.java:21)

FAILURES!!!
Tests run: 1,  Failures: 1
```

> 28271600 是闰年，可被 400 整除

```bash
jqf-zest -c .:$(${JQF_HOME}/scripts/classpath.sh) CalendarTest testCompare

# output:
# COOL!
Semantic Fuzzing with Zest
--------------------------

Test name:            CalendarTest#testCompare
Instrumentation:      Janala
Results directory:    /home/fe1w0/SoftwareAnalysis/DynamicAnalysis/tutorial/fuzz-results
Elapsed time:         11s (no time limit)
Number of executions: 6,254 (no trial limit)
Valid inputs:         6,144 (98.24%)
Cycles completed:     1
Unique failures:      1
Queue size:           36 (5 favored last cycle)
Current parent input: 0 (favored) {564/700 mutations}
Execution speed:      776/sec now | 523/sec overall
Total coverage:       32 branches (0.05% of map)
Valid coverage:       32 branches (0.05% of map)
```


小节
JQF 设置属性的方式，有点类似从反射或者set的方式，对object中的属性进行设置，但example 给的变量有点不太auto.

JQF 的 driver 编写流程:
1. 初始化阶段:
	1. 编写 生成器，需要符合一定规范，以供 test driver 使用。
		1. `public class CalendarGenerator extends Generator<GregorianCalendar> `
		2. `    public CalendarGenerator() { super(GregorianCalendar.class); // Register the type of objects that we can create    }`
		3. `@Override     public GregorianCalendar generate(SourceOfRandomness random, GenerationStatus __ignore__) { ... ... ret cal // Return the randomly generated calendar object }` 
	2. 通过 反射或者set的方式，对需要设置的field进行随机赋值，如`cal.setLenient(true);` 、`cal.set(MONTH, random.nextInt(12) + 1);`等等
3. 测试阶段，编写 test driver :
	1. 基本要求，类注解 `@RunWith(JQF.class)`
	2. 测试函数注解，`@Fuzz`
	3. 测试函数参数注解
		1. `public void testLeapYear(@From(CalendarGenerator.class) GregorianCalendar cal)` 
		2. `public void testCompare(@Size(max=100) List<@From(CalendarGenerator.class) GregorianCalendar> cals)`
	4. 断言与函数测试脚本编写
### 0x02 Fuzzing a Compiler
