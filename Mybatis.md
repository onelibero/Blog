# Mybatis

## 1.#{}和${}

- ${}是变量占位符，属于静态文本替换

  - 主要用于传入的参数是sql片段的场景下，会直接进行字符串替换完成sql的拼接

    > 比如有这样一个需求：对于后台管理系统的table列，想让用户选择展示哪些列，不展示哪些列，或者根据权限来分配不同的用户可以看到哪些列？
    >
    > 这样的话就是每次展示列表的时候根据用户ID和table类型，到数据表中查询该用户对该table能看到的列名列表，然后吧该字段名列表传入Mybatis进行查询

- \#{}是sql的参数占位符，Mybatis会将sql中的#{}替换为?号，在执行时在取值，可以防止sql注入

  - 主要进行数据类型检查和安全检查两部分
  - 在sql映射文件中动态拼接sql时的开发场景

## 2.常用标签

- select
- insert
- update
- delete
- resultMap：表名列之间的关系
- parameterMap
- selectKey：为不支持自增的主键生成策略
- 动态sql：trim，where，set，foreach，if，choose，when，otherwise，bind
  - sql：<sql>为sql片段标签
  - include：通过<include>标签引入sql片段

## 3.Dao接口工作原理

通常一个xml映射文件，都会写一个Dao接口与之对应。Dao接口就是常说的Mapper接口

- 接口的全限名，就是映射文件中namespace的值
- 接口的方法名，就是映射文件中mappendStatement的id值
- 接口方法内的参数，就是传递给sql的参数

Mapper接口是没有实现类的，当调用接口方法时，接口全限名+方法名拼接字符串作为key，可唯一定位一个MappedStatement

在Mybatis中，每一个标签都会被解析成一个MappedStatement对象

> dao接口的方法可以重载，但是Mybatis的xml里面的ID不能重复

```java
/**
 * Mapper接口里面方法重载
 */
public interface StuMapper {

 List<Student> getAllStu();

 List<Student> getAllStu(@Param("id") Integer id);
}
```

通过动态sql解决重载的问题（但是多个接口对应的映射只要一个，否则启动就会报错）

> 1. 仅有一个无参方法和一个有参方法
> 2. 多个有参方法时，参数数量必须一致。且使用相同的 `@Param` ，或者使用 `param1` 这种

```java
<select id="getAllStu" resultType="com.pojo.Student">
  select * from student
  <where>
    <if test="id != null">
      id = #{id}
    </if>
  </where>
</select>
```

## 4.分页

- Mybatis使用RowBounds对象进行分页，是针对ResultSet结果集执行的内存分页，而非物理分页
- 可以在sql内存直接书写带有物理分页的参数来完成物理分页
- 可以使用分页插件来完成物理分页
  - 分页插件的基本原理是使用Mybatis提供的插件接口，实现自定义插件，在插件的拦截方法内拦截待执行的sql，然后重写sql，根据dialect方言，添加对应的物理分页语句和物理分页参数

## 5.动态sql