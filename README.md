# framework-research

# JPA Projection

우리는 Spring Data Jpa를 사용하면서 Repository의 쿼리결과를 Entity Object에 Mapping에서 가져옵니다. 하지만 우리는 모든 Entity의 정보를 필요로하지 않을 때가 존재합니다. Entity의 일부 데이터만 필요한 상황이지만 Entity Object로 결과를 return 받는 것은 불필요하다고 느낄 것입니다. 또한 Entity를 response를 위해서 DTO로 변환해야 한다면 추가적인 mapping 과정이 발생하게 됩니다.

Jpa Projection은 위와 같은 상황을 해결해줄 수 있습니다. 원하는 Object를 사용하여 쿼리 결과를 전달받는 것입니다.

Projection에는 총 3가지 종류가 있습니다.

**Interface Projection, Class Projection, Dynamic Projection**입니다. 

종류가 너무 많아보이지만 핵심 역할은 하나입니다. '쿼리 결과를 원하는 Object에 Mapping한다' 입니다

앞으로 나올 예시는 아래 sample code 기반으로 진행됩니다.

```
class Person {

  @Id UUID id;
  String firstname, lastname;
  Address address;

  static class Address {
    String zipCode, city, street;
  }
}

interface PersonRepository extends Repository<Person, UUID> {

  Collection<Person> findByLastname(String lastname);
}
```

### Interface Projection

이름 그대로 Interface 를 사용해서 결과를 가져오는 것입니다.

```
interface NamesOnly {

  String getFirstname();
  String getLastname();
}

interface PersonRepository extends Repository<Person, UUID> {

  Collection<NamesOnly> findByLastname(String lastname);
}
```

위와 같이 사용하면 Person object에서 firstname과 lastname의 정보만 가져올 수 있게됩니다. Query 또한 'select * from person'이 아닌 'select firstname, lastname from person' 으로 나가게 됩니다. 그렇기 때문에 쿼리최적화 효과를 누릴 수 있습니다. 

하지만 이런 요구사항이 있을 수 있습니다. '결과값으로 단순히 entity의 field가 아니라 가공된 값을 가져오고 싶어'

```
interface NamesOnly {

  @Value("#{target.firstname + ' ' + target.lastname}")
  String getFullName();
  …
}
```

위와 같이 여러개의 필드 들을 조합해서 새로운 값을 만들어내는 것도 가능합니다.

```
interface NamesOnly {

  String getFirstname();
  String getLastname();

  default String getFullName() {
    return getFirstname().concat(" ").concat(getLastname());
  }
}
```
또한 default method를 사용해서 다양한 비즈니스 로직을 적용하여 값을 만들어낼 수도 있죠.

그렇다면 relation이 있는 경우는 어떻게 가져올 수 있을까요?

```
interface PersonSummary {

  String getFirstname();
  String getLastname();
  AddressSummary getAddress();

  interface AddressSummary {
    String getCity();
  }
}
```
다음과 같이 연관관계로 가지고 있는 entity의 정보도 함께 가져올 수 있습니다. 이는 Nested Projection이라고 부릅니다. 그런데 궁금한점이 생길 수 있습니다. Interface는 구현체가 필요한 친구인데 Interface만 가지고 어떻게 정보들을 가져올 수 있는것이냐? 그건 Spring이 Interface를 구현한 Proxy object를 따로 내부적으로 생성하여 값들을 리턴해줍니다.

### Class Projection

'나는 interface말고 Controller에서 바로 응답으로 줄 수 있는 DTO에 Mapping하고 싶다' 그때는 Class Projection을 사용하면 됩니다.

사용방법은 Interface Projection과 동일하게 Repository method의 return값으로 class를 넣어주기만 하면 됩니다.

```
record NamesOnly(String firstname, String lastname) {
}

interface PersonRepository extends Repository<Person, UUID> {

  Collection<NamesOnly> findByLastname(String lastname);
}
```

다음과 같이 원하는 DTO를 정의해서 사용하면 됩니다. Class는 하지만 Interface Projection과 달리 relation을 가져오는 것이 불가능합니다. 

### Dynamic Projection

다이나믹 프로젝션은 이름에서 부터 짐작가능한것 처럼 projection을 동적으로 할당할 수 있습니다. 즉 return type에 Generic을 선언하고 Class정보를 받아서 동적으로 설정할 수 있습니다.


```
interface PersonRepository extends Repository<Person, UUID> {

  <T> Collection<T> findByLastname(String lastname, Class<T> type);
}

void someMethod(PersonRepository people) {

  Collection<Person> aggregates =
    people.findByLastname("Matthews", Person.class);

  Collection<NamesOnly> aggregates =
    people.findByLastname("Matthews", NamesOnly.class);
}
```

그러면 위와 같이 동일한 메소드일지라도 원하는 오브젝트에 받아서 가져올 수 있게됩니다. interface, class 모두 파라미터로 사용 가능합니다.


# Json view

우리는 개발을 하면서 수많은 DTO를 생성해냅니다. 비슷한 응답값일지라도 Client요구사항에 맞춰서 응답하기 위해 중복이 많더라도 비슷한 DTO를 많이 생성하게 됩니다. DTO가 늘어난다는 것은 그만큼 관련 Mapping function도 늘어날 것입니다. 그렇다면 DTO를 사용하지 않고 Client에게 원하는 필드만 노출시킬 수 있는 방법은 없는것일까요?

@JsonView를 사용하면 가능합니다.

@JsonView는 Entity Object에서 원하는 field만 직렬화를 가능케합니다. 그렇기 때문에 DTO 없이도 원하는 필드만 노출시킬 수 있습니다. 하지만 하나의 entity를 여러개의 API endpoint에서 사용할때 다양한 종류의 응답이 필요할 것입니다.

예를 들어 A라는 endpoint에서는 name만 B 라는 endpoint에서는 name, address, C라는 endpoint에서는 Address, Cellphonenumber... 이렇게 다양한 요구사항이 존재한다면 어떻게할 수 있을까요?

@JsonView는 여러개의 View 설정을 가능케합니다. 예제를 보면서 설명해드리겠습니다.


```
public class Views {
    public static interface Public {}
    public static interface Internal extends Public {}
}
```

다음과 같이 여러개의 View를 설정할 수 있습니다. 이는 Entity에서 annotation을 사용해서 손쉽게 적용이 가능합니다.

```
public class User {
    @JsonView(Views.Public.class)
    private Long id;

    @JsonView(Views.Public.class)
    private String username;

    @JsonView(Views.Internal.class)
    private String password;

    // getters and setters
}
```

Entity의 필드들은 각각 어떤 View에서 보여질지 선택됩니다. field들은 한개의뷰 뿐만 아니라 여러개의 View에서 보여지도록 설정도 가능합니다. 또한 View를 설정할때는 상속을 통해서 더 편리한 구성이 가능합니다. 현재 Internal은 Public을 상속받고 있으므로 Internal view에서는 Public view로 설정된 필드들 역시 포함이 됩니다.

```
@RestController
public class UserController {

    @GetMapping("/user/{id}")
    @JsonView(Views.Public.class)
    public User getUser(@PathVariable Long id) {
        // 사용자 조회 로직
        return user;
    }

    @GetMapping("/user/{id}/detail")
    @JsonView(Views.Internal.class)
    public User getUserWithDetails(@PathVariable Long id) {
        // 사용자 상세 조회 로직
        return user;
    }
}

```

이는 Controller에서 응답시에 Entity가 어떤 view를 사용해서 직렬화될건지 선택함으로써 Endpoint마다 동일한 Entity를 사용해서 서로다른 응답을 만들어낼 수 있게됩니다.

Controller위에 덕지덕지 어노테이션이 붙는게 verbose하게 느껴지실 수 있습니다. 이를 해결하기 위해 default view를 따로 설정하여 예외 케이스만 따로 view를 바꾸도록 설정할 수 있습니다.




# Graphql

graphql이란? API를 위한 쿼리 언어입니다. 서버측에서 미리 정의해놓은 데이터(type)에 대해 쿼리를 통해 질의를 할 수 있습니다. Graphql은 type과 field, field에 대한 function으로 만들어집니다. GraphQL 서비스를 통해서 누가 로그인된 유저인지 확인하는 예제를 만들어 보겠습니다.

다음과 같이 type을 정의하고 type안에는 여러개의 field들이 존재합니다.
```
type Query {
  me: User
}

type User {
  id: ID
  name: String
}
```

각 type의 field들에 대해서 function을 정의할 수 있습니다.
```
function Query_me(request) {
  return request.auth.user
}

function User_name(user) {
  return user.getName()
}
```
Query라는 type에는 Query_me라는 함수를 정의해서 요청으로부터 인증이 완료된 유저를 return하는 함수를 정의하였고

User라는 type에는 User_name이라는 함수를 정의해서 유저의 이름을 가져오는 function을 정의하였습니다.

```
{
  me {
    name
  }
}
```

다음과 같은 쿼리를 서버에 날린다면 아래와 같은 응답을 받게 됩니다.

```
{
  "me": {
    "name": "Jinwoo Bae"
  }
}
```

### Schemas and Types

GraphQL의 Type 시스템에 대해서 조금 더 깊게 알아보고 Schame라는 개념에 대해서 알아보겠습니다.

이전에 봤던 것처럼 GraphQL 쿼리는 오브젝트의 필드들을 선택하는것을 기본으로 합니다.

```
{
  hero {
    name
    appearsIn
  }
}
```

자 위의 예제코드를 간략하게 표현해보겠습니다.

1. 특별한 'root' object로부터 시작합니다.

2. 우리는 hero라는 field를 선택했습니다.

3. hero라는 오브젝트에서 name과 apperasIn이라는 필드값을 선택했습니다.


GraphQL은 위와 같이 우리가 받게될 응답값을 우리가 예상할 수 있습니다. 즉 Client 측에서 서버 측의 데이터중에서 원하는 데이터를 선택해서 질의할 수 있다는 겁니다. 하지만 여기서 한가지 의문이 생깁니다. 서버에 무슨 데이터가 있는줄 알고 질의를 하나? 어떤 오브젝트가 있고 어떤 필드들이 있는지 어떻게 알수 있나? 또한 해당 필드들의 return type은 무엇인가? (String이냐 object냐).

여기서 바로 Schema라는 개념이 등장합니다.

모든 GraphQL 서비스는 Schema를 통해서 타입에 대한 decription을 정의해놓습니다. 이는 가용가능한 데이터들을 표현합니다. 모든 쿼리가 질의됐을때 해당 쿼리들은 Schema에 의해 validation이 된 이후 실행됩니다.

Schame는 오브젝트 형태로 표현됩니다.

```
type Character {
  name: String!
  appearsIn: [Episode!]!
}
```

위 Schema를 예제로 간단히 설명드리겠습니다. 

Character는 GraphQL Object Type으로 여러개의 필드들을 가지고 있다는 의미입니다. Schema에 있는 대부분의 type은 오브젝트입니다. name과 appearsIn은 필드 이름입니다. Character type에 대해서 쿼리를 사용할때 가져올 수 있는 필드들 입니다.

String은 built-in scalar type입니다. 

String!이 의미하는 것은 non-nullable을 의미합니다.

[Episode!]! 는 Episode라는 Object의 배열형태를 의미합니다. !가 붙어 있으므로 non-nullable을 나타내고 있습니다. 배열 안에는 반드시 하나 이상의 Episode Object가 있어야 하고 배열 안에 있는 모든 Episode는 null이 될 수 없습니다.


# Query and Mutation

Schema에있는 대부분의 타입은 일반적인 Object type이지만 GraphQL에는 특별한 두가지 Type이 존재합니다.

그것은 바로 Query type와 mutation type입니다. 두개의 type은 일반적인 object type과 동일합니다. 하지만 다른 점이 있다면
GraphQL query에 대한 entry point를 정의한다는 점입니다.


```
query {
  hero {
    name
  }
  droid(id: "2000") {
    name
  }
}
```

우리가 GraphQL 서버에 위와 같은 질의를 하려면 Query type이 다음과 같이 정의되어 있어야 합니다.

```
type Query {
  hero(episode: Episode): Character
  droid(id: ID!): Droid
}
```

그렇다면 Query와 Mutation의 차이는 무엇일까요

Query는 읽기전용으로 데이터를 가져오기 위한 type이며 Mutation은 server측의 데이터에 변경을 주고싶을때 사용됩니다.

쿼리의 field들은 parallel하게 실행되지만 mutation의 field들을 series하게 실행됩니다.

만약 우리가 incrementCredits라는 mutation두개를 하나의 request에 보냈다면 첫번째가 먼저 끝나는 것이 확인된 뒤에 두번째 incrementCredits가 실행됩니다. 이는 race condition을 예방하기 위함입니다.

mutation도 쿼리와 마찬가지로 생성된 자원에 대한 필드들을 응답값으로 받을 수 있습니다.

# Spring for graphql

graphql-java위에서 만들어진 Spring application에서 GraphQL을 지원하기 위해 나온 라이브러리입니다. spring graphql은 Schema-first를 지향하기 때문에 Schema를 먼저 선언한뒤에 개발을 진행하게 됩니다.

GraphQL의 schema는 src/main/resources/graphql 폴더 안에 *.graphqls 형태의 파일로 정의내립니다.

직접 간단한 예제를 만들어보겠습니다.

```
type Query {
    bookById(id: ID): Book
}

type Book {
    id: ID
    name: String
    pageCount: Int
    author: Author
}

type Author {
    id: ID
    firstName: String
    lastName: String
}
```

다음과 같은 Schema를 정의합니다. 여기서 전에 나온 개념을 집고 넘어가자면 Query type은 data를 fetch하기 위한 특별한 type입니다. bookById라는 필드는 Argument를 받을 수 있게 되어있고 Id를 argument로 받아서 Id에 해당하는 Book type을 리턴하게 되어 있습니다.

자 API를 만들려면 우선 API로 접근할 수 있는 Endpoint를 구성해야합니다. 기존 Rest같은 경우에는 url로 구분하여 여러개의 endpoint를 구성하였습니다. 하지만 GraphQL에서는 조금 다른 형태를 보입니다.

```
package com.example.graphqlserver;

import org.springframework.graphql.data.method.annotation.Argument;
import org.springframework.graphql.data.method.annotation.QueryMapping;
import org.springframework.graphql.data.method.annotation.SchemaMapping;
import org.springframework.stereotype.Controller;

@Controller
public class BookController {
    @QueryMapping
    public Book bookById(@Argument String id) {
        return Book.getById(id);
    }

    @SchemaMapping
    public Author author(Book book) {
        return Author.getById(book.authorId());
    }
}
```
GraphQL의 Controller는 다음과 같습니다. url로 정의된 endpoint가 없는것이 특징입니다. GraphQL에서는 /graphql 이라는 하나의 endpoint에서 쿼리형태로 다양한 요청을 수용할 수 있기 때문에 url로 endpoint를 나눠주는 행위는 필요하지 않습니다.

GraphQL의 Controller에서 필요한 것은 Query type에 대해서 Spring에서 어떻게 동작할 것인지 정의해주는 함수가 필요합니다. @QueryMapping이라는 어노테이션을 붙여야 하며 parameter로 Schema에 존재했던 argument를 받도록 정의해야합니다.

여기서 method 이름은 query type에 정의되어 있는 field 이름과 같아야합니다.

@SchemaMapping 어노테이션은 hanlder method와 GraphQL schema의 field를 연결시켜주는 역할을 합니다. 그리고 메소드를 해당 필드에 대한 DataFetcher로 사용하겠다고 선언하는 것입니다. 필드 이름은 디폴트로 메소드 이름이 되고 type 이름은 디폴트로 메소드에 주입받는 오브젝트의 이름이 됩니다. 이 예제에서는 author가 필드가 되는 것이고 type은 default로 Book이 되는 것입니다. Schema로 표현해보자면 아래와 같습니다.
```
type Book {
    author: Author
}
```

여기서 DataFetcher란 무엇인가? DataFetcher는 스키마의 어떤필드에 대해서든 가져올 수 있는 로직을 제공하는 기능입니다.

즉 

```
    @SchemaMapping
    public Author author(Book book) {
        return Author.getById(book.authorId());
    }
```
위의 코드는 Schema에서 book을 사용해서 author 필드를 가져올 수 있는 DataFetcher를 선언하는 것으로 이해할 수 있습니다.






































# reference



https://docs.spring.io/spring-data/jpa/reference/repositories/projections.html
https://graphql.org/learn/
https://docs.spring.io/spring-graphql/reference/index.html
https://spring.io/guides/gs/graphql-server/



























