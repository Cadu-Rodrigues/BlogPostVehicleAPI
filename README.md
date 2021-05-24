# Criando uma API REST com Spring Boot + Hibernate
Consumir dados de terceiros é uma parte crucial do desenvolvimento de aplicações web, para isso desenvolvedores disponibilizam rotinas em um formato padronizado que chamamos  de API's (Aplication Program Interface), que são interfaces responsáveis por disponibilizar dados de um software a um outro software, atualmente a arquitetura REST é utilizada para retornar nossos dados estruturados com mensagens significativas.

Juntando essa necessidade e uma das linguagens mais utilizadas no mundo nasce o Spring, um framework para desenvolvimento de aplicações java em várias plataformas diferentes, cuja especificação Spring Boot,voltada para web, é uma das tecnologias mais difundidas para esse intuito.

Neste post vamos construir uma API baseada em um projeto brinquedo, cujo objetivo é gerenciar veículos, para entendermos melhor como e porque utilizar essa stack de tecnologias.

O código completo desta implementação se encontra neste [link](https://github.com/Cadu-Rodrigues/vehicleapi)
Tecnologias Utilizadas:

* Maven: Nosso gerenciador de pacotes, seu papel é fazer toda a instalação de bibliotecas/dependências que não venham por padrão no java.
* [Spring Boot](https://spring.io/projects/spring-boot): Um dos projetos do ecossistema Spring, o Spring Boot é um pacote que insere funcionalidades deste ecossistema, um servidor que hospeda nossa aplicação para testes locais e facilita o trabalho de configuração do nosso projeto java.
* Hibernate: Nossa biblioteca de mapeamento Objeto-Relacional, o Hibernate é utilizado para criar mapeamentos entre o paradigma Orientado a Objetos do java e o modelo relacional do SQL.
* Spring JPA: Uma implementação da JPA (especificação que dita como os ORM's devem ser utilizados) que utiliza o Hibernate e cria mapeamentos de nossas classes para tabelas no nosso banco de dados, utilizaremos para reduzir a quantidade de código necessária para consumir nosso banco de dados.
* H2: Um banco de dados que utiliza a linguagem SQL e que nos possibilita subir um banco de dados na memória do servidor (ou do nosso computador, caso o projeto esteja rodando localmente), para que seja nossa fonte de dados e seja simples de implementar.
* Uma IDE de seu feitio (Eclipse, Intellij e para os mais ousados até mesmo um vscode :)).
## Conhecendo nosso escopo 

Claro que antes de começar nosso desenvolvimento precisamos saber o que vamos desenvolver, para este artigo temos uma especificação de requisitos que iremos seguir para aprendizado.
Vamos desenvolver uma API REST que precisará controlar veículos de usuários.

O primeiro passo deve ser a construção de um cadastro de usuários, sendo obrigatórios: nome, e-mail, CPF e data de nascimento, sendo que e-mail e CPF devem ser únicos.

O segundo passo é criar um cadastro de veículos, sendo obrigatórios: Marca, Modelo do Veículo e Ano. E o serviço deve consumir a API da FIPE (https://deividfortuna.github.io/fipe/) para obter os dados do valor do veículo baseado nas informações inseridas.

O terceiro passo é criar um endpoint que retornará um usuário com a lista de todos seus veículos cadastrados.

Devemos construir 3 endpoints neste sistema, o cadastro do usuário, o cadastro de veículo e a listagem dos veículos para um usuário específico.

No endpoint que listará seus veículos, devemos considerar algumas configurações a serem exibidas para o usuário final. Vamos criar dois novos atributos no objeto do carro, sendo eles:

1.) Dia do rodízio deste carro, baseado no último número do ano do veículo, considerando as condicionais:
Final 0-1: segunda-feira
Final 2-3: terça-feira
Final 4-5: quarta-feira
Final 6-7: quinta-feira
Final 8-9: sexta-feira

2.) Também devemos criar um atributo de rodízio ativo, que compara a data atual do sistema com as condicionais anteriores e, quando for o dia ativo do rodizio, retorna true; caso contrario, false.

Exemplo A: hoje é segunda-feira, o carro é da marca Fiat, modelo Uno do ano de 2001, ou seja, seu rodízio será às segundas-feiras e o atributo de rodízio ativo será TRUE.
Exemplo B: hoje é quinta-feira, o carro é da marca Hyundai, modelo HB20 do ano de 2021, ou seja, seu rodizio será às segundas-feiras e o atributo de rodízio ativo será FALSE.

- Caso os cadastros estejam corretos, é necessário voltar o Status 201. Caso hajam erros de preenchimento de dados, o Status deve ser 400.
- Caso a busca esteja correta, é necessário voltar o status 200. Caso haja erro na busca, retornar o status adequado e uma mensagem de erro amigável.

## Criando os modelos de dados
Vamos começar criando nossas classes que vão mapear os dados para o banco de dados. Aqui entra em cena o Spring JPA, com ele é possível criar classes que representam entidades e associá-las as tabelas que criaremos em nosso banco de dados, nas entidades mapearemos chaves primárias, entidades que estão contidas nesta entidade, a tabela associada àquela entidade no banco entre outras sobrescrições possíveis, sendo feito por meio de anotações marcadas com @.
* Classe Users
``` Java
package com.cadu.vehicleapi.model;

@Entity
@Table(name = "Users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long id;
    public String name;
    public String email;
    public int CPF;
    @Column(name = "Birthdate")
    public LocalDate birthDate;
    @OneToMany(mappedBy = "owner")
    public List<Vehicle> vehicles;

    public User() {

    }

    public User(String name, String email, int CPF, LocalDate birthDate) {
        this.name = name;
        this.email = email;
        this.CPF = CPF;
        this.birthDate = birthDate;
    }
}
```
* Classe Vehicle
``` Java
package com.cadu.vehicleapi.model;
@Entity
@Table(name = "Vehicles")
public class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public long id;
    public String brand;
    public String model;
    public String year;
    public String value;
    @ManyToOne
    public User owner;

    public Vehicle() {
    }

    public Vehicle(String brand, String model, String year, String value, User owner) {
        this.brand = brand;
        this.model = model;
        this.year = year;
        this.value = value;
        this.owner = owner;
    }

}
```
Cada classe é mapeada como entidade pela anotação *@Entity* e está associada a uma tabela do banco definida pela anotação *@Table* que define o nome da tabela a ser associada, além disso todas as nossas entidades precisam conter uma [chave primária](https://pt.wikipedia.org/wiki/Chave_prim%C3%A1ria) cujo mapeamento é dado por *@Id*, no caso das chaves primárias também é preciso definir a estratégia de geração do valor nas tabelas com o *@GeneratedValue*, para nosso contexto será utilizada a estratégia [identity](https://www.devmedia.com.br/jpa-como-usar-a-anotacao-generatedvalue/38592).

Utilizamos também a anotação *@Column*, por padrão o Spring JPA mapeia nossos atributos com seu nome para o banco de dados, porém caso os valores sejam diferentes entre nossa entidade e nosso banco (por usar estratégias diferentes de nomeação como [camelCase](https://pt.wikipedia.org/wiki/CamelCase) e [snake_case](https://en.wikipedia.org/wiki/Snake_case) por exemplo) podemos utilizar essa anotação para criar nosso mapeamento.

Também precisamos mapear nossos relacionamentos, um veículo possui um dono e um usuário possui vários veículos, portanto temos as relações:
* 1 User - possui - N Veículos
* 1 Veículo - possui - 1 dono
  
Para isso declaramos uma lista de entidades vehicle em nossa entidade user e uma entidade user dentro de nossa entidade vehicle, para mapear esse relacionamento precisamos dizer nas entidades o tipo de relação que está ali contida:
* Na classe vehicle temos Many vehicles To one owner por isso é utilizada a notação *@ManyToOne*
* Na classe user temos One user To many vehicles por isso é utilizada a notação *@OneToMany*
* Por fim para marcar que essas duas anotações tratam de um único relacionamento entre vehicle e user utilizados o parâmetro mapped by e o campo correlato na outra entidade como parâmetro.

Para finalizar nossas entidades declaramos nossos métodos construtores, sempre lembrando que o Spring JPA necessita de um construtor vazio declarado para realizar de forma correta seu mapeamento.
## Criando o banco de dados em memória
Bom, nosso modelo agora mapeia nossas classes para as tabelas do banco de dados... mas que banco de dados? Afinal, ainda não criamos nada que seja utilizado por esse mapeamento como banco de dados. 

Ai que entra o H2! Escolhemos o H2, por ser um banco de dados mais simples de subir, pois demanda pouca configuração e é descartável após o servidor ser desligado. Assim podemos nos concentrar na API, que é o foco do nosso estudo.

Considerando esses pontos utilizaremos o H2 como um banco de dados em memória, isso significa que quando o nosso servidor for iniciado criaremos um banco de dados ali na memória para ser utilizado temporariamente e que quando nosso servidor for desligado ou reiniciado ele deixará de existir.

E como fazer isso? o Spring por padrão escaneia o diretório resources,
caso exista o arquivo de configuração application.properties, ele o utiliza para definir configurações do banco de dados e os drivers de acesso para uso do Hibernate.

Veja a seguir nossa configuração do arquivo application.properties:
``` Properties
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.url=jdbc:h2:mem:vehicledb
spring.datasource.username=sa
spring.datasource.password=

# jpa
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=update

# h2
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```
Neste arquivo definimos quais drivers de banco de dados utilizar, a url em que o banco será criado (neste caso utilizamos esse campo para definir a criação do banco em memória no servidor ao invés de um diretório fixo de arquivos ou um servidor externo), o nome de usuário e senha de acesso ao banco e de quebra declarar uma url dentro de nossa api para que tenhamos acesso a uma interface gráfico de uso do banco.

### Inserindo dados e tabelas no banco
Tudo bem bonito mas como inserimos nossos dados e tabelas? Mais uma vez o Spring salva nossa vida, por padrão ele executa nesse mesmo diretório resources o arquivo como nome data.sql, basta criar este arquivo com os códigos SQL que queremos executar em nosso banco e a magia ocorre.
```SQL
DROP TABLE IF EXISTS Users CASCADE;

DROP TABLE IF EXISTS Vehicles CASCADE;

CREATE TABLE Users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    Name VARCHAR(250),
    Email VARCHAR(250),
    CPF INT(11),
    Birthdate Date
);

CREATE TABLE Vehicles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    Brand VARCHAR(250),
    Model VARCHAR(250),
    Year VARCHAR(250),
    Value VARCHAR(250),
    Owner_id INT
);

INSERT INTO
    Users (Name, Email, CPF, Birthdate)
VALUES
    (
        'Carlos',
        'CarlosEmail@email.com',
        1234,
        '1988-05-05'
    );

INSERT INTO
    Users (Name, Email, CPF, Birthdate)
VALUES
    (
        'Amadeu',
        'AmadeuEmail@email.com',
        4321,
        '1989-05-05'
    );

INSERT INTO
    Users (Name, Email, CPF, Birthdate)
VALUES
    (
        'Robson',
        'RobsonEmail@email.com',
        3333,
        '1964-05-05'
    );

INSERT INTO
    Vehicles (Brand, Model, Year, Value, Owner_id)
VALUES
    (
        'Ford',
        'Fiesta 1.6 16V Flex Aut. 5p',
        '2016 Gasolina',
        '10000',
        1
    );

INSERT INTO
    Vehicles (Brand, Model, Year, Value, Owner_id)
VALUES
    (
        'Ford',
        'Fusion 2.5L I-VCT Flex Aut.',
        '2016 Gasolina',
        '20000',
        2
    );

INSERT INTO
    Vehicles (Brand, Model, Year, Value, Owner_id)
VALUES
    (
        'Ford',
        'Ka 1.0 8V/1.0 8V ST Flex 3p',
        '2013 Gasolina',
        '30000',
        1
    );
```
Finalizado isto teremos uma estrutura de diretórios parecida com isto:

![Estrutura de diretórios após criação dos arquivos de banco](/images/estrutura_de_diretorios.png "Estrutura de diretórios após criação dos arquivos de banco")
*<center>Figura 1. Estrutura de diretórios após criação dos arquivos de banco</center>*

Assim, quando subirmos nosso servidor, o banco de dados terá dados iniciais e poderá ser acessado pelo link http://localhost:8080/h2-console.

![Tela de login do banco de dados acessado pelo navegador](/images/login_h2_console.png "Tela de login do banco de dados acessado pelo navegador")
*<center>Figura 2. Tela de login do banco de dados acessado pelo navegador</center>*
![Interface do banco de dados em utilização](/images/tela_h2_em_funcionamento.png "Interface do banco de dados em utilização")
*<center>Figura 3. Interface do banco de dados em utilização</center>*
## Acessando o banco de dados 
Para acessar nossos dados, precisamos ler essas entidades e enviar os resultados dela para nosso usuário por meio dos endpoints de nossa API.

Para isso, vamos utilizar o padrão repository, em que isolamos o código de acesso ao banco de dados em uma classe específica (uma para cada funcionalidade), a fim de desacoplar código e aproveitar das
interfaces de auxílio da JPA.

Nesse momento criaremos dois respositorys (um para cada entidade) e é onde a implementaçãp da JPA nos ajuda. Nossos repositorys serão apenas interfaces que implementam a JpaRepository, isso significa que os métodos dessa classe serão todos implementados pelo próprio Spring, estes métodos fazem operações básicas e extremamente corriqueiras como criar, retornar, modificar, e deletar um objeto do banco de dados o que abstrai lógica repetitiva e gera menos código escrito.

Para implementar uma JpaRepository é necessário passar como parâmetros a entidade a ser manipulada e o tipo da chave primária dela em nosso mapeamento.

Por ser uma interface que implementa uma classe do Spring JPA dentro de nosso repository só podemos criar assinaturas de funções, e aqui temos uma funcionalidade muito interessante do framework, caso declaremos uma assinatura cujo nome seja um dos métodos implementados (findBy por exemplo) seguido por um atributo da entidade (no caso cpf), a biblioteca consegue gerar esse método baseado na implementação ja existente (no caso findById) e adaptar a execução para utilizar o atributo requerido. 

* Classe repository de usuários:
``` Java
public interface UsersRepository extends JpaRepository<User, Long> {

    User findByCPF(int cpf);

}
```
* Classe repository de veículos:
```Java
public interface VehiclesRepository extends JpaRepository<Vehicle, Long> {

    List<Vehicle> findByOwnerCPF(int cpf);

}
```
É importante destacar que, caso o atributo seja de uma entidade dentro da entidade do repository (o cpf dentro do user que está mapeado em vehicle por exemplo), a entidade precisa ser passada na assinatura do método antes do atributo.

Também é possível que esse relacionamento gere uma ambiguidade caso a assinatura do método possa ser a mesma para um atributo de uma entidade relacionada e um atributo da entidade inicial (por exemplo caso houvesse um atributo OwnerCPF em vehicle), nesse caso a diferenciação pode ser demarcada no método que utiliza a entidade + atributo em sua assinatura separando por _.

Pronto, temos os nossos repositorys para acesso ao banco e agora precisamos consumi-lo para retornar seus dados.

## Exibindo dados ao cliente
Finalmente a hora de ver coisas funcionando (já estava na hora né?). Agora precisamos acessar nosso repository, os métodos que essas interfaces implementam e exibi-los para nosso usuário, ou seja precisamos criar nossos endpoints.

Em nossos endpoints a ideia é acessar o repository relativo àquele endpoint, executar seus métodos e retornar os valores em um formato amigável ao cliente.

Considerando que vamos retornar dados é possível que nem todos os atributos de nossa entidade sejam pertinentes a essa resposta, imagine retornar um id do banco de dados ou um campo senha em nossa API como não seria problemático.
Pensando nisso criaremos classes DTO(Data Transfer Objects).
### Classes DTO
A ideia aqui é criar uma visualização especifíca da nossa entidade e retornar um objeto deste tipo para quem acessa nosso endpoint, isso desacopla o retorno de dados das nossas entidades da JPA e possibilita que tenhamos diferentes objetos de retorno para a mesma entidade (por exemplo caso um determinado endpoint deva retornar um veículo sem seu valor no futuro).

Implementação das classes DTO:
* Classe UserDTO
```Java
public class UserDTO {
    public String name;
    public String email;
    public int CPF;
    public LocalDate birthDate;

    public UserDTO(User user) {
        this.name = user.name;
        this.email = user.email;
        this.CPF = user.CPF;
        this.birthDate = user.birthDate;
    }

    public UserDTO() {

    }

}
```
* Classe VehicleDTO
```Java
public class VehicleDTO {
    public String brand;
    public String model;
    public String year;
    public String value;
    public int ownerCPF;
    public String rodizioDay;
    public Boolean rodizioIsAtive;
    private VehicleService vehicleService = new VehicleService();

    public VehicleDTO(Vehicle vehicle) throws Exception {
        this.brand = vehicle.brand;
        this.model = vehicle.model;
        this.year = vehicle.year;
        this.value = vehicle.value;
        this.ownerCPF = vehicle.owner.CPF;
        this.rodizioDay = vehicleService.getRodizioDay(vehicle);
        this.rodizioIsAtive = vehicleService.getRodizioAtive(this, LocalDate.now());
    }

    public VehicleDTO() {

    }
}
```
```Java
public class VehiclesFromUserDTO {
    public String ownerName;
    public List<VehicleDTO> vehicles;

    public VehiclesFromUserDTO(List<Vehicle> vehicles, User user) throws Exception {
        this.vehicles = new ArrayList<VehicleDTO>();
        VehicleMapper mapper = new VehicleMapper();
        this.vehicles = mapper.convert(vehicles);
        this.ownerName = user.name;
    }
}
```
Cada DTO possui um construtor que recebe seu correspondente em entidade e armazena os dados necessários em si mesma. No caso do DTO de veículos, os campos *rodizioDay* e *rodizioIsAtive* recebem valores calculados em uma implementação que veremos mais à
frente. Além disso, temos um novo tipo de classe utilizada aqui, os mappers.

### Classes Mapper
Em nossa API convertemos entidades em DTO para retorná-las nos endpoints, quando é apenas um objeto para ser convertido nosso construtor resolve o problema, mas e quando temos objetos mais complexos necessitando dessa conversão como por exemplo uma lista de entidades?

A partir desse problema criamos mappers que são classes encarregadas de implementar essas conversões e isolar esse código de nosso DTO.
* Mapper de usuários
```Java
public class UserMapper {
    public List<UserDTO> convert(List<User> vehicles) {
        List<UserDTO> array = new ArrayList<UserDTO>();
        vehicles.forEach(it -> {
            UserDTO iterate = new UserDTO();
            iterate.name = it.name;
            iterate.email = it.email;
            iterate.CPF = it.CPF;
            iterate.birthDate = it.birthDate;
            array.add(iterate);
        });
        return array;
    }
}
```
* Mapper de veículos
```Java
public class VehicleMapper {
    private VehicleService vehicleService = new VehicleService();

    public List<VehicleDTO> convert(List<Vehicle> vehicles) {
        List<VehicleDTO> array = new ArrayList<VehicleDTO>();
        vehicles.forEach(it -> {
            VehicleDTO iterate = new VehicleDTO();
            iterate.brand = it.brand;
            iterate.model = it.model;
            iterate.year = it.year;
            iterate.value = it.value;
            iterate.ownerCPF = it.owner.CPF;
            iterate.rodizioDay = vehicleService.getRodizioDay(it);
            iterate.rodizioIsAtive = vehicleService.getRodizioAtive(iterate, LocalDate.now());
            array.add(iterate);
        });
        return array;
    }

}
```
Novamente precisaremos de nosso serviço inserindo dados específicos, mas a boa notícia é que estamos prontos para criar os endpoints da API.
## Criando os endpoints de retorno 
Agora, finalmente, criaremos nossos controllers, que são as classes que o Spring irá mapear (e que são nossos endpoints, afinal) de acordo com a URL desejada. Neste primeiro momento, criaremos os 3 endpoints de retorno, que enviam veículos, usuários e todos os veículos de um determinado usuário dado seu CPF para o cliente.
* Controller de Usuários
```Java
@RestController
public class UsersController {
    @Autowired
    private UsersRepository repository;

    @GetMapping("/users")
    public List<UserDTO> listUsers() {
        List<User> users = repository.findAll();
        UserMapper mapper = new UserMapper();
        return mapper.convert(users);
    }
}
```
Neste trecho, inserimos a anotação que diz ao Spring que a classe é um controller da API, para que o spring se encarregue do mapeamento de nossos endpoints. Também injetamos nosso repository, afim de que quando nosso controller seja criado, os métodos do nosso repository estejam disponíveis.

Após isso inserimos nossa anotação que especifica que quando o endpont /users for chamado pelo verbo GET aquele método será executado. 
A partir disso:
* Criamos uma lista de usuários e chamamos o método implementado pela JPA findAll() para retornar todos os usuários de nosso banco de dados.
* Instanciamos nosso mapper e retornamos o valor dessa conversão para a nossa rota.

Executando uma requisição para nossa rota temos esse resultado:
![Resultado de requisição para endpoint de usuários](/images/requisicao_users.png "Resultado de requisição para endpoint de usuários")
*<center>Figura 4. Resultado de requisição para endpoint de usuários</center>*
O [Postman](https://www.postman.com/product/api-client/) é um aplicativo para fazer solicitações HTTP muito útil para testar requisições e algumas funcionalidades dele  serão utilizadas mais a fundo nos próximos passos.

Agora criaremos nossos controllers dos dois próximos endpoints.
* Controller de veículos
```Java
@RestController
public class VehiclesController {

    @Autowired
    private VehiclesRepository repository;

    @GetMapping("/vehicles")
    public List<VehicleDTO> listVehicles() throws Exception {
        VehicleMapper mapper = new VehicleMapper();
        List<Vehicle> vehicles = repository.findAll();
        return mapper.convert(vehicles);
    }
}
```
* Controller de veículos de um usuário
```Java
@RestController
public class VehiclesFromUserController {
    @Autowired
    private UsersRepository usersRepository;
    @Autowired
    private VehiclesRepository vehiclesRepository;

    @GetMapping("/vehiclesUser/{cpf}")
    public VehiclesFromUserDTO listVehiclesUser(@PathVariable int cpf) throws Exception {
        UserDTO dto = new UserDTO();
        User user = usersRepository.findByCPF(cpf);
        List<Vehicle> vehicles = vehiclesRepository.findByOwnerCPF(cpf);
        VehiclesFromUserDTO result = new VehiclesFromUserDTO(vehicles, user);
        return result;
    }
}
```
Aqui no VehiclesFromUserController, estamos fazendo uso dos métodos que criamos pela interface de repository para acharmos um usuário pelo seu CPF e depois todos os veiculos cujo CPF de seu dono são iguais ao parâmetro passado.

Testando o retorno temos. 
![Resultado de requisição para endpoint de veículos](/images/requisicao_vehicles.png "Resultado de requisição para endpoint de veículos")
*<center>Figura 5. Resultado de requisição para endpoint de veículos</center>*
![Resultado de requisição para endpoint de veículos de um usuário](/images/requisicao_vehiclesfromuser.png "Resultado de requisição para endpoint de veículos de um usuário")
*<center>Figura 6. Resultado de requisição para endpoint de veículos de um usuário</center>*

Nesse retorno temos um pequeno spoiler sobre nosso serviço mas a gente chega lá, agora o próximo problema é como criar novos veículos e usuários.
## Inserindo dados pelo controller
Nesse instante o próximo passo é cadastrar dados pelo endpoint, dessa forma será necessário o uso do verbo POST, neste ponto também é necessário formatar os dados recebidos pela requisição em classes, mas como visto anteriormente nossos DTO's são visualizações específicas das entidades, portanto os dados inseridos são modelos diferentes dos nossos DTO's.
## Classes Form
Para separar os arquivos que tratam de retorno de dados ao nosso usuário e os arquivos que se referem aos dados que o cliente nos envia para salvar, criaremos classes novas chamadas de form's.
* Classe form de usuário
```Java
public class UserForm {
    @NotNull
    @NotEmpty
    public String name;
    @NotNull
    @NotEmpty
    @Email
    public String email;
    @NotNull
    private int CPF;
    @NotNull
    public LocalDate birthDate;

    public User convert(UserForm form) {
        return new User(form.name, form.email, form.CPF, form.birthDate);
    }
}
```
* Classe form de veículo
```Java
public class VehicleForm {
    @NotNull
    @NotEmpty
    public String brand;
    @NotNull
    @NotEmpty
    public String model;
    @NotNull
    @NotEmpty
    public String year;
    @NotNull
    public int ownerCPF;

    public Vehicle convert(VehicleForm form, UsersRepository repository) {
        User user = repository.findByCPF(form.ownerCPF);
        return new Vehicle(form.brand, form.model, form.year, "0", user);
    }
}
```
## Bean Validation
Nossa implementação de cadastro de dados está funcionando, porém o que acontece quando o cliente que consome nossa API não manda todos os campos necessários? insere valores nulos nos campos? Manda o hino do Flamengo num campo inteiro?.

Como não podemos confiar que nosso usuário sempre inserirá os dados corretamente é preciso validar os valores passados para inserção em nossa API.

Uma primeira abordagem que possa vir a mente é criar estruturas condicionais que tratem cada possibilidade de um parametro ser inválido porém isso geraria toneladas de código e está sujeito a não lembrarmos de um caso específico que pode quebrar a aplicação.

Por isso a abordagem de Bean validation é interessante, nela simplesmente declaramos por meio de anotações, restrições a um dado (não deve ser nulo, não pode ser vazio, tamanho mínimo ou máximo) e ao utilizar aquelas classes que modelam a inserção passamos uma anotação que diga ao Spring para verificar as restrições mapeadas e nos expelir um erro caso algo não ocorra bem.

## Escrevendo os métodos de inserção no endpoint
Agora em nossos endpoints vamos escrever o mapeamento das requisições que inserem novos dados no banco.
* Inserindo método de criação de usuário
```Java
@RestController
public class UsersController {
    @Autowired
    private UsersRepository repository;

    @GetMapping("/users")
    public List<UserDTO> listUsers() {
        List<User> users = repository.findAll();
        UserMapper mapper = new UserMapper();
        return mapper.convert(users);
    }

    @PostMapping("/users")
    public ResponseEntity<UserDTO> createUser(@RequestBody @Valid UserForm form, UriComponentsBuilder uriBuilder) {
        User user = form.convert(form);
        this.repository.save(user);
        URI uri = uriBuilder.path("/users/{id}").buildAndExpand(user.id).toUri();
        return ResponseEntity.created(uri).body(new UserDTO(user));
    }

}
```
Em nosso controller de usuários criamos nossa variável do tipo de nossa entidade e recebemos a conversão do valor mapeado pelo form, salvamos no banco de dados com o repository e retornamos o link de acesso daquele objeto como cabeçalho da resposta além de um DTO do usuário como corpo.

Vale notar o uso da anotação @Valid para inserir o método no Bean validation descrito anteriormente.

* Inserindo método de criação de veículo
```Java
@RestController
public class VehiclesController {

    @Autowired
    private VehiclesRepository repository;
    @Autowired
    private UsersRepository userRepository;

    @GetMapping("/vehicles")
    public List<VehicleDTO> listVehicles() throws Exception {
        VehicleMapper mapper = new VehicleMapper();
        List<Vehicle> vehicles = repository.findAll();
        return mapper.convert(vehicles);
    }

    @PostMapping("/vehicles")
    public ResponseEntity<VehicleDTO> createVehicle(@RequestBody @Valid VehicleForm form,
            UriComponentsBuilder uriBuilder) throws Exception {
        Vehicle vehicle = form.convert(form, userRepository);
        VehicleService service = new VehicleService();
        vehicle.value = service.getVehicleValue(vehicle);
        this.repository.save(vehicle);
        URI uri = uriBuilder.path("/vehicles/{id}").buildAndExpand(vehicle.id).toUri();
        return ResponseEntity.created(uri).body(new VehicleDTO(vehicle));

    }
}
```
Já em nossa implementação de veículos poucas mudanças exceto o uso do serviço externo VehicleService novamente para retornar o valor do veículo.

Ao fazer nossas requisições temos o seguinte resultado:
![Resultado de requisição post para endpoint de usuário](/images/requisicao_userspost.png "Resultado de requisição post para endpoint de usuário")
*<center>Figura 7. Resultado de requisição post para endpoint de usuário</center>*
![Resultado de requisição post para endpoint de veículo](/images/requisicao_vehiclespost.png "Resultado de requisição post para endpoint de veículo")
*<center>Figura 8. Resultado de requisição post para endpoint de veículo</center>*

## Fazendo a magia acontecer no service
Finalmente chegamos à nossa camada de regra de negócio, aqui nós criamos as funções responsáveis por retornar o valor de um veículo de uma API externa e descobrir o dia de rodízio de um carro e se hoje é o dia de rodízio dele.
### Testes unitários
Inicialmente aqui nós temos uma especificação muito clara do que nossas funções devem fazer, na camada de negócio (que é onde implementamos nossas restrições e lógica que manipula e gera novos dados) é importante garantir que o que nossas funções retornam é o esperado delas.

Por isso é um ótimo momento para utilizar o JUnit para criar testes que validem nossas funções e vejam se elas retornam o que deveriam retornar.

Para isso criaremos nosso arquivo de testes no repositório criado por padrão pelo spring e criaremos os testes que validam o retorno do dia do rodízio de um veículo.

* Classe de testes
```Java
package com.cadu.vehicleapi;
@SpringBootTest
class VehicleServiceTests {
	private VehicleService service = new VehicleService();

	@Test
	void shouldReturnTrueWhenIsRodizioDay() throws Exception {
		User user = new User("Carlos", "carlosEmail@email.com", 1234, LocalDate.parse("2000-04-13"));
		Vehicle vehicle = new Vehicle("ford", "fiesta", "2006", "9000", user);
		VehicleDTO vehicleDTO = new VehicleDTO(vehicle);
		assertTrue(service.getRodizioAtive(vehicleDTO, LocalDate.parse("2021-05-20")));
	}

	@Test
	void shouldReturnFalseWhenIsNotRodizioDay() throws Exception {
		User user = new User("Carlos", "carlosEmail@email.com", 1234, LocalDate.parse("2000-04-13"));
		Vehicle vehicle = new Vehicle("ford", "fiesta", "2006", "9000", user);
		VehicleDTO vehicleDTO = new VehicleDTO(vehicle);
		assertFalse(service.getRodizioAtive(vehicleDTO, LocalDate.parse("2021-05-21")));
	}

}
```
Marcamos nossa classe como uma classe de teste do spring boot e cada função como um teste, criamos um teste para dizer se um carro cujo dia do rodízio é hoje está retornando true e um outro para dizer se um carro cujo dia do rodízio não é hoje está retornando false.

Sempre necessário frisar que testar os casos falsos e Corner Cases (casos onde sua função não lida muito bem ou não são comuns) é importante para a correta validação do código.
### Criando o service
Temos nossos testes que validam o service, falta apenas o service :), brincadeiras a parte essa forma de abordagem evita que pensemos os testes para que o código seja aceito, ao invés de pensarmos o código para que passe nos testes, o que significa que teremos testes muito mais assertivos em sua proposta.

A primeira parte a se codar são as funções que consomem a API externa.
```Java
@Service
public class VehicleService {
    @Autowired
    public RestTemplate restTemplate;

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public VehicleService() {
        this.restTemplate = new RestTemplate();
    }

    private final String baseURL = "https://parallelum.com.br/fipe/api/v1/carros/marcas";

    public String getVehicleValue(Vehicle vehicle) throws Exception {
        String brandCode = "";
        String modelCode = "";
        String year = "";
        try {
            brandCode = getBrandCode(vehicle.brand);
            modelCode = getModelCode(brandCode, vehicle.model);
            year = getYear(brandCode, modelCode, vehicle.year);
        } catch (IllegalArgumentException exception) {
            throw new Exception(exception);
        }
        String url = baseURL + "/" + brandCode + "/modelos/" + modelCode + "/anos/" + year;
        FipeReturnValue result = restTemplate.getForObject(url, FipeReturnValue.class);
        return result.valor;
    }

    public String getBrandCode(String brandName) throws IllegalArgumentException {
        String result = "";
        if (brandName == "")
            throw new IllegalArgumentException("Brand name is invalid");
        ResponseEntity<Brand[]> response = restTemplate.getForEntity(baseURL, Brand[].class);
        Brand brands[] = response.getBody();
        for (int i = 0; i < brands.length; i++) {
            if (brands[i].nome.equals(brandName))
                result = brands[i].codigo;
        }
        if (result == "")
            throw new IllegalArgumentException("Brand name is invalid");
        return result;
    }

    public String getModelCode(String brandCode, String modelName) throws IllegalArgumentException {
        String result = "";
        String URL = baseURL + "/" + brandCode + "/modelos";
        if (brandCode == "")
            throw new IllegalArgumentException("Brand name is invalid");
        if (modelName == "")
            throw new IllegalArgumentException("Model name is invalid");
        ModelAndYear response = restTemplate.getForObject(URL, ModelAndYear.class);
        Model models[] = response.modelos;
        for (int i = 0; i < models.length; i++) {
            if (models[i].nome.equals(modelName))
                result = String.valueOf(models[i].codigo);
        }
        if (result == "")
            throw new IllegalArgumentException("Model name is invalid");
        return result;
    }

    public String getYear(String brandCode, String modelCode, String carYear) throws IllegalArgumentException {
        String result = "";
        String URL = baseURL + "/" + brandCode + "/modelos/" + modelCode + "/anos";
        if (brandCode == "")
            throw new IllegalArgumentException("Brand name invalid");
        if (modelCode == "")
            throw new IllegalArgumentException("Model name is invalid");
        if (carYear == "")
            throw new IllegalArgumentException("Vehicle year is invalid");
        ResponseEntity<Year[]> response = restTemplate.getForEntity(URL, Year[].class);
        Year years[] = response.getBody();
        for (int i = 0; i < years.length; i++) {
            if (years[i].nome.equals(carYear))
                result = years[i].codigo;
        }
        if (result == "")
            throw new IllegalArgumentException("Vehicle year is invalid");
        return result;
    }
```
Aqui temos 3 funções que servem para, dado seus atributos, retornar da API um valor que posteriormente será utilizado para uma requisição, por exemplo retornar o código interno da API que identifica a marca VolksWagen, depois iremos utilizar este código para descobrir o código que identifica o modelo AMAROK, em seguida descobrir seu ano e por fim seu valor na tabela FIPE.

Importante frisar a necessidade de ter exceptions que façam com que nosso código nos diga se tem algo errado principalmente com os parâmetros já que um parâmetro errado geraria um efeito em cascata e atrapalharia todo o funcionamento de nossa API.

A seguir vamos nos focar em retornar o dia do rodízio e se o dia passado para o serviço é o dia do rodízio do carro.
```Java
    public String getRodizioDay(Vehicle vehicle) throws IllegalArgumentException {
        String words[] = vehicle.year.split(" ");
        String year = words[0];
        if (year.substring(year.length() - 1).equals("0") || year.substring(year.length() - 1).equals("1"))
            return DayOfWeek.MONDAY.getDisplayName(TextStyle.FULL, LocaleContextHolder.getLocale());
        if (year.substring(year.length() - 1).equals("2") || year.substring(year.length() - 1).equals("3"))
            return DayOfWeek.TUESDAY.getDisplayName(TextStyle.FULL, LocaleContextHolder.getLocale());
        if (year.substring(year.length() - 1).equals("4") || year.substring(year.length() - 1).equals("5"))
            return DayOfWeek.WEDNESDAY.getDisplayName(TextStyle.FULL, LocaleContextHolder.getLocale());
        if (year.substring(year.length() - 1).equals("6") || year.substring(year.length() - 1).equals("7"))
            return DayOfWeek.THURSDAY.getDisplayName(TextStyle.FULL, LocaleContextHolder.getLocale());
        if (year.substring(year.length() - 1).equals("8") || year.substring(year.length() - 1).equals("9"))
            return DayOfWeek.FRIDAY.getDisplayName(TextStyle.FULL, LocaleContextHolder.getLocale());
        throw new IllegalArgumentException("Vehicle has an invalid year: " + vehicle.year);
    }

    public Boolean getRodizioAtive(VehicleDTO vehicle, LocalDate now) {
        DayOfWeek dayOfWeek = now.getDayOfWeek();
        return dayOfWeek.getDisplayName(TextStyle.FULL, LocaleContextHolder.getLocale()).equals(vehicle.rodizioDay);
    }

}
```
A ideia é puxar do ano do carro a parte inicial, verificar o último dígito e baseado nisso retornar o dia em formato de string, interessante notar o uso do LocaleContextHolder que é uma implementação do Spring que permite que o cliente mande pelo cabeçalho da requisição a linguagem que deseja receber suas respostas, portanto o dia retornado pela API será na linguagem requerida.

A outra função apenas compara o dia passado como parâmetro com o dia de rodízio armazenado no veículo, caso sejam iguais retorna true senão retorna false.

![Resultado de testes do service](/images/testes_com_sucesso.png "Resultado de testes do service")
*<center>Figura 9. Resultado de testes do service</center>*

![Retornando veículos em espanhol](/images/retornando_veículos_em_espanhol.png "Retornando veículos em espanhol")
*<center>Figura 9. Retornando veículos em espanhol :)</center>*
## Error Handling
TERMINAMOS NOSSA API SOLTEM FOGOS!!!!!!!!!!!!!!!!!!

Quê? não? como assim?

Fizemos todo o caminho feliz contando com a boa vontade de nosso usuário em inserir e requerer dados da forma correta para que tudo dê certo.
Mas e quando o mundo não for tão bonito assim?

Nesse caso precisamos nos preparar para que quando nossa aplicação falhe nosso usuário tenha uma descrição clara do que deu errado, ao invés de receber logs confusos e infinitos.

Dessa forma vamos criar interceptadores afim de que quando um determinado tipo de erro seja encontrado, uma mensagem amigável seja retornada ao consumidor da API.
```Java
@RestControllerAdvice
public class ValidationHandler {
    @Autowired
    private MessageSource messageSource;

    @ResponseStatus(code = HttpStatus.BAD_REQUEST)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public List<FormErrorDTO> handle(MethodArgumentNotValidException exception) {
        List<FormErrorDTO> dto = new ArrayList<>();
        List<FieldError> fieldErrors = exception.getBindingResult().getFieldErrors();
        fieldErrors.forEach(e -> {
            String message = messageSource.getMessage(e, LocaleContextHolder.getLocale());
            FormErrorDTO erro = new FormErrorDTO(e.getField(), message);
            dto.add(erro);
        });
        return dto;
    }

    @ResponseStatus(code = HttpStatus.BAD_REQUEST)
    @ExceptionHandler(IllegalArgumentException.class)
    public String handle(IllegalArgumentException exception) {
        String message = exception.getMessage();
        return message;
    }
}
```

A principio tratamos 2 erros, quando o usuário passa strings vazias em nosso endpoint para POST, e para quando alguma coisa errada acontece em nossa busca na API, por exemplo nosso usuário ter passado uma marca inexistente.

No primeiro caso criamos uma DTO para dizer como iremos retornar ao usuário nosso erro, aqui temos um campo que diz qual campo da requisição estava errado e um outro campo que armazena a mensagem que descreve o erro.

Posteriormente, listados todos os erros que foram expelidos pela API, em seu Bean Validation (lembra dele?), para cada erro armazenamos sua mensagem e a linguagem que o usuário nos passou, finalmente retornamos essa lista de erros como um JSON.
![Retornando erros em strings vazias](/images/erros_em_strings_vazias.png "Retornando erros em strings vazias")
*<center>Figura 10. Retornando erros em strings vazias</center>*
No segundo caso quando uma exceção de argumento inválido é lançada simplesmente retornamos o texto da exception que criamos no serviço.
![Retornando erros em strings inválidas](/images/erro_valores_invalidos.png "Retornando erros em strings inválidas")
*<center>Figura 11. Retornando erros em strings inválidas</center>*

Voilá, temos uma API que trata seus erros e se preocupa com informar seus consumidores com mensagens que o ajudem a entender o que ocorreu.

Espero que esse post tenha ajudado a mostrar formas de abordar a construção de uma API REST, alguns problemas comuns e ajude a entender para que servem as principais tecnologias desse universo.

É importante levar em consideração o contexto de sua aplicação sempre para tomar as melhores decisões sobre tecnologias e formas de lidar com determinadas situações, então tudo que está aqui descrito é um grande depende e está condicionado a nosso domínio da aplicação.
