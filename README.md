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

## HTTP server

Spring for graphql은 GraphQlHttpHandler을 제공함으로써 HTTP Reqeust를 핸들링할 수 있도록 기능들을 제공합니다.

Spring mvc와 Spring webflux 두가지를 모두 지원합니다. 요청은 비동기식으로 처리하며 응답을 처리할때는 각각 mvc, webflux에 맞는 방식으로 처리합니다.

GraphQL 요청을 위해서는 반드시 **Http Post** 요청이 사용되어야 하며 **"application/json"** 콘텐츠 타입을 사용해야 합니다. 

## FILE UPLOAD

Graphql은 텍스트 기반의 데이터를 주고받는데 중점을 두고 있으므로 media형태의 파일을 지원하지 않습니다.

다만 file upload를 위한 graphql-multipart-request-spec라는 별도의 스펙을 정의하고 있습니다. Spring for graphql이 이를 직접적으로 지원하지는 않으나 **multipart-spring-graphql** 라는 라이브러리를 사용하면 파일 업로드도 가능합니다.

## Interception

Spring graphql은 HTTP 요청이 Graphql Java engine에서 처리되기전, 후에 요청을 가로채서 부가적인 작업을 하는 것이 가능합니다.

```
class RequestHeaderInterceptor implements WebGraphQlInterceptor { 

	@Override
	public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) {
		String value = request.getHeaders().getFirst("myHeader");
		request.configureExecutionInput((executionInput, builder) ->
				builder.graphQLContext(Collections.singletonMap("myHeader", value)).build());
		return chain.next(request);
	}
}
```

```
class ResponseHeaderInterceptor implements WebGraphQlInterceptor {

	@Override
	public Mono<WebGraphQlResponse> intercept(WebGraphQlRequest request, Chain chain) { 
		return chain.next(request).doOnNext(response -> {
			String value = response.getExecutionInput().getGraphQLContext().get("cookieName");
			ResponseCookie cookie = ResponseCookie.from("cookieName", value).build();
			response.getResponseHeaders().add(HttpHeaders.SET_COOKIE, cookie.toString());
		});
	}
}

@Controller
class MyCookieController {

	@QueryMapping
	Person person(GraphQLContext context) { 
		context.put("cookieName", "123");
		...
	}
}
```

위의 예제들과 같이 요청 전후로 GraphQL Context의 값들을 조작할 수 있습니다.

## Schema resource

Boot starter는 classpath:graphql/**에 있는 .graphqls" or ".gqls" 파일들로 존재하는 schema 파일들을 자동으로 load해줍니다.

## RuntimeWiringConfigurer

RuntimeWiringConfigurer은 다음과 같은 네가지 역할을 합니다.

* Custom scalar types.
* Directives handling code.
* Default TypeResolver for interface and union types.

GraphQL java는 오직 Map으로된 데이터만 직렬화가 가능합니다. 그렇기 때문에 Client의 input은 모두 Map으로 변환됩니다. 서버의 output역시 선택된 필드들에 한해서 Map으로 변환됩니다.

GraphQL의 타입 시스템의 리프노드를 **scalar**라고 부릅니다. 즉 또 다른 depth가 존재하지 않는 field라는 뜻입니다.
우리는 여기서 다양한 custom scalar type을 정의할 수 있습니다. ex) Date

**directive**는 GraphQL document에서 type validation이나 런타임 실행을 대체할 수 있는 것들을 나타내는 GraphQL language입니다.
예를들면

```
type Employee
  id : ID
  name : String!
  startDate : String!
  salary : Float
}
```

다음과 같은 employee type에서 우리는 salary가 아무한테나 노출되는 것을 꺼려할 수 있습니다. 이러한 경우 directive를 사용한다면 보다 손쉽게 필드에 대한 런타임 제어를 가능케 해줍니다.

```
directive @auth(role : String!) on FIELD_DEFINITION

type Employee
  id : ID
  name : String!
  startDate : String!
  salary : Float @auth(role : "manager")
}
```

또 하나의 예를들면

```
directive @Size(min : Int = 0, max : Int = 2147483647, message : String = "graphql.validation.Size.message")
                        on ARGUMENT_DEFINITION | INPUT_FIELD_DEFINITION

    input Application {
        name : String @Size(min : 3, max : 100)
    }
```

다음과 같이 directive를 사용하여 손쉽게 필드의 사이즈를 validation할 수도 있습니다.

***** 

TypeResolver의 목적은 GraphQL Java가 GraphQL Interface나 Union field를 위해서 DataFetcher에 의해서 return되는 GraphQL Object의 Type을 결정할 수 있게 해줍니다. 
The purpose of a TypeResolver in GraphQL Java is to determine the GraphQL Object type for values returned from the DataFetcher for a GraphQL Interface or Union field.

ClassNameTypeResolver을 예로들어보자면 해당 TypeResolver는 class이름으로 GraphQL Object의 type을 결정해줍니다. 만약 못찾을시에 상위 클래스로 순회하면서 매칭되는 클래스 타입을 찾습니다.

## Operation Caching

GraphQL은 operation을 실행하기 위해서는 반드시 해당 operation을 parsing하고 validation해야합니다. 이것은 퍼포먼스에 심각한 영향을 미칠 수 있습니다. 이러한 것을 피하기 위해서는 **PreparsedDocumentProvider** 설정을 통해서 Document instance를 **캐싱**하여 재사용할 수 있습니다.

## Web MVC

Spring graphQL에서는 mvc servlet thread와 data fetcher의 스레드가 다를 수 있습니다. 문서에는 나와있지 않지만 비동기적으로 요청을 처리하는 방식때문에 발생하는 문제로 생각이 됩니다. Spring graphQL에서는 thread local을 propagation하는 기능을 지원합니다. 이것이 서로 다른 스레드임에도 불구하고 thread local정보를 전파해준다고 합니다.

다음과 같은 설정을 통해서 가능합니다.

```
public class RequestAttributesAccessor implements ThreadLocalAccessor<RequestAttributes> {

    @Override
    public Object key() {
        return RequestAttributesAccessor.class.getName();
    }

    @Override
    public RequestAttributes getValue() {
        return RequestContextHolder.getRequestAttributes();
    }

    @Override
    public void setValue(RequestAttributes attributes) {
        RequestContextHolder.setRequestAttributes(attributes);
    }

    @Override
    public void reset() {
        RequestContextHolder.resetRequestAttributes();
    }

}
```

## Exceptions

GraphQL Java에서는 **DataFetcherExceptionHandler**를 제공합니다. 이는 data fetching시에 에러가 발생하면 그것을 어떻게 표현할지 결정할 수 있게 해줍니다.

DataFetcherExceptionHandler는 **DataFetcherExceptionResolver**를 등록할 수 있게 지원하는데 spring for graphql에서는 여러개의 DataFetcherExceptionResolver를 등록할 수 있습니다.

spring graphql의 annotation controller를 사용해서 datafetcher를 구현한 경우에는 @GraphQlExceptionHandler 어노테이션을 사용해서 손쉽게 에러를 핸들링할 수 있습니다.

```
@Controller
public class BookController {

	@QueryMapping
	public Book bookById(@Argument Long id) {
		// ...
	}

	@GraphQlExceptionHandler
	public GraphQLError handle(BindException ex) {
		return GraphQLError.newError().errorType(ErrorType.BAD_REQUEST).message("...").build();
	}

}
```

## Pagination

spring graphql은 커서기반의 페이지네이션을 제공하고 있습니다. paging된 resultset을 **Connection Type**이라고 부릅니다.

Connection type은 edge라는 type이 존재하며 edge는 item과 cursor를 가지고 있습니다. 또한 connection type은 pageInfo라는 필드를 가지고 있습니다.

우리는 페이징이 필요한 type에 Connection Type을 정의할 수 있습니다.

```
Query {
	books(first:Int, after:String, last:Int, before:String): BookConnection
}

type Book {
	id: ID!
	title: String!
}
```

Book type 쿼리에 BookConnection타입을 리턴 타입으로 설정하면 Spring graphql은 내부적으로 다음과 같이 Connection type을 생성해줍니다.

```
type BookConnection {
	edges: [BookEdge]!
	pageInfo: PageInfo!
}

type BookEdge {
	node: Book!
	cursor: String!
}

type PageInfo {
	hasPreviousPage: Boolean!
	hasNextPage: Boolean!
	startCursor: String
	endCursor: String
}
```

정렬같은 경우 Spring graphql에서 제공해주는 표준적인 방법은 없으나 우리는 페이지네이션을 진행할때 sort order를 같이 넣어서 정렬된 형태의 페이지를 제공하는 것이 가능합니다. 이때는 요청으로 sort order를 따로 받을 수도 있고 default로 sort설정을 추가할 수도 있습니다.

```
    @QueryMapping
    fun posts(subrange: ScrollSubrange): Window<Post> {
        val scrollPosition = subrange.position().orElse(ScrollPosition.offset())
        val limit = Limit.of(subrange.count().orElse(10))
        val sort = Sort.by("title").ascending()
        return postService.findAllPosts(scrollPosition, limit, sort)
    }

```

## Batch Loading

우리가 Book type과 Author type을 가지고 있다고 가정할때 우리는 두개의 각기 다른 type에 대해서 DataFetcher를 생성해줄 수 있습니다. 그렇지만 이는 book과 author를 같이 load하지 않기 때문에 N+1문제가 발생할 수 있습니다. 

GraphQL Java는 이러한 연관 엔티티를 같이 조회하기 위해서 DataLoader를 제공합니다. spring graphql에서 batch loading을 하기 위해서는 BatchLoaderRegistry이 제공하는 팩터리 메서드를 사용하여 batch loading function들을 등록할 수 있습니다.

```
@Configuration
public class MyConfig {

	public MyConfig(BatchLoaderRegistry registry) {

		registry.forTypePair(Long.class, Author.class).registerMappedBatchLoader((authorIds, env) -> {
				// return Mono<Map<Long, Author>
		});

		// more registrations ...
	}

}
```

하지만 대부분의 케이스에서 우리는 @BatchMapping annotation을 사용해서 손쉽게 batch loading을 구현할 수 있습니다.

```
    @BatchMapping
    fun comments(posts: List<Post>): Map<Post, List<Comment>> {

        val postIds: MutableList<Long> = posts.stream().map { posts -> posts!!.id }.collect(Collectors.toList())
        val comments: List<Comment> = commentService.findCommentsByPostIds(postIds)
        val commentsByPost: Map<Post, List<Comment>> = comments.groupBy { it.post }

        return commentsByPost
    }
```

이렇게하면 일일이 DataLoader를 구현하여 registry에 등록하는 일을 하지 않아도 연관 entity를 함께 조회하는 batch loading을 구현할 수 있습니다.




























































# reference



https://docs.spring.io/spring-data/jpa/reference/repositories/projections.html
https://graphql.org/learn/
https://docs.spring.io/spring-graphql/reference/index.html
https://spring.io/guides/gs/graphql-server/



























