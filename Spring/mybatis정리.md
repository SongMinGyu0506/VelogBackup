## MyBatis란?
마이바티스는 개발자가 지정한 SQL, 저장프로시저 그리고 몇가지 고급 매핑을 지원하는 퍼시스턴스 프레임워크이다. 마이바티스는 JDBC로 처리하는 상당부분의 코드와 파라미터 설정및 결과 매핑을 대신해준다. 마이바티스는 데이터베이스 레코드에 원시타입과 Map 인터페이스 그리고 자바 POJO 를 설정해서 매핑하기 위해 XML과 애노테이션을 사용할 수 있다.  (MyBatis 공식 문서)  

즉, 한줄로 요약하자면 기존의 JDBC 이용 방식의 SQL 사용의 불편함을 해소하기 위해 파라미터 설정, 결과를 대신 처리해주는 프레임 워크.

## 사용 방법?
**Maven, Spring MVC 기준**  
### 1. pom.xml 의존성 등록(MyBatis 설치)
```xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>x.x.x</version>
</dependency>
```
### 2. SqlSessionFactory 빌드
* 해당 단계는 데이터베이스의 드라이버, url, 계정정보를 등록한다. 트랜잭션 제어를 위한 등록이나, hikariCP를 이용한 커넥션 풀을 만들어서 이용하였다.

```xml
<!-- root-context.xml -->
<bean id="hikariConfig" class="com.zaxxer.hikari.HikariConfig">
		<!-- <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"></property> -->
		<property name="driverClassName" value="net.sf.log4jdbc.sql.jdbcapi.DriverSpy"></property> 
        <!--  사용하는 DB 드라이버, 여기서는 log4jdbc를 이용하기 위해 해당 드라이버 사용 -->
		<property name="jdbcUrl" value="DB URL"></property>
		<property name="username" value="ID"></property>
		<property name="password" value="PW"></property>
	</bean>
	
	<bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource" destroy-method="close">
		<constructor-arg ref="hikariConfig"/>
	</bean>
	
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource"></property>
	</bean>	

    <mybatis-spring:scan base-package="mapper 패키지 이름"/>
```
해당 xml 코드를 정리하면, hikariConfig Bean의 정보를 dataSource Bean에 주입, 그리고 dataSource Bean의 정보를 sqlSessionFactory Bean에 주입한다.  
그리고 MyBatis 어노테이션을 찾기 위해서 스캔 대상이 될 패키지를 선택한다.  

그렇다면 해당 패키지에서 어노테이션으로 mapper를 만들어서 사용하든, xml 파일로 따로 만들어서 사용하여 SQL을 따로 관리할 수 있게 된다. 하지만 어노테이션 방식은 간단한 쿼리문에서는 유용하나 복잡할수록 관리에 문제가 생기므로 XML을 권장한다.  

### 3. Mapper XML 파일 생성
우선 XML 파일과 매칭될 인터페이스를 만든다.
```java
package org.zerock.mapper;

import java.util.List;

import org.zerock.domain.BoardVO;
import org.zerock.domain.Criteria;

public interface BoardMapper {
	public List<BoardVO> getList();
	public void insert(BoardVO board);
	public BoardVO read(int bno);
	public int delete(int bno);
	public int update(BoardVO board);
	public List<BoardVO> getListPaging(Criteria cri);
	public int getTotal(Criteria cri);
}
```
해당 인터페이스는 
게시판 CRUD를 이용하기 위한 인터페이스이다.
```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.zerock.mapper.BoardMapper">
    <select id="getList" resultType="org.zerock.domain.BoardVO">
        <![CDATA[select * from tbl_board where bno > 0]]>
    </select>
    <insert id="insert">
        insert into tbl_board (title,content,writer) values(#{title},#{content},#{writer})
    </insert>
    <select id="read" resultType="org.zerock.domain.BoardVO">
        select * from tbl_board where bno = #{bno}
    </select>
    <delete id="delete">
        delete from tbl_board where bno = #{bno}
    </delete>
    <update id="update">
        update tbl_board
        set title = #{title}, content=#{content},writer=#{writer},updateDate=CURRENT_TIMESTAMP
        where bno = #{bno};
    </update>
    <sql id="criteria">
    	<trim prefix="where (" suffix=")" prefixOverrides="OR">
    		<foreach collection="typeArr" item="type">
    			<trim prefix="OR">
    				<choose>
    					<when test="type == 'T'.toString()">
    						title like concat('%',#{keyword},'%')
    					</when>
    					<when test="type == 'C'.toString()">
    						content like concat('%',#{keyword},'%')
    					</when>
    					<when test="type == 'W'.toString()">
    						writer like concat('%',#{keyword},'%')
    					</when>
    				</choose>
    			</trim>
    		</foreach>
    	</trim>
    </sql>
    ...
</mapper>
```
이러한 방식으로 Mapper XML 파일을 작성해주면 되는데, 주의 해야할 점은
* mapper xml 파일의 위치, 인터페이스와 XML 파일은 분리하여 관리하는 것이 유지보수에 좋다. 보통 여러 도서나 사용자들의 관리 위치는 main > resources > 패키지 이름과 같은 폴더 생성 후 관리
* mapper 태그에서 namespace의 경우, 인터페이스의 위치를 나타낸다. 그러므로 정확한 위치를 작성해야한다.

## Mapper 태그 설명
### 속성
* **id** => Namespace 내의 구분자, 즉, 인터페이스에 작성한 메소드와 같다면 매칭이 된다.
* **parameterType** => 구문에 전달될 파라미터의 패키지 경로를 포함한 전체 클래스명이나 별칭
* **resultType** => 이 구문에 의해 리턴되는 기대타입의 패키지 경로를 포함한 전체 클래스 명이나 별칭
* 파라미터의 경우 #{이름} 으로 사용한다. 예로 BoardVO를 파라미터로 받은 경우 BoardVO 객체 내부의 속성값들을 사용할 수 있다.

### 각 태그
* select, delete, insert, update 태그는 SQL의 CRUD와 같다.

### sql, include 태그
* 일반 프로그래밍에서 프로시저와 같은 역할을 한다. sql 태그 내부에서 작성된 것들을 id 속성의 이름으로 저장하고 있다가, include refid로 참조하여 사용한다.


## 동적 SQL
특정 상황일 때, 해당 상황에 맞는 SQL을 사용할 수 있도록 용이하게 만든다.
### if
```xml
<select id="findActiveBlogWithTitleLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
</select>
```
test 내부의 조건을 충족시킬경우 if 태그내의 SQL을 동작시킨다.

### trim, where
* where 태그 : 수행되는 역할은 SQL의 where과 다른게 없으나, 만약 if문 내에서 where 바로 다음 and를 처리해야한다면? 해당 경우와 같이 문법 오류를 배제하기 위해서 where 태그를 사용한다.
* trim : suffix/prefix, suffixOverrides/prefixOverrides 태그를 이용하여 trim 태그 앞 뒤에 문자를 지우거나, 추가한다.
  * suffix/prefix => trim 태그 앞/뒤에 문자 추가
  * Overrides => trim 태그 앞/뒤에 해당 문자 존재할 경우 해당 문자 삭제

### foreach
흔히 사용하는 반복문을 사용하기 위해서 사용하는 태그  
foreach에 입력받는 형태는 List, Array 형태만 받는다.
```xml
<foreach collection="ListName" item="attr" open="(" close=")" separator="or">
#{attr.value} // #{attr[index]}
</foreach>
```
* collection => 전달받은 파라미터
* item => 전달받은 파라미터의 이명
* open => 해당 구문 시작시 삽입할 문자열
* close => open 반대
* separator => 1회 실행시 각 구간마다 출력할 문자열
* index => 반복되는 구문번호 0부터 순차적으로 증가


