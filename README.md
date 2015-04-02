mybatis-generator生成代码隔离演示
======================================

#为什么要用它
1.自定义代码与自动生成的代码隔离，方便维护(特别是增删表字段的时候)
2.自动生成的代码不需要再受版本控制(如果编译后的代码也要受版本控制，那要让mybatis-generator生成的注释不包含日期，不然每次编译后的自动生成代码都不相同，可以通过<commentGenerator><property name="suppressDate" value="true" /></commentGenerator>实现)
3.如果只使用xml隔离，对运行时性能几乎没有影响，因为xml是在启动阶段加载的；而类隔离对运行时唯一的影响就是多了一些父类和父接口.

#原理
通过两步实现代码隔离，这两步可以单独使用，也可以组合使用.
1.使用IsolateParentPlugin隔离自动生成的代码和手动添加的代码(model和client)
	在生成代码时添加父类和父接口，自定义的属性和接口方法直接在父类和父接口中添加.
	如自定义的User类:
```java
package com.example.idomain;
public abstract class User {
	protected List<com.example.domain.Role> roles;
	public List<com.example.domain.Role> getRoles() {
		return roles;
	}
	public void setRoles(List<com.example.domain.Role> roles) {
		this.roles = roles;
	}
}
```
而自动生成的User类:
```java
public class User extends com.example.idomain.User {
    /**
     * This field was generated by MyBatis Generator.
     * This field corresponds to the database column user.id
     *
     */
    private Long id;
	...
}
```
接口类似
2.使用com.example.mybatis中的IsolateXxx类使mybatis能加载隔离的xml文件
	通过配置路径替换规则(这里只是简单地按字符串替换)自动载入自定义的xml.
	通过XML元素的标签名称和id判断是否重复，如果重复则覆盖自动生成的元素块，否则将自定义的元素块追加到加载的XML文件中.
	如：假设isolateReplacement=mapper|imapper, 自动生成的xml为com/example/mapper/User.xml, 我们自定义了一个xml为com/example/imapper/User.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.UserMapper">
	<resultMap id="BaseResultMap" type="com.example.domain.User" >
    <id column="u_id" property="id" jdbcType="BIGINT" />
    <result column="u_name" property="name" jdbcType="VARCHAR" />
    <result column="u_age" property="age" jdbcType="INTEGER" />
    <result column="u_address" property="address" jdbcType="VARCHAR" />
    <result column="u_email" property="email" jdbcType="VARCHAR" />
	</resultMap>
	<resultMap id="ResultMapWithRoles" type="com.example.domain.User"
		extends="BaseResultMap">
		<association column="u_id" property="roles" select="com.example.mapper.RoleMapper.selectByUserId"/>
	</resultMap>

	<select id="selectByPrimaryKeyWithRoles" parameterType="java.lang.Long"
		resultMap="ResultMapWithRoles">
		select
		<include refid="Base_Column_List" />
		from user u where u.id = #{id}
	</select>
</mapper>
```
	由于BaseResultMap在自动生成的xml中已存在，所以它会覆盖自动生成的BaseResultMap，而ResultMapWithRoles和selectByPrimaryKeyWithRoles则会被加到载入的xml中
	

#使用方法(首先复制com.example.mybatis目录下的四个IsolateXxx类)
如果你是使用spring配置创建sqlSessionFactory的方式，使用以下配置即可实现代码隔离
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    ...
    <property name="sqlSessionFactoryBuilder">
		<bean class="com.example.mybatis.IsolateSqlSessionFactoryBuilder">
			<property name="isolateReplacement" value="mapper|imapper"/>
		</bean>
	</property>
</bean>
```

如果是程序创建sqlSessionFactory的方式，则可以使用如下方法
```java
IsolateSqlSessionFactoryBuilder sqlSessionFactoryBuilder = new IsolateSqlSessionFactoryBuilder();
sqlSessionFactoryBuilder.setIsolateReplacement("mapper|imapper");
sqlSessionFactory = sqlSessionFactoryBuilder.build(configInputStream, props);
```


#附带的PaginationSqlXxx是什么？
mybatis的rowBounds实现的分页是查出所有记录，再使用游标分页，数据量大的时候不可取，而使用拦截器的方式也不够灵活
此方式仍保留mybatis的rowBounds控制是否分页的灵活性，只是在处理SqlSource时根据RowBounds参数将分页SQL加到SqlSource中，支持如mysql、oracle等

如果不使用这个分页方式，只需要将com.example.mybatis.IsolateSqlSessionFactoryBuilder中第106行修改如下即可:
return new PaginationSqlSessionFactory(config);==>return new SqlSessionFactory(config);
