# Criando uma API com Spring Boot + Hibernate

Conhecer tecnologias e aprender novos contextos é sempre revigorante e nada melhor para utilizar nesse momento do que um projeto brinquedo para testar os conhecimentos.

Neste post vamos destrinchar o mundo do Java para construção de API's com uma API de gerenciamento de veículos.
Tecnologias Utilizadas:

* Maven: Um gerenciador de pacotes que vai facilitar o processo de instalação de dependências.
* Spring Boot: Um framework mundialmente difundido para construções de aplicações web dos mais variados tipos em Java.
* Hibernate: Uma biblioteca de abstração de mapeamento Objeto-Relacional
* H2: um banco de dados em memória (para fins de simplicidade)
* Uma IDE de seu feitio (Eclipse,Intellij e para os mais ousadas até mesmo um vscode :)).
## Conhecendo nosso escopo 
Claro que antes de começar nosso desenvolvimento precisamos saber o que vamos desenvolver, para este artigo temos um contexto pré-determinado que iremos seguir para aprendizado.
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
Vamos começar criando nossas classes que vão mapear nossos dados para o banco de dados. O spring JPA é uma implementação da JPA que utiliza o Hibernate e que auxilia a manipulação de objetos e seu mapeamento para o banco de dados, para utilizá-lo vamos criar nossas classes User e Vehicle.
Classe Users
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
Classe Vehicle
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
Nestas classes estamos mapeando as entidades que vamos manipular no banco de dados, com as anotações definidas pelo caractere @ mapeamos diretrizes para que o spring JPA gere determinadas facilidades para nosso código.

Inicialmente temos a anotação *@Entity* que define uma determinada classe como uma entidade da JPA, em seguida mapeamos essa entidade para uma tabela com *@Table*, por padrão o JPA tenta mapear uma entidade para uma tabela com o exato mesmo nome, porém como queremos manter nomes diferentes é necessário passar o atributo *name* como parâmetro, o mesmo ocorre com a anotação *@Column* que mapeia uma variável a uma coluna de uma determinada tabela em nosso banco.

Após esta marcação inicial precisamos declarar nossos atributos do banco de dados como variáveis em nossa classe (aqui o tipo de acesso está público para todas as variáveis a título de simplificação já que não serão utilizados conceitos de encapsulamento).

É também necessário definir qual de nossas variáveis é a chave primária da entidade no banco de dados com a anotação *@Id*, além dela precisamos marcar que este valor será gerado automaticamente pelo nosso banco e que por isto será passado como nulo à nossa entidade, com isso precisamos passar a estratégia de geração de valor (que nesse caso será a identity) e finalizamos a descrição do nosso ID.

Além disso existe o mapeamento dos nossos relacionamentos, como um veículo possui um dono e um usuário pode ter N veículos é necessário escrever nas duas entidades anotações que mapeiem este relacionamento Um para Muitos, para isso se insere a anotação @ManyToOne e a anotação @OneToMany, sendo necessário na segunda especificar qual campo está mapeando esta relação para evitar duplicidades.

Por fim precisamos apenas mapear nossos construtores (lembrando que a JPA necessita de um construtor vazio) e temos nossas entidades construídas.
## Criando o banco de dados em memória
Bom nosso modelo agora mapeia nossas classes para as tabelas do banco de dados... mas que banco de dados? afinal ainda não criamos nada que seja utilizado por esse mapeamento como banco de dados. Bom ai que entra nosso querido H2, poderiam ser utilizados vários bancos de dados aqui você provavelmente deve conhecer vários como MySQL,SQL Server, Mongodb etc, porém como queremos nos concentrar na API e na usabilidade a ideia é utilizar um banco de dados que seja descartável e demande pouco esforço.

Considerando esses pontos utilizaremos o H2 como um banco de dados em memória, isso significa que quando o nosso servidor for iniciado criaremos um banco de dados ali na memória para ser utilizado temporariamente e que quando nosso servidor for desligado ou reiniciado ele deixará de existir.
E como fazer isso? Bom o spring nos ajuda dentro do diretório resources existe o arquivo de configuração application.properties em que podemos definir nossas configurações de banco de dados para uso do Hibernate.
``` 
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
Nele dizemos quais drivers utilizar para criar uma url de acesso e criá-lo em memória o username e password e de quebra conseguimos declarar até uma URL para acesso a interface gráfica do banco.

Bom tudo bem bonito mas como inserimos nossos dados e tabelas? Mais uma vez o spring salva nossa vida, por padrão o spring executa nesse mesmo diretório resources o arquivo como nome data.sql, basta criar este arquivo com os códigos SQL que queremos executar em nosso banco e a magia ocorre.
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
Com isso quando subirmos nosso servidor nosso banco de dados terá dados iniciais e poderá ser acessado pelo link http://localhost:8080/h2-console.

![Tela de login do banco de dados acessado pelo navegador](/images/login_h2_console.png "Tela de login do banco de dados acessado pelo navegador")
*<center>Figura 2. Tela de login do banco de dados acessado pelo navegador</center>*
![Interface do banco de dados em utilização](/images/tela_h2_em_funcionamento.png "Interface do banco de dados em utilização")
*<center>Figura 3. Interface do banco de dados em utilização</center>*
## Acessando o banco de dados 
E agora como puxar os dados do banco? precisamos consumir essas entidades e mandar os resultados dela para nosso usuário por meio dos endipoints de nossa API,para isso vamos utilizar o padrão repository em que isolamos o código de acesso ao banco de dados em uma classe específica (uma para cada funcionalidade) a fim de desacoplar código e aproveitar das interfaces de auxílio da JPA.

Classe repository de usuários
``` Java
public interface UsersRepository extends JpaRepository<User, Long> {

    User findByCPF(int cpf);

}
```
Classe repository de veículos
```Java
public interface VehiclesRepository extends JpaRepository<Vehicle, Long> {

    List<Vehicle> findByOwnerCPF(int cpf);

}
```
Aqui temos nossos dois repositorys e é onde a implementação da JPA do Spring se faz valer, com ela quando instanciarmos o repository ele terá implementado todos os métodos da JpaRepository, esses métodos fazem operações básicas como criar modificar remover e retornar objetos do nosso banco de dados, abstraindo essa lógica e reduzindo código repetitivo no acesso ao banco de dados, necessitando apenas da Entidade que será consumida e do tipo do identificador único dela.
Além disso conseguimos criar métodos próprios descrevendo apenas a assinatura do método, para isso basta seguirmos o padrão da biblioteca como por exemplo na função *findByCPF* e completarmos o nome com o campo que queremos manipular com a função (neste caso CPF).

Caso o retorno seja de um campo dentro de uma entidade relacionada como no segundo repository, precisamos mapear no nome a entidade que será acessada e o campo dentro dela para que o mapeamento se dê corretamente, vale ressaltar que para evitar ambiguidades caso exista um campo na entidade "mãe" cujo nome seja igual ao retorno de Entidade + Nome campo (caso houvesse um campo OwnerCPF na entidade Vehicle por exemplo) a ambiguidade pode ser retirada pelo uso do _ entre a entidade e o campo.

Aqui já temos os nossos repositorys para acesso ao banco agora precisamos consumi-lo e retornar nossos dados.
## Exibindo dados ao cliente
Finalmente a hora de ver coisas funcionando (já estava na hora né), agora precisamos acessar nosso repository e os métodos que essas interfaces implementam e exibi-los para nosso usuário, ou seja precisamos criar nossos endpoints.

Em nossos endpoints a ideia é consumir nosso repository relativo àquele endpoint e aqui temos um detalhe interessante, imagine que nossa entidade contenha campos que não deveriam ser expostos numa API pública, por exemplo se armazenassemos a senha na entidade ou se quisessemos que o CPF não fosse um atributo publico, retornar apenas a entidade para o cliente poderia gerar problemas e por isso criaremos classes DTO.
### Classes DTO
A ideia aqui é criar uma visualização especifíca da nossa entidade e retornar um objeto deste tipo para que acessa nosso endpoint, isso desacopla o retorno de dados das nossas entidades da JPA e possibilita que tenhamos diferentes objetos de retorno para a mesma entidade.
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
Cada DTO nossa possui um construtor que recebe seu correspondente em entidade e armazena os dados necessários em si, no caso do DTO de veículos os campos rodizioDay e rodizioIsAtive recebem valores calculados em uma implementação que veremos mais à frente.
Além disso temos um novo tipo declasse utilizada aqui que são os mappers.

### Classes Mapper
Em nossa implementação da API precisaremos passar uma lista de veículos e receber uma lista de DTO, cada DTO sendo um veículo com seus dados, para isso criaremos classes que possuam métodos que convertam nossa lista de usuários e isolaremos isso em classes que chamaremos de mapper.

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
Aqui novamente precisaremos de nosso serviço inserindo dados específicos mass neste momento estamos prontos para criar os endpoints.
## Criando os endpoints de retorno 
Agora finalmente criaremos nossos controllers que são as classes que o spring irá chamar de acordo com a URL mapeada, neste primeiro momento criaremos os 3 endpoints de retorno, que mapeiam veículos,usuários e todos os veículos de um determinado usuário dado seu CPF.
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
Neste trecho aqui inserimos a anotação que diz ao Spring que nossa classe é um controller da nossa API para que o spring se encarregue do mapeamento de nossas rotas, também injetamos nosso repository para que quando nosso controller seja criado os métodos do nosso repository estejam disponíveis.
Após isso inserimos nossa anotação que especifica que quando a rota /users for chamada pelo método GET aquele método será executado, criamos uma lista de usuários e chamamos o método implementado pela JPA findAll() para retornar todos os usuários de nosso banco de dados, ciramos nosso mapper e retornamos o valor dessa conversão para a nossa rota.

Executando uma requisição para nossa rota temos esse resultado.

