---
title: maven
date: 2017-10-14 13:52:37
categories:
- java
---

## Maven

### 坐标

maven 的所有构件均通过坐标进行组织和管理。maven 的坐标通过 5 个元素进行定义，其中 groupId、artifactId、version 是必须的，packaging 是可选的（默认为jar），classifier 是不能直接定义的。

- groupId：定义当前 Maven 项目所属的实际项目，跟 Java 包名类似，通常与域名反向一一对应。
- artifactId：定义当前 Maven 项目的一个模块，默认情况下，Maven 生成的构件，其文件名会以 artifactId 开头，如 hibernate-core-3.6.5.Final.jar。
- version：定义项目版本。
- packaging：定义项目打包方式，如 jar，war，pom，zip ……，默认为 jar。
- classifier：定义项目的附属构件，如 hibernate-core-3.6.6.Final-sources.jar，hibernate-core-3.6.6.Final-javadoc.jar，其中 sources 和 javadoc 就是这两个附属构件的 classifier。classifier 不能直接定义，通常由附加的插件帮助生成。

### Maven提供的一些管理依赖功能

- Dependency mediation: 当出现依赖项目为多个版本时，决定使用哪个版本。如果两个版本在依赖树上处于相同深度时，先定义的依赖版本将会被使用。
- Dependency management: 依赖传递时，会使用直接指定的版本。例如，B < C, 在C中定义依赖时在它的dependencyManagement部分直接指定B的版本，在C被依赖时，将会使用对应B的版本。
- Dependency scope: 按构建阶段包含依赖
- Excluded dependencies: 传递依赖时可在“exclusion”元素中排除依赖。例如，B是A的依赖，C是B的依赖，A可以排除C作为依赖
- Optional dependencies: 依赖传递时，可使用"optional"将依赖标记为可选项。例如，B是A的依赖，C是B的依赖，然后B标记C是可选项的，那么A将不会使用C。

粘贴一下别人的大白话：

#### 依赖冲突

通常我们不需要关心传递性依赖，当多个传递性依赖中有对同一构件不同版本的依赖时，如何解决呢？

- 短路径优先：假如有以下依赖：A -> B -> C ->X(版本 1.0) 和 A -> D -> X(版本 2.0)，则优先解析较短路径的 X(版本 2.0)；
- 先声明优先：若路径长度相同，则谁先声明，谁被解析。

#### 依赖排除

针对依赖冲突中的“短路径优先”，如果我们想使用长路径的依赖怎么办呢？这时可以使用依赖排除 <exclusions> 元素，显示排除短路径依赖。在非冲突的情况下，这种方法同样有效。

#### 依赖归类

通常在项目中，我们会同时依赖同一个构件的不同模块，如 spring-orm-3.2.0，spring-context-3.2.0，且多个模块版本相同，为了维护和升级方便，我们可以对其同一管理，这时可以使用到 Maven 属性，类似于变量的概念。

```
 <properties>
     <springframework.version>3.2.0.RELEASE</springframework.version>
  </properties>

  <dependencies>
      <dependency>
          <groupId>org.springframework</groupId>
          <artifactId>spring-orm</artifactId>
          <version>${springframework.version}</version>
     </dependency>
     <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-context</artifactId>
         <version>${springframework.version}</version>
     </dependency>
 </dependencies>

```

--

## Maven Test

1. 新建测试类，new JUnit Case Test

2. 新建类时选择的几个方法的解释。    

   - setUp： 测试前的初始化工作
   - tearDown: 测试完成后垃圾回收工作
   - constructor: 构造方法

3. 几种标注的介绍。

   - @Test,表示这个是测试方法
   - @Before,这个方法会在每个测试方法前都执行一次
   - @After,这个方法会在每个测试方法后都执行一次
   - @Ignore, 表示这个方法在测试的时候会被忽略
   - @BeforeClass, 只在测试用例初始化时执行
   - @AfterClass,，当所有测试执行完毕之后进行收尾工作
   - 每个测试类只能有一个方法被标注为 @BeforeClass 或 @AfterClass ，并且该方法必须是 Public 和 Static 的。
   - @Test(timeout  =   1000 )  限时测试
   - @Test(expected  =  ArithmeticException. class ) 异常测试
   - 参数化测试，这个以代码来解释怎么用。下面的测试数据是3组，会有3次结果。

   ```
   	public class SquareTest {

   		private static Calculator calculator = new Calculator();
   		private int param;
   		private int result;

   		@Parameters
   	      public   static  Collection data()  {
   	          return  Arrays.asList( new  Object[][] {
   	                  { 2 ,  4 } ,
   	                  { 0 ,  0 } ,
   	                  {－ 3 ,  9 } ,
   	         } );
   	     }

   		// 构造函数，对变量进行初始化
   		public SquareTest(int param, int result) {
   			this.param = param;
   			this.result = result;
   		}

   		@Test
   		public void square() {
   			calculator.square(param);
   			assertEquals(result, calculator.getResult());
   		}

   	}
   ```

4. 使用mvn test进行测试
