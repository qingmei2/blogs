> 原文：**Kotlin-ifying a Builder Pattern**
原文地址：https://medium.com/google-developers/kotlin-ifying-a-builder-pattern-e5540c91bdbe
原文作者：[Doug Sigelbaum](https://medium.com/@sigelbaum)
翻译：[却把清梅嗅](https://blog.csdn.net/mq2553299)

## 译者说
[Doug Sigelbaum](https://medium.com/@sigelbaum)是Google的Android工程师，在这篇文章中，作者讲述了如何用Kotlin中Builder模式的实现方式，并且针对会出现的问题提出了对应的解决方案。

我最近在网上翻看了很多Kotlin对Builder模式实现方式的文章，说实话，个人感觉都不是很好，当我阅读到这篇文章时，我认为这是目前我 **比较满意** 的实现方式（可能Google有加分），因此翻译下来以供大家参考。

* * * 

在Java语言中，当一个对象的实例化需要多个参数时，建造者模式（Builder）已被认可为非常好的实现方式之一。 正如《Effective Java》指出的，当一个构造器拥有太多的参数时，对于构造器中所需参数的修改很容易影响到实际的代码。

当然，Kotlin语言中的命名参数在许多情况下解决了这个问题，因为在调用Kotlin的函数时，开发者可以指定每个参数的名称，从而减少错误的发生。 但是，由于Java并没有这样的特性，因此Builder模式仍然是有必要的。 此外，对于可选参数的动态设置，这种情况下也需要借助于Builder模式。

让我们思考一下通过Java实现的一个简单的Builder模式案例。 我们首先有一个POJO Company类，它包含几个属性，也许这种情况足以使用Builder模式：

```Java
public final class Company {
    public final String name;
    public final double marketCap;
    public final double annualCosts;
    public final double annualRevenue;
    public final List<Employee> employees;
    public final List<Office> offices;

    private Company(Builder builder) {
        List<Employee> builtEmployees = new ArrayList<>();
        for (Employee.Builder employee : builder.employees) {
            builtEmployees.add(employee.build());
        }
        List<Office> builtOffices = new ArrayList<>();
        for (Office.Builder office : builder.offices) {
            builtOffices.add(office.build());
        }
        employees = Collections.unmodifiableList(builtEmployees);
        offices = Collections.unmodifiableList(builtOffices);
        name = builder.name;
        marketCap = builder.marketCap;
        annualCosts = builder.annualCosts;
        annualRevenue = builder.annualRevenue;
    }

    public static class Builder {
        private String name;
        private double marketCap;
        private double annualCosts;
        private double annualRevenue;
        private List<Employee.Builder> employees = new ArrayList<>();
        private List<Office.Builder> offices = new ArrayList<>();

        public Company build() {
            return new Company(this);
        }

        public Builder addEmployee(Employee.Builder employee) {
            employees.add(employee);
            return this;
        }

        public Builder addOffice(Office.Builder office) {
            offices.add(office);
            return this;
        }

        public Builder setName(String name) {
            this.name = name;
            return this;
        }

        public Builder setMarketCap(double marketCap) {
            this.marketCap = marketCap;
            return this;
        }

        public Builder setAnnualCosts(double annualCosts) {
            this.annualCosts = annualCosts;
            return this;
        }

        public Builder setAnnualRevenue(double annualRevenue) {
            this.annualRevenue = annualRevenue;
            return this;
        }
    }
}
```
此外，公司有List<Employees>和List<Offices>。 这些类也使用构建器模式：

```Java
public final class Employee {
    public final String firstName;
    public final String lastName;
    public final String id;
    public final boolean isManager;
    public final String managerId;

    private Employee(Builder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.id = builder.id;
        this.isManager = builder.isManager;
        this.managerId = builder.managerId;
    }

    public static class Builder {
        private String firstName;
        private String lastName;
        private String id;
        private boolean isManager;
        private String managerId;

        Employee build() {
            return new Employee(this);
        }

        public Builder setFirstName(String firstName) {
            this.firstName = firstName;
            return this;
        }

        public Builder setLastName(String lastName) {
            this.lastName = lastName;
            return this;
        }

        public Builder setId(String id) {
            this.id = id;
            return this;
        }

        public Builder setIsManager(boolean manager) {
            isManager = manager;
            return this;
        }

        public Builder setManagerId(String managerId) {
            this.managerId = managerId;
            return this;
        }
    }
}
```
来看看Office.java：

```Java
public final class Office {
    public final String address;
    public final int capacity;
    public final int occupancy;
    public final int sqft;

    private Office(Builder builder) {
        address = builder.address;
        capacity = builder.capacity;
        occupancy = builder.occupancy;
        sqft = builder.sqft;
    }

    public static class Builder {
        private String address;
        private int capacity;
        private int occupancy;
        private int sqft;

        Office build() {
            return new Office(this);
        }

        public Builder setAddress(String address) {
            this.address = address;
            return this;
        }

        public Builder setCapacity(int capacity) {
            this.capacity = capacity;
            return this;
        }

        public Builder setOccupancy(int occupancy) {
            this.occupancy = occupancy;
            return this;
        }

        public Builder setSqft(int sqft) {
            this.sqft = sqft;
            return this;
        }
    }
}
```

现在，如果我们想构建一个包含Employee和Office的Company，我们可以这样：

```Java
public class JavaClient {
    public Company buildCompany() {
        Company.Builder company = new Company.Builder();
        Employee.Builder employee = new Employee.Builder()
                .setFirstName("Doug")
                .setLastName("Sigelbaum")
                .setIsManager(false)
                .setManagerId("XXX");
        Office.Builder office = new Office.Builder()
                .setAddress("San Francisco")
                .setCapacity(2500)
                .setOccupancy(2400);
        company.setAnnualCosts(0)
                .setAnnualRevenue(0)
                .addEmployee(employee)
                .addOffice(office);
        return company.build();
    }
}
```
在Kotlin中，我们会这样去实现：

```Kotlin
class KotlinClient {
    fun buildCompany(): Company {
        val company = Company.Builder()
        val employee = Employee.Builder()
            .setFirstName("Doug")
            .setLastName("Sigelbaum")
            .setIsManager(false)
            .setManagerId("XXX")
        val office = Office.Builder()
            .setAddress("San Francisco")
            .setCapacity(2500)
            .setOccupancy(2400)
        company.setAnnualCosts(0.0)
            .setAnnualRevenue(0.0)
            .addEmployee(employee)
            .addOffice(office)
        return company.build()
    }
}
```

## Kotlin中实现Lambda参数的方法封装

在Dokka（译者注:这应该是作者之前所在的一个公司），我们使用kotlinx.html，它可以通过一个漂亮的DSL来实例化HTML的对象。 在Android中，它和通过Anko Layouts构建布局类似。 正如我在上一篇文章中所讨论的，slice-builders-ktx还在Builder模式之外提供了DSL包装器。 所有这些库都使用lambda参数提供了DSL的实现方式。 Lambda参数在Kotlin和Java 8+中可用，它们的使用方式略有不同。 有很多同行朋友，特别是Android开发者，都在使用Java 7，我们只会在这篇文章中简单使用一下Kotlin lambda参数。 现在让我们尝试为Company类提供DSL的支持！

### 顶层的封装（Top Level Wrapper）
这里是Kotlin中的一个顶层函数，这个函数将会为Company对象提供DSL的唯一支持：

```Kotlin
inline fun company(buildCompany: Company.Builder.() -> Unit): Company {
    val builder = Company.Builder()
    // Since `buildCompany` is an extension function for Company.Builder,
    // buildCompany() is called on the Company.Builder object.
    builder.buildCompany()
    return builder.build()
}
```
**注意**：这里我们使用了 **内联函数（inline）** 以避免lambda的额外开销。

因为方法中的lambda参数类型为 **Company.Builder.() -> Unit** ，因此，该lambda中所有语句都处于Company.Builder的内部。现在，通过Kotlin的语法，我们可以通过调用build（）以获得Company.Builder的实例，而不是直接实例化Builder:

```Kotlin
class KtxClient1 {
    fun buildCompany(): Company {
        return company {
            // `this` scope is the Company.Builder being built.
            addEmployee(
                Employee.Builder()
                    .setFirstName("Doug")
                    .setLastName("Sigelbaum")
                    .setIsManager(false)
                    .setManagerId("XXX")
            )
            addOffice(
                Office.Builder()
                    .setAddress("San Francisco")
                    .setCapacity(2500)
                    .setOccupancy(2400)
            )
        }
    }
}
```
### 封装嵌套的Builder
我们现在可以为Company.Builder添加更多扩展函数，以避免直接实例化或将Employee.Builders或Office.Builders添加到父Company.Builder。 这是一个潜在的解决方案：

```Kotlin
inline fun Company.Builder.employee(
    buildEmployee: Employee.Builder.() -> Unit
) {
    val builder = Employee.Builder()
    builder.buildEmployee()
    addEmployee(builder)
}

inline fun Company.Builder.office(buildOffice: Office.Builder.() -> Unit) {
    val builder = Office.Builder()
    builder.buildOffice()
    addOffice(builder)
}
```
通过这些拓展函数，在Kotlin中的使用方式等效变成了：

```Kotlin
class KtxClient2 {    
    fun buildCompany(): Company {
        return company {
            employee {
                setFirstName("Doug")
                setLastName("Sigelbaum")
                setIsManager(false)
                setManagerId("XXX")
            }
            office {
                setAddress("San Francisco")
                setCapacity(2500)
                setOccupancy(2400)
            }
        }
    }
}
```

几乎大功告成了！ 我们已经完成了对Builder中API的优化，但是我们不得不面对一个新问题：

```Kotlin
class KtxBadClient {  
    fun buildBadCompany(): Company {
        return company {
            employee {
                setFirstName("Doug")
                setLastName("Sigelbaum")
                setIsManager(false)
                setManagerId("XXX")
                employee {
                    setFirstName("Sean")
                    setLastName("Mcq")
                    setIsManager(false)
                    setManagerId("XXX")
                }
            }
            office {
                setAddress("San Francisco")
                setCapacity(2500)
                setOccupancy(2400)
            }
        }
    }
}
```

不幸的是，我们把一个employee的Builder嵌套进入了另外一个employee的Builder中，但是这样仍然会通过编译并运行！在Kotlin中，类的作用范围似乎发生了混乱，这意味着，company { … } 的lambda代码块中，这些嵌套的lambda代码块中都可以任意访问Employee.Builder和Company.Builder中的内容。现在，代码将两名员工（Employee）“Doug”和“Sean”添加到公司（Company），但是这两名员工实际上并没有直接的关系。

当作用域发生混乱时，如何修改扩展函数以避免上例所示的错误呢？ 换句话说，我们该如何才能使我们的DSL类型安全？ 幸运的是，Kotlin 1.1引入了DslMarker注释类来解决这个问题。

## 使用DslMarker注解保证DSL的类型安全

让我们首先创建一个使用了DslMarker注解的注解类：

```Kotlin
@DslMarker
annotation class CompanyDsl
```

现在，如果使用@CompanyDsl注释了一些类，开发者将无法对多个接收器进行隐式地访问，这些接收器的类位于带注释的类集中。 相反，调用者只能使用最近的作用域隐式访问接收器。

DslMarker注解类，位于Kotlin的stdlib包中，因此您需要添加其依赖。 如果没有的话，您可以尝试对构建器进行子类化，并在Kotlin封装好的函数中使用这些子类：

```Kotlin
@CompanyDsl
class CompanyBuilderDsl : Company.Builder()

@CompanyDsl
class EmployeeBuilderDsl : Employee.Builder()

@CompanyDsl
class OfficeBuilderDsl : Office.Builder()

inline fun company(buildCompany: CompanyBuilderDsl.() -> Unit): Company {
    val builder = CompanyBuilderDsl()
    // Since `buildCompany` is an extension function for Company.Builder,
    // buildCompany() is called on the Company.Builder object.
    builder.buildCompany()
    return builder.build()
}

inline fun CompanyBuilderDsl.employee(
    buildEmployee: EmployeeBuilderDsl.() -> Unit
) {
    val builder = EmployeeBuilderDsl()
    builder.buildEmployee()
    addEmployee(builder)
}

inline fun CompanyBuilderDsl.office(
    buildOffice: OfficeBuilderDsl.() -> Unit
) {
    val builder = OfficeBuilderDsl()
    builder.buildOffice()
    addOffice(builder)
}
```

现在，我们重复之前错误的行为，会得到下面的提示：

> …can’t be called in this context by implicit receiver. Use the explicit one if necessary.

完美，大功告成！
