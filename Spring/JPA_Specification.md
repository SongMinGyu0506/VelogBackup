# JPA Specification
## 기존의 조회쿼리
우리는 데이터베이스를 사용하여 조회쿼리를 수행할 때 정말 다양한 조건들을 적용하여 사용하고 있다.
또한, 특정한 상황에서 몇몇 조건을 추가하거나, 사용하지 않는 일 또한 존재한다. 
만약 JPA에서 조건에 맞는 쿼리를 작성하려면 조건이 많아질수록 작성해야할 쿼리 또한 기하급수적으로 늘어날 수 밖에 없다.

예를 들어 게시글의 조회 기능을 구현한다고 할 때, 게시글 타입, 작성자, 제목으로 검색한다고 가정해보자.
먼저 각각의 단일 조건의 검색 쿼리 3개, 그리고 두개의 조건을 이용한 검색 쿼리 3개, 전체 조건을 이용한 쿼리 1개, 조건 없는 조회 1개가 필요하다.
```sql
select board_type from board;
select writer from board;
select title from board;
select board_type, writer from board;
select board_type, title from board;
select writer, title from board;
select board_type, writer, title from board;
select * from board;
...
```
3개의 조건을 이용하였는데도 많은 쿼리가 필요하다. 상당히 비효율적이라고 생각하고 있다.

JPA를 사용하는 개발환경의 경우 JPA Specification을 이용한다면 어느정도 해결 할 수 있다.

## JPA Specification을 이용하여 해결
### JPA Specification이란?
JPA에서 동적 쿼리를 생성하기위한 방법 중 하나이다. 일종의 검색 조건을 추상화한 인터페이스로, 조회 기능을 구현 할 때 Specification 인터페이스를 구현한다면 원하는 조회 기능을 정의 할 수 있다.

### 구현
#### Repository
실제 내가 개발에 적용 했을때는 아래와 같은 방식을 사용했다.
```java
@Repository
public interface BoardRepository extends JpaRepository<Board,Integer>, JpaSpecificationExecutor<Board> {
	List<Board> findAll(Specification<Board> keyword);
    ...
}
```
Repository 인터페이스에서는 ```JpaSpecificationExecutor``` 인터페이스를 상속한다.
여기서 ```JpaSpecificationExecutor``` 인터페이스는 `List` 또는 `Page` 형태로 반환하고 `Specification<T>`를 파라미터로 받는 findAll 메소드를 지원하고 있다.
#### Specification
```java
public class BoardSpec {
    public static Specification<Board> equalType(String type) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.equal(root.get("boardType"),type);
    }
    public static Specification<Board> equalAuthor(String author) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.equal(root.get("member").get("name"),author);
    }
    public static Specification<Board> likeTitle(String title) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.like(root.get("title"),"%"+title+"%");
    }
    public static Specification<Board> equalIsAdd(boolean isAdd) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.equal(root.get("isAdd"),isAdd);
    }
}
```
해당 소스코드는 인터페이스 메소드의 구현을 람다식으로 표현되어있다. 
반환 메소드의 `(root, query, criteriaBuilder)`를 이용하여 쿼리를 만들어 낼 수 있다.
각 파라미터가 뭘 의미하는지 찾아보면
* `root` : `javax.persistence.criteria.Root` 타입의 매개변수, 엔티티의 루트(root)를 나타낸다. 엔티티의 루트는 JPA Criteria 쿼리의 시작점이며, 쿼리의 FROM 절에 해당하는 엔티티에 해당, 이 root 객체를 사용하여 엔티티의 속성에 접근할 수 있다.
* `query` : `javax.persistence.criteria.CriteriaQuery` 타입의 매개변수로, JPA Criteria 쿼리를 나타낸다. 이 query 객체는 JPA 쿼리의 메타데이터 정보를 포함 주로 쿼리의 제약조건(WHERE 절)을 정의
* `criteriaBuilder` : `javax.persistence.criteria.CriteriaBuilder` 타입의 매개변수로, JPA Criteria API를 사용하여 쿼리 조건을 생성하는데 사용되는 빌더 객체, `criteriaBuilder` 객체는 `equal()`, `greaterThan()`, `like()` 등의 메서드를 제공하여 쿼리 조건을 생성
#### 조회 기능 구현
```java
private List<Board> getBoards(String type, String title, String author, Specification<Board> spec) {
        spec = spec.and(BoardSpec.equalType(type));
        if (title != null) {
            spec = spec.and(BoardSpec.likeTitle(title));
        } else if (author != null) {
            spec = spec.and(BoardSpec.equalAuthor(author));
        }
        return boardRepository.findAll(spec);
    }
```
기능 구현에서는 `Specification<T>` 파라미터를 이용하여 쿼리들을 추가시켜주고, `repository`의 `findAll(Specification<T>)`를 이용하면 동적 쿼리를 사용할 수 있다.
여기서 `and` 메서드를 이용하여 조건들을 차례대로 추가해주면 데이터베이스 쿼리문을 만들지 않고 동적 쿼리만 활용하여 간결한 코드를 작성할 수 있다.

## 마치면서
JPA에서 지원하는 동적쿼리는 Specification 뿐만 아니라 다양한 방법들을 지원하고 있다.
Specification 또한 동적쿼리를 사용하는 방법 중 하나 일뿐 해당 방법이 절대적인 활용방법은 아니다. 프로젝트의 상황에 맞춰서 다양한 동적쿼리 사용 방법을 알아야한다고 생각한다.
