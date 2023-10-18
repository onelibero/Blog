# **Spring Data JPA**

## **一、概念**


Spring Data JPA是基于SpringFramework的JPA规范实现的一个持久层框架，目的是简化使用JPA进行数据访问的开发。提供了一种更加简单。便捷的方式来执行常见的持久化操作，列如查询、更新、删除、插入实体等

实现了JPA规范的一些核心特性，列如实体映射、实体管理和查询语言，同时提供了一些额外的特性，列如动态查询、命名查询和存储过程调用等

## **二、基础使用**


（此处用于替代mybatis完成持久层工作在springboot中）
方便性：
●不再需要使用xml类型的配置文件配置实体和数据库表直接的映射关系，改在实体类中使用注解来处理，以此实现零配置。
●数据库表不再需要自行创建，而是由spring data jpa根据实体类及其注解来自动创建
●dao包中数据访问逻辑的接口方法由spring data jpa来简化操作

### **1.项目添加jpa依赖** 

### **2.编写yml配置文件，添加持久层配置** 

简述配置
●properties：配置jpa的具体orm实现框架，此处为hibernate
●hibernate中的具体配置：
●hbm2ddl:此配置是hibernate model到sql定义语句，使用此配置后我们只需要创建数据库，不再需要创建数据库表，它会根据java实体类自动映射生成
●dialect：即方言，此处针对mysql数据库生成优化的sql
●format_sql：格式化sql语句，使其可读性更好
●enable_lazy_load_no_trans：配置为true，可以在测试环境下模拟出事务效果
●show-sql：配置为true，可以将sql语句打印输出到控制台便于调试

### **3.创建实体类添加jpa注解** 

#### **（1）相应注解**

- Entity：该注解表示该类为实体类
- Table：该注解表示该实体类对应的数据库表，可以使用name属性指定数据库的表名，为指定表名为类名
- ID：该注解表示该属性为主键，同时使用@GeneratedValue注解指定主键的生成策略，通常使用IDENTITY
- Transient：表示该实体类的方法不会映射到数据库中（可以通过其他属性计算或者什么方式获取）
- Column：指定该属性映射到数据库表的对应字段，可以为空
- ManyToOne：用于多对一关系
- OneToMany：用于一对多关系，使用mapperBy属性表示该关系由另一方维护，一般在较多那方维护
  - mappedBy：属性则是用来指定该关系中对方实体类中与当前实体类关联的属性名称。
- ManyToMany：用于多对多关系
- JoinTable：指定关系表的名称，并在其中指定该表内的列的信息
  - name：指定中间表的名称
  - joinColumns：指定本实体在关系表中的列名、外键约束
  - inverseJoinColumn：指定另一个实体在关系表中的列名、外键约束
  - uniqueConstraints：指定联合主键
- JoinColumn：常与一对多，多对一，多对多注解处使用
  - name：指定与当前属性相关联的外键列的名称
  - referencedColumnName：指定当前属性关联的目标实体类中的主键属性名称
  - nullable：指定外键列是否可以为空
  - insertable：指定是否在插入实体时插入当前属性的值
  - updatable：指定是否在更新实体时更新当前属性的值
  - foreignKey：具体而言这是个注解，用于指定外键约束的详细信息
    - name：外键约束的名称
    - foreignKeyDefinition：指定外键约束的SQL定义
    - value：指定外键约束的详细信息，包括name、foreignKeyDefinition、updateAction和deleteAction。
    - updateAction，deleteAction属性用于指定在更新或删除实体时，如何处理与该实体相关联的外键约束
      - ForeignKeyAction.CASCADE: 执行级联操作，将与该实体相关联的所有外键约束一并更新或删除；
      - ForeignKeyAction.RESTRICT: 禁止执行该操作，如果存在与该实体相关联的外键约束，则抛出异常；
      - ForeignKeyAction.SET_NULL: 将与该实体相关联的外键列值设置为null；
      - ForeignKeyAction.NO_ACTION: 不执行任何操作，即不更新或删除与该实体相关联的外键约束。 

#### **（2）展示一对多和多对一的关系写法**


此下是一个部门表

```java
@Entity
@Table
@Data
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    @Column
    private String name;
    @Column
    private Integer number;
 		@OneToMany(cascade = {CascadeType.PERSIST},mappedBy = "dep")
    private List<Employee> employeeList;
}
```

此下是一个员工表

```java
@Entity
@Table
@Data
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    @Column
    private Integer number;
    @Column
    private String name;
    @Column
    private String gender;
    @Column
    private Integer age;
 		@ManyToOne
    @JoinColumn(foreignKey = @ForeignKey(name = "none",value = ConstraintMode.NO_CONSTRAINT))
    private Department dep;
}
```

#### **（3）展示多对多的写法**


此处阐述 ：用户-角色 角色-权限
以下是一个用户表的创建

```java
@Entity
@Data
@Table(name = "user")
public class SysUser {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    @Column
    private String username;
    @Column
    private String password;
    @ManyToMany
    @OrderBy("id asc ")
    @JoinTable(name = "r_user_role",
                joinColumns =@JoinColumn(name = "u_id",foreignKey = @ForeignKey(name = "none",value = ConstraintMode.NO_CONSTRAINT)),
                inverseJoinColumns = @JoinColumn(name = "r_id",foreignKey = @ForeignKey(name = "none",value = ConstraintMode.NO_CONSTRAINT)),
                uniqueConstraints = @UniqueConstraint(columnNames = {"u_id","r_id"}))
    private List<SysRole> roles;
    @Transient
    private SysRole role;
}
```



以下是一个角色表的创建

```java
package cdu.gu.springdatajpa.entity;

import lombok.Data;

import javax.persistence.*;
import java.util.List;

@Entity
@Table(name = "role")
@Data
public class SysRole {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    @Column
    private String code;
    @Column
    private String name;
    @ManyToMany
    @JoinTable(name = "r_role_permission",
                joinColumns = @JoinColumn(name = "r_id",foreignKey = @ForeignKey(name = "none",value = ConstraintMode.NO_CONSTRAINT)),
               inverseJoinColumns = @JoinColumn(name = "p_id",foreignKey = @ForeignKey(name = "none",value = ConstraintMode.NO_CONSTRAINT)),
                uniqueConstraints = @UniqueConstraint(columnNames = {"r_id","p_id"}))
    private List<SysPermission> sysPermissions;

    @ManyToMany(mappedBy = "roles")
    private List<SysUser> users;
}
```

以下是一个权限表的创建

```java
@Entity
@Table(name = "permission")
@Data
public class SysPermission {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    @Column
    private String code;
    @Column
    private String name;
}
```