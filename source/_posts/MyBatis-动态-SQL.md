---
title: MyBatis 动态 SQL
date: 2021-10-07 22:37:33
categories: [MyBatis]
tags: [MyBatis, 后端开发框架学习]
---

**MyBatis 的动态 SQL 功能**可以帮助我们根据不同条件拼接 SQL 语句，并自动处理 SQL 语法，动态 SQL 功能通过 OGNL(Object-Graph Navigation Language) 表达式和以下几个标签实现，下方详细介绍。

<!--more-->

首先列出本文涉及到的数据表 DDL、entity 对象和 Mybatis 基本配置。

-   DDL （源自 MySQL 官方演示用的 world_x ，简单修改字段名更符合我们熟知的开发规范）

    ```sql
    CREATE TABLE country
    (
        primary_code   VARCHAR(3)  DEFAULT '' NOT NULL PRIMARY KEY,
        country_name   VARCHAR(52) DEFAULT '' NOT NULL,
        capital        INT                    NULL,
        secondary_code VARCHAR(2)  DEFAULT '' NOT NULL
    );
    ```

-   Country 对象

    ```java
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public class Country implements Serializable {

        /** 主要国家代码 */
        private String primaryCode;

        /** 国家名 */
        private String countryName;

        /** 首都ID */
        private Integer capital;

        /** 次要国家代码 */
        private String secondaryCode;

        private static final long serialVersionUID = 1L;

    }
    ```

-   Mapper 接口大致内容：

    ```java
    package pers.cncsl.ft.mybatis.mapper;

    import org.apache.ibatis.annotations.Param;
    import org.springframework.stereotype.Repository;
    import pers.cncsl.ft.mybatis.entity.Country;

    import java.util.List;
    import java.util.Map;
    import java.util.Set;

    /**
     * 国家Mapper
     */
    @Repository
    public interface CountryMapper {
    	//...
    }
    ```

-   mapper.xml 文件大致内容：

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="pers.cncsl.ft.mybatis.mapper.CountryMapper">

        <resultMap id="BaseResultMap" type="pers.cncsl.ft.mybatis.entity.Country">
            <id column="primary_code" jdbcType="VARCHAR" property="primaryCode"/>
            <result column="country_name" jdbcType="VARCHAR" property="countryName"/>
            <result column="capital" jdbcType="INTEGER" property="capital"/>
            <result column="secondary_code" jdbcType="VARCHAR" property="secondaryCode"/>
        </resultMap>

        <!--其他 MyBatis 标签-->

    </mapper>
    ```

# if

if 是最基本的动态 SQL 标签，其 test 属性是一个用于判断输入参数的 OGNL 表达式，当条件判断为真时会拼接其中的内容。与编程语言不同，mybatis 没有 else 标签，if 标签仅用于最基本的条件判断，如果有多个并列的条件需要连续使用相应数量的 if 标签。例如根据主要国家代码或（和）国家名搜索：

```java
/**
* 根据主键或国家名查询一条记录
*
* @param primaryKey 主键
* @param countryName 国家名
* @return 查询结果
*/
Country selectByCodeOrName(@Param("primaryKey") String primaryKey, @Param("countryName") String countryName);
```

```xml
<select id="selectByCodeOrName" resultMap="BaseResultMap">
    SELECT primary_code, country_name, capital, secondary_code
    FROM country
    WHERE 1 = 1
    <if test="primaryCode != null">
        and primary_code = #{primaryCode, jdbcType=VARCHAR}
    </if>
    <if test="countryName != null">
        and country_name = #{countryName, jdbcType=VARCHAR}
    </if>
</select>
```

如下三次调用：

```java
Country resultByCode = mapper.selectByCodeOrName("CHN", null);
Country resultByName = mapper.selectByCodeOrName(null, "China");
Country resultByCodeAndName = mapper.selectByCodeOrName("CHN", "China");
```

从 Mybatis 的日志中可以看到分别执行的 SQL 语句是：

```sql
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE 1 = 1 and primary_code = ?
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE 1 = 1 and country_name = ?
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE 1 = 1 and primary_code = ? and country_name = ?
```

# choose

choose 标签的应用场景是**多个条件选择一个**，类似编程语言的 switch 语法。它有 when 和 otherwise 两个子标签：

-   when 子标签和 if 类似，test 属性用于判断 OGNL 表达式，一个 otherwise 标签内可以有任意数量、只会命中第一个 test 属性为真的。
-   otherwise 子标签类似 switch 的 default，如果前方所有的 when 子标签条件判断都为假，才会拼接上 otherwise 子标签里的内容。

例如上面的例子可以进一步改造为（接口内容不变）：

```xml
<select id="selectByCodeOrName" resultMap="BaseResultMap">
    SELECT primary_code, country_name, capital, secondary_code
    FROM country
    WHERE 1 = 1
    <choose>
        <when test="primaryCode != null">
            and primary_code = #{primaryCode, jdbcType=VARCHAR}
        </when>
        <otherwise>
            and country_name = #{countryName, jdbcType=VARCHAR}
        </otherwise>
    </choose>
</select>
```

和之前同样的参数再调用三次，执行的 SQL 语句分别是：

```sql
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE 1 = 1 and primary_code = ?
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE 1 = 1 and country_name = ?
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE 1 = 1 and primary_code = ?
```

# trim

trim 标签用于避免使用 if 或 choose 时为了 SQL 语法的正确性而不得不插入类似 `1=1 and` 这种语句，保证生成的 SQL 更干净、优雅，where 和 set 标签是 trim 的一种具体用法。

## where

如果 where 标签内的其他标签有返回值，就在标签位置插入一个 where，并剔除后续字符串开头的 AND 和 OR。

在之前的两个例子中，为了保证生成正确的 SQL，我们加入了 `WHERE 1=1` 这样的代码段，可以通过 where 标签来优化（接口内容不变）：

```xml
<select id="selectByCodeOrName" resultMap="BaseResultMap">
    SELECT primary_code, country_name, capital, secondary_code
    FROM country
    <where>
        <choose>
            <when test="primaryCode != null">
                and primary_code = #{primaryCode, jdbcType=VARCHAR}
            </when>
            <otherwise>
                and country_name = #{countryName, jdbcType=VARCHAR}
            </otherwise>
        </choose>
    </where>
</select>
```

改造后仍然用之前的参数调用，执行的 SQL 语句分别是：

```sql
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE primary_code = ?
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE country_name = ?
SELECT primary_code, country_name, capital, secondary_code FROM country WHERE primary_code = ?
```

可以看出，MyBatis 已经帮我们在生成的 SQL 中插入了 WHERE。

## set

如果 set 标签内的其他标签有返回值，就在标签位置插入一个 set，并剔除后续字符串结尾处的逗号。

很常见的需求是仅更新数据表中的指定字段，这些字段需要通过 if 标签处理，而 set 标签用于插入一个 SET 关键字：

```java
/**
* 根据主键和入参的其他数据更新一条记录，入参为 null 的字段不变更
*
* @param entity 入参数据
* @return 更新结果
*/
int updateByPrimaryKeySelective(Country entity);
```

```xml
<update id="updateByPrimaryKeySelective" parameterType="pers.cncsl.ft.mybatis.entity.Country">
    UPDATE country
    <set>
        <if test="countryName != null">
            country_name = #{countryName, jdbcType=VARCHAR},
        </if>
        <if test="capital != null">
            capital = #{capital, jdbcType=INTEGER},
        </if>
        <if test="secondaryCode != null">
            secondary_code = #{secondaryCode, jdbcType=VARCHAR},
        </if>
    </set>
    WHERE primary_code = #{primaryCode, jdbcType=VARCHAR}
</update>
```

如下调用：

```Java
Country update = Country.builder().primaryCode(PRIMARY_CODE).countryName("中国").build();
int rows = mapper.updateByPrimaryKeySelective(update);
```

执行的 SQL 为：

```sql
UPDATE country SET country_name = ? WHERE primary_code = ?
```

Mybatis 帮我们插入了 SET 关键字、并删除了后续多余的逗号。

## trim 属性详解

trim 标签有以下四个属性：

-   prefix：当 trim 内有内容时，会给内容增加 prefix 指定的前缀
-   prefixOverrides：当 trim 内有内容时，会将内容中匹配的前缀字符串取掉
-   suffix：当 trim 内有内容时，会给内容增加 suffix 指定的后缀
-   suffixOverrides：当 trim 内有内容时，会将内容中匹配的后缀字符串取掉

所以 where 标签和 set 标签实际相当于：

```xml
<trim prefix="WHERE" prefixOverrides="AND | OR" > ... </trim>

<trim prefix="SET" suffixOverrides="," > ... </trim>
```

# foreach

顾名思义，foreach 标签用于实现集合遍历，它有以下属性：

-   collection：要迭代遍历的属性名。
-   item：变量名，每次迭代时从集合中取出的值的名称。
-   index：索引值，对 List 集合值为当前索引值，对 Map 集合值为当前 key。
-   open：整个循环内容开头处添加的字符串。
-   close：整个循环内容结尾处添加的字符串。
-   separator：每次循环的分隔符。

例如根据主键批量查询：

```java
/**
* 根据入参的主键List集合批量查询，使用 List 较为简单。
*
* @param keys 主键List集合
* @return 查询结果
*/
List<Country> selectByPrimaryCodeList(List<String> keys);
```

```xml
<select id="selectByPrimaryCodeList" parameterType="java.util.Collection"
        resultType="pers.cncsl.ft.mybatis.entity.Country">
    SELECT primary_code, country_name, capital, secondary_code
    FROM country
    <where>
        <if test="list != null and !list.isEmpty()">
            primary_code IN
            <foreach collection="list" open="(" close=")" separator="," item="code">
                #{code}
            </foreach>
        </if>
    </where>
</select>
```

注意上面的 where、if 和 foreach 三种标签的组合使用，能避免入参错误导致 MyBatis 拼接出错误的 SQL 语句，不过入参为 null 或空集合时会查出全表。编码时对于这种需求应该小心对待，尽量不传入 null 或空集合。

collection 属性的值是 Mybatis 中的参数名，通常用 `@Param` 注解指定，不指定时 Mybatis 会对不同类型的参数添加一个默认的名称，数组和集合有关的默认参数名规则如下：

-   数组类型 `array`
-   Collection 集合（java.util.Collection 的实现类）都为 collection
-   List 也可用 list

仔细下方内容，重点在于 Mapper 接口中函数入参为数组、List 集合和 Set 集合时 xml 文件中 foreach 标签的属性和内容：

```java
/**
* 根据入参的主键数组批量查询，使用数组是最基础的情况，使用场景不多。
*
* @param keys 主键数组
* @return 查询结果
*/
List<Country> selectByPrimaryArray(String[] keys);

/**
* 根据入参的主键List集合批量查询，使用 List 较为简单。
*
* @param keys 主键List集合
* @return 查询结果
*/
List<Country> selectByPrimaryCodeList(List<String> keys);

/**
* 根据入参的主键Set集合批量查询，使用 Set 可以保证集合元素唯一。
*
* @param keys 主键Set集合
* @return 查询结果
*/
List<Country> selectByPrimaryCodeSet(Set<String> keys);
```

```xml
<select id="selectByPrimaryArray" parameterType="java.util.Collection"
        resultType="pers.cncsl.ft.mybatis.entity.Country">
    SELECT primary_code, country_name, capital, secondary_code
    FROM country
    <where>
        <if test="array != null and array.length gt 0">
            primary_code IN
            <foreach collection="array" open="(" close=")" separator="," item="code">
                #{code}
            </foreach>
        </if>
    </where>
</select>

<select id="selectByPrimaryCodeList" parameterType="java.util.Collection"
        resultType="pers.cncsl.ft.mybatis.entity.Country">
    SELECT primary_code, country_name, capital, secondary_code
    FROM country
    <where>
        <if test="list != null and !list.isEmpty()">
            primary_code IN
            <foreach collection="list" open="(" close=")" separator="," item="code">
                #{code}
            </foreach>
        </if>
    </where>
</select>

<select id="selectByPrimaryCodeSet" parameterType="java.util.Collection"
        resultType="pers.cncsl.ft.mybatis.entity.Country">
    SELECT primary_code, country_name, capital, secondary_code
    FROM country
    <where>
        <if test="collection != null and !collection.isEmpty()">
            primary_code IN
            <foreach collection="collection" open="(" close=")" separator="," item="code">
                #{code}
            </foreach>
        </if>
    </where>
</select>
```

foreach 还可实现批量插入和动态更新功能，不过这两个功能在开发中用到的场景不多，就不详细说了。
