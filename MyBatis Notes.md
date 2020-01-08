# MyBatis
## SqlSessionFactoryBuilder -> SqlSessionFactory -> SqlSession -> Mapper
学习MyBatis要首先分清楚这几个概念：
+ **SqlSessionFactoryBuilder**：构造器，根据配置信息负责生成SqlSessionFactory（工厂接口）。
+ **SqlSessionFactory**：依靠本工厂接口来生成SqlSession。
+ **SqlSession**：包含了面向数据库执行SQL命令所需的所有方法。既可以发送Sql去执行并返回结果，也可以获取Mapper接口。
+ **Mapper**：由一个Java接口和对应的XML文件（或者注解）构成，需要给出对应的SQL和映射规则，负责发送SQL去执行，并返回结果。

每个基于 MyBatis 的应用都是以一个 SqlSessionFactory 的实例为核心的。SqlSessionFactory 的实例可以通过 SqlSessionFactoryBuilder 获得。而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先定制的 Configuration 的实例构建出 SqlSessionFactory 的实例。

### SqlSessionFactoryBuilder
SqlSessionFactoryBuilder 有五个 build() 方法
```
SqlSessionFactory build(InputStream inputStream)
SqlSessionFactory build(InputStream inputStream, String environment)
SqlSessionFactory build(InputStream inputStream, Properties properties)
SqlSessionFactory build(InputStream inputStream, String env, Properties props)
SqlSessionFactory build(Configuration config)
```

### 构建SqlSessionFactory
构建SqlSessionFactory有两种方法，一种是通过XML，一种不通过XML。
通过XML构建的方法是通过InputStream构建：
```
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```
不通过XML的方式，则是直接shiyongjava代码创建配置：
```
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

### 从 SqlSessionFactory 中获取 SqlSession
构建了SqlSessionFactory之后，就可以从中获得SqlSession实例。SqlSession是执行SQL语句的接口。
以下就是从BlogMapper中执行selectBlog方法，并传入参数101。
```
try (SqlSession session = sqlSessionFactory.openSession()) {
  Blog blog = (Blog) session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
}
```
但是以上方法有个缺点，就是有可能有的值会被强制类型转换，因此有另外一种方法
```
try (SqlSession session = sqlSessionFactory.openSession()) {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
}
```
我们可以通过BlogMapper.class这个类描述返回值的类型。

### Mapper
#### 增删改查
映射文件是执行SQL语句的关键，这个文件包含了SQL语句，以及返回的类型声明。Mapper有自己的语法，如常见的增删改查：
```
<select 
  id="selectPerson"
  parameterType="int"
  parameterMap="deprecated"
  resultType="hashmap"
  resultMap="personResultMap"
  flushCache="false"
  useCache="true"
  timeout="10"
  fetchSize="256"
  statementType="PREPARED"
  resultSetType="FORWARD_ONLY">
  SELECT * FROM PERSON WHERE ID = #{id}
</select>

<insert 
  id="insertAuthor"
  parameterType="domain.blog.Author"
  flushCache="true"
  statementType="PREPARED"
  keyProperty=""
  keyColumn=""
  useGeneratedKeys=""
  timeout="20">
  insert into Author (id,username,password,email,bio)
  values (#{id},#{username},#{password},#{email},#{bio})
</insert>

<update 
  id="updateAuthor"
  parameterType="domain.blog.Author"
  flushCache="true"
  statementType="PREPARED"
  timeout="20">
  update Author set
    username = #{username},
    password = #{password},
    email = #{email},
    bio = #{bio}
  where id = #{id}
</update>

<delete
  id="deleteAuthor"
  parameterType="domain.blog.Author"
  flushCache="true"
  statementType="PREPARED"
  timeout="20">
  delete from Author where id = #{id}
</delete>
```
可以看到除了有具体的SQL语句（SELECT * FROM PERSON WHERE ID = #{id}）外，还能为语句配置各种属性，如id，传入的参数类（parameterType），传出的参数类（resultType）等。
#### SQL代码段重用
sql修饰被重用的代码，然后在其他语句中使用include引入：
```
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>

<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns"><property name="alias" value="t1"/></include>,
    <include refid="userColumns"><property name="alias" value="t2"/></include>
  from some_table t1
    cross join some_table t2
</select>
```
#### resultMap结果映射
resultMap描述的就是SQL语句执行完毕之后，返回数据的样子。比如最简单的：
```
<select id="selectUsers" resultType="map">
  select id, username, hashedPassword
  from some_table
  where id = #{id}
</select>
```
由于没有显式指定resultMap，这里将映射到HashMap。
如果需要指定resultMap，需要先定义结果类：
```
package com.someapp.model;
public class User {
  private int id;
  private String username;
  private String hashedPassword;

  public int getId() {
    return id;
  }
  public void setId(int id) {
    this.id = id;
  }
  public String getUsername() {
    return username;
  }
  public void setUsername(String username) {
    this.username = username;
  }
  public String getHashedPassword() {
    return hashedPassword;
  }
  public void setHashedPassword(String hashedPassword) {
    this.hashedPassword = hashedPassword;
  }
}
```
这个类定义了3个属性，id，username，hashedPassword，类型分别是int，string，string。
这时候我们就可以在映射文件里定义resultMap：
```
<select id="selectUsers" resultType="com.someapp.model.User">
  select id, username, hashedPassword
  from some_table
  where id = #{id}
</select>
```
我们也可以给结果类定义一个别名，在使用的时候就可以减少些这么长的类名：
```
<!-- mybatis-config.xml 中 -->
<typeAlias type="com.someapp.model.User" alias="User"/>

<!-- SQL 映射 XML 中 -->
<select id="selectUsers" resultType="User">
  select id, username, hashedPassword
  from some_table
  where id = #{id}
</select>
```
如果表的column名和类的属性名不一致，可以用as来确定匹配关系：
```
<select id="selectUsers" resultType="User">
  select
    user_id             as "id",
    user_name           as "userName",
    hashed_password     as "hashedPassword"
  from some_table
  where id = #{id}
</select>
```
也可以使用外部的resultMap。通过单独定义一个resultMap来确定表的column名和类的属性名的对应关系，在sql语句中直接引用这个resultMap
```
<resultMap id="userResultMap" type="User">
  <id property="id" column="user_id" />
  <result property="username" column="user_name"/>
  <result property="password" column="hashed_password"/>
</resultMap>

<select id="selectUsers" resultMap="userResultMap">
  select user_id, user_name, hashed_password
  from some_table
  where id = #{id}
</select>
```

#### 动态 SQL
动态SQL的作用就是根据不同的条件，拼接出不同的SQL语句，实现不同的需求。比如通过用户名查找用户信息，或者通过用户id查找用户信息，两者可以整合到一个动态SQL语句中，如果传入的是用户名，则通过用户名查找，如果传入的是用户id，则通过id查找。
动态SQL主要有四个语句：
+ if
+ choose (when, otherwise)
+ trim (where, set)
+ foreach

##### if
```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
```

##### choose, when, otherwise
```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <choose>
    <when test="title != null">
      AND title like #{title}
    </when>
    <when test="author != null and author.name != null">
      AND author_name like #{author.name}
    </when>
    <otherwise>
      AND featured = 1
    </otherwise>
  </choose>
</select>
```

##### trim, where, set
trim处理的情况是，加入所有的where后面的条件都在<if>里，加入所有的条件都不满足，就会出现错误语法。
```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE
  <if test="state != null">
    state = #{state}
  </if>
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
SELECT * FROM BLOG
WHERE
```
因此通过<where>，where 元素只会在至少有一个子元素的条件返回 SQL子句的情况下才去插入“WHERE”子句。而且，若语句的开头为“AND”或“OR”，where元素也会将它们去除。
```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG
  <where>
    <if test="state != null">
         state = #{state}
    </if>
    <if test="title != null">
        AND title like #{title}
    </if>
    <if test="author != null and author.name != null">
        AND author_name like #{author.name}
    </if>
  </where>
</select>
```

##### foreach
foreach一般是在使用IN语句的时候，对一个集合进行遍历的时候使用，可以指定一个集合，声明可以在元素体内使用集合项（item）和索引（index）。
```
<select id="selectPostIn" resultType="domain.blog.Post">
  SELECT *
  FROM POST P
  WHERE ID in
  <foreach item="item" index="index" collection="list"
      open="(" separator="," close=")">
        #{item}
  </foreach>
</select>
```

## SQL类
SQL类提供了使用java语言生成SQL语句的方法。
```
private String selectPersonSql() {
  return new SQL() {{
    SELECT("P.ID, P.USERNAME, P.PASSWORD, P.FULL_NAME");
    SELECT("P.LAST_NAME, P.CREATED_ON, P.UPDATED_ON");
    FROM("PERSON P");
    FROM("ACCOUNT A");
    INNER_JOIN("DEPARTMENT D on D.ID = P.DEPARTMENT_ID");
    INNER_JOIN("COMPANY C on D.COMPANY_ID = C.ID");
    WHERE("P.ID = A.ID");
    WHERE("P.FIRST_NAME like ?");
    OR();
    WHERE("P.LAST_NAME like ?");
    GROUP_BY("P.ID");
    HAVING("P.LAST_NAME like ?");
    OR();
    HAVING("P.FIRST_NAME like ?");
    ORDER_BY("P.ID");
    ORDER_BY("P.FULL_NAME");
  }}.toString();
}
```


## Reference
https://mybatis.org/mybatis-3/zh/index.html
