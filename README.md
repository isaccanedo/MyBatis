## MyBatis

# 1. Introdução
MyBatis é uma estrutura de persistência de código aberto que simplifica a implementação de acesso a banco de dados em aplicativos Java. Ele fornece suporte para SQL customizado, procedimentos armazenados e diferentes tipos de relações de mapeamento.

Simplificando, é uma alternativa para JDBC e Hibernate.

# 2. Dependências Maven
Para fazer uso do MyBatis, precisamos adicionar a dependência ao nosso pom.xml:

```
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.4.4</version>
</dependency>
```

# 3. APIs Java
### 3.1. SQLSessionFactory
SQLSessionFactory é a classe principal para todos os aplicativos MyBatis. Esta classe é instanciada usando o método builder() de SQLSessionFactoryBuilder, que carrega um arquivo XML de configuração:

```
String resource = "mybatis-config.xml";
InputStream inputStream Resources.getResourceAsStream(resource);
SQLSessionFactory sqlSessionFactory
  = new SqlSessionFactoryBuilder().build(inputStream);
```

O arquivo de configuração Java inclui configurações como definição de fonte de dados, detalhes do gerenciador de transações e uma lista de mapeadores que definem relações entre entidades, que juntos são usados para construir a instância SQLSessionFactory:

```
public static SqlSessionFactory buildqlSessionFactory() {
    DataSource dataSource 
      = new PooledDataSource(DRIVER, URL, USERNAME, PASSWORD);

    Environment environment 
      = new Environment("Development", new JdbcTransactionFactory(), dataSource);
        
    Configuration configuration = new Configuration(environment);
    configuration.addMapper(PersonMapper.class);
    // ...

    SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
    return builder.build(configuration);
}
```

### 3.2. SQLSession
SQLSession contém métodos para executar operações de banco de dados, obter mapeadores e gerenciar transações. Ele pode ser instanciado da classe SQLSessionFactory. As instâncias desta classe não são seguras para threads.

Depois de executar a operação do banco de dados, a sessão deve ser encerrada. Como SqlSession implementa a interface AutoCloseable, podemos usar o bloco try-with-resources:

```
try(SqlSession session = sqlSessionFactory.openSession()) {
    // do work
}
```

# 4. Mapeadores
Mapeadores são interfaces Java que mapeiam métodos para as instruções SQL correspondentes. MyBatis fornece anotações para definir as operações do banco de dados:

```
public interface PersonMapper {

    @Insert("Insert into person(name) values (#{name})")
    public Integer save(Person person);

    // ...

    @Select(
      "Select personId, name from Person where personId=#{personId}")
    @Results(value = {
      @Result(property = "personId", column = "personId"),
      @Result(property="name", column = "name"),
      @Result(property = "addresses", javaType = List.class,
        column = "personId", many=@Many(select = "getAddresses"))
    })
    public Person getPersonById(Integer personId);

    // ...
}
```

5. Anotações MyBatis
Vejamos algumas das principais anotações fornecidas por MyBatis:

- @Insert, @Select, @Update, @Delete - essas anotações representam instruções SQL a serem executadas chamando métodos anotados:

```
@Insert("Insert into person(name) values (#{name})")
public Integer save(Person person);

@Update("Update Person set name= #{name} where personId=#{personId}")
public void updatePerson(Person person);

@Delete("Delete from Person where personId=#{personId}")
public void deletePersonById(Integer personId);

@Select("SELECT person.personId, person.name FROM person 
  WHERE person.personId = #{personId}")
Person getPerson(Integer personId);
```

- @Results - é uma lista de mapeamentos de resultados que contém os detalhes de como as colunas do banco de dados são mapeadas para atributos de classe Java:

```
@Select("Select personId, name from Person where personId=#{personId}")
@Results(value = {
  @Result(property = "personId", column = "personId")
    // ...   
})
public Person getPersonById(Integer personId);
```

- @Result - representa uma única instância de Result da lista de resultados recuperados de @Results. Inclui detalhes como o mapeamento da coluna do banco de dados para a propriedade do Java bean, o tipo Java da propriedade e também a associação com outros objetos Java:

```
@Results(value = {
  @Result(property = "personId", column = "personId"),
  @Result(property="name", column = "name"),
  @Result(property = "addresses", javaType =List.class) 
    // ... 
})
public Person getPersonById(Integer personId);
```

- @Many - especifica um mapeamento de um objeto para uma coleção de outros objetos:

```
@Results(value ={
  @Result(property = "addresses", javaType = List.class, 
    column = "personId",
    many=@Many(select = "getAddresses"))
})
```

Aqui getAddresses é o método que retorna a coleção de Address consultando a tabela de endereços.

```
@Select("select addressId, streetAddress, personId from address 
  where personId=#{personId}")
public Address getAddresses(Integer personId);
```

Semelhante à anotação @Many, temos a anotação @One que especifica a relação de mapeamento um para um entre os objetos.

- @MapKey - é usado para converter a lista de registros em Mapa de registros com a chave definida pelo atributo de valor:

```
@Select("select * from Person")
@MapKey("personId")
Map<Integer, Person> getAllPerson();
```

- @Options - esta anotação especifica uma ampla gama de opções e configurações a serem definidas para que, em vez de defini-los em outras instruções, possamos definir @Options:

```
@Insert("Insert into address (streetAddress, personId) 
  values(#{streetAddress}, #{personId})")
@Options(useGeneratedKeys = false, flushCache=true)
public Integer saveAddress(Address address);
```

# 6. SQL dinâmico
SQL dinâmico é um recurso muito poderoso fornecido pelo MyBatis. Com isso, podemos estruturar nosso SQL complexo com precisão.

Com o código JDBC tradicional, temos que escrever instruções SQL, concatená-las com a precisão dos espaços entre elas e colocar as vírgulas nos lugares certos. Isso é muito sujeito a erros e muito difícil de depurar, no caso de grandes instruções SQL.

Vamos explorar como podemos usar SQL dinâmico em nosso aplicativo:

```
@SelectProvider(type=MyBatisUtil.class, method="getPersonByName")
public Person getPersonByName(String name);
```

Aqui, especificamos uma classe e um nome de método que realmente constrói e gera o SQL final:

```
public class MyBatisUtil {
 
    // ...
 
    public String getPersonByName(String name){
        return new SQL() {{
            SELECT("*");
            FROM("person");
            WHERE("name like #{name} || '%'");
        }}.toString();
    }
}
```

O SQL dinâmico fornece todas as construções de SQL como uma classe, por exemplo, SELECT, WHERE etc. Com isso, podemos alterar dinamicamente a geração da cláusula WHERE.

# 7. Suporte de procedimento armazenado
Também podemos executar o procedimento armazenado usando a anotação @Select. Aqui, precisamos passar o nome do procedimento armazenado, a lista de parâmetros e usar uma chamada explícita para esse procedimento:

```
@Select(value= "{CALL getPersonByProc(#{personId,
  mode=IN, jdbcType=INTEGER})}")
@Options(statementType = StatementType.CALLABLE)
public Person getPersonByProc(Integer personId);
```

# 8. Conclusão
Neste tutorial rápido, vimos os diferentes recursos fornecidos pelo MyBatis e como ele facilita o desenvolvimento de aplicativos voltados para banco de dados. Também vimos várias anotações fornecidas pela biblioteca.