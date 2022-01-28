두가지 방법이 있다. 
# Testcontainer 설정하기

테스트의 구성에 따라 디테일한 설정에 따라 다를 수 있겠다.

1. 모든 테스트에 대해 1개의 컨테이너로 테스트한다. 
2. 1개의 테스트당 1개의 컨테이너로 테스트한다. 

나는 1번과 같이 구성해서 사용했다. container를 static하게 구성하고 모든테스트에서 하나의 컨테이너를 사용하게 했다. 2번은 너무 느려서; 상황에 따라서는 확실한 분리가 되니 좋을 수도 있겠다. 

## 1. ApplicationContextInitializer 사용한다.

```java
@Slf4j
public class PostgreSQLContainerInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    private static PostgreSQLContainer sqlContainer = new PostgreSQLContainer("postgres:10.7");

    static {
        
        sqlContainer.start();
    }

    public void initialize (ConfigurableApplicationContext configurableApplicationContext){
        TestPropertyValues.of(
                "spring.datasource.url=" + sqlContainer.getJdbcUrl(),
                "spring.datasource.username=" + sqlContainer.getUsername(),
                "spring.datasource.password=" + sqlContainer.getPassword()
        ).applyTo(configurableApplicationContext.getEnvironment());
    }
}
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace= AutoConfigureTestDatabase.Replace.NONE)
@ContextConfiguration(initializers = {PostgreSQLContainerInitializer.class})
class UserRepositoryTest {

    @Autowired
    EntityManager entityManager;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldReturnUserGivenValidCredentials() {
        User user = new User(null, "test@gmail.com", "test", "Test");
        entityManager.persist(user);
        
        Optional<User> userOptional = userRepository.login("test@gmail.com", "test");
        
        assertThat(userOptional).isNotEmpty();
    }
}
```

## 2. @DynamicPropertySource, Java 8+ Interface 를 사용한다. 

```java
@Testcontainers
public interface PostgreSQLContainerInitializer {

    @Container
    PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:12.3");

    @DynamicPropertySource
    static void registerPgProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace= AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest implements PostgreSQLContainerInitializer {

    ....
    ....
}
```

## 참고
https://stackoverflow.com/questions/68602204/how-to-combine-testcontainers-with-datajpatest-avoiding-code-duplication



