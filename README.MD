#JMS com ActiveMQ

<p>Download ActiveMQ <a href="https://activemq.apache.org/components/classic/">aqui.</a></p>
<p>Execução do ActiveMQ: </p>

~~~bash
./activemq console
~~~

##Explicações

* Fila (Queue):
As filas dividem as mensagens entre os consumidores, uma mensagem x só poderá ser entregue a um dos consumidores interessados.
* Topico (Topic): 
Os topicos distribuem as mensagens entre os consumidores, assim a mesma mensagem pode ser entregue a mais de um consumidor interessado. 
O consumidor de um topico precisa se indentificar ao menos uma vez para que as mensagens sejam guardadas quando o consumidor estiver indisponível. 
Existe também a possibilidade de criar um selector para um consumidor, assim serão consumidas apenas as mensagens que satisfazem a condição pré estabelecida, os selectors possuem uma sintaxe parecida com SQL e buscam valores apenas nos headers (cabeçalho) e properties (propriedades) das mensagens.


#Adicionando restrição de acesso

Definindo usuários e grupos
Para definir usuários e seus grupos devemos usar plugins do ActiveMQ, no nosso caso um plugin de autenticação e outro de autorização. O primeiro passo é adicionar no arquivo conf/activemq.xml os usuários, senhas e seus grupos através do <simpleAuthenticationPlugin>.

No XML, logo após o elemento <broker ...> adicione:

~~~xml
<plugins>
  <simpleAuthenticationPlugin anonymousAccessAllowed="false">
    <users>
        <authenticationUser username="admin" password="admin" groups="users,admins"/>
        <authenticationUser username="user" password="senha" groups="users"/>
        <authenticationUser username="guest" password="senha" groups="guests"/>
    </users>
  </simpleAuthenticationPlugin>

  <!-- aqui vem ainda o authorizationPlugin -->
</plugins>
~~~
Repare que configuramos anonymousAccessAllowed="false". Isso significa que não podemos mais conectar ao ActiveMQ sem ter um usuário definido. Ao executar o código Java que realiza a conexão, recebemos uma exceção:

~~~java
java.lang.SecurityException: User name [null] or password is invalid.COPIAR CÓDIGO
~~~

Faz todo sentido pois agora devemos nos autenticar!

Usando uma conexão com usuário e senha
Para definir o user name e password devemos mexer no código Java. Para nos conectar ao ActiveMQ sempre usamos o método createConnection() da interface ConnectionFactory. Esse método é sobrecarregado e possui uma versão que recebe o user name e password:

~~~java
Connection connection = cf.createConnection("user", "senha");
~~~
Ao reiniciar o ActiveMQ com plugin de autenticação configurado podemos estabelecer uma conexão autenticada através do código Java!

Definindo permissões
Agora só falta dizer quais são as permissões de cada grupo. Existem 3 permissões disponíveis:

leitura (read);
escrita (write);
administrativa (admin).
Cada permissão fica associada a um tópico/fila e aos seus grupos. Repare que usamos os grupos users, admins e guests. Sabendo disso, segue um exemplo de autorização para o tópico comercial:

<authorizationEntry topic="comercial" read="users" write="users" admin="users,admins" />COPIAR CÓDIGO
O grupo users pode ler, escrever e administrar o tópico e o grupo admin também possui a permissão administrativa.

A configuração completa deve ser feita dentro do arquivo conf/activemq.xml, dentro do elemento <plugins>. No nosso exemplo o grupo users tem acesso completo às destinations financeiro e comercial:

~~~xml
<authorizationPlugin>
    <map>
      <authorizationMap>
        <authorizationEntries>
          <authorizationEntry queue="fila.financeiro" read="users" write="users" admin="users,admins" />
          <authorizationEntry topic="comercial" read="users" write="users" admin="users,admins" />
          <authorizationEntry topic="ActiveMQ.Advisory.>" read="users,admins" write="users,admins" admin="users,admins"/>
        </authorizationEntries>
        <tempDestinationAuthorizationEntry>
          <tempDestinationAuthorizationEntry read="admin" write="admin" admin="admin"/>
        </tempDestinationAuthorizationEntry>
      </authorizationMap>
    </map>
</authorizationPlugin>
~~~
Além do nosso tópico comercial e a fila financeira, existe uma terceira configuração relacionada ao tópico ActiveMQ.Advisory.. Esse tópico já existe por padrão no ActiveMQ e recebe mensagens administrativas (AdvisoryMessage) sempre que for criado um consumer, producer e um novo destination. Mais infos em: <a>http://activemq.apache.org/advisory-message.html

Uma vez feita toda a configuração dos plugins, basta reiniciar o ActiveMQ. Além disso, como já falamos, devemos usar no código Java o usuário e a senha para nos conectar corretamente com a fila financeiro ou com o tópico comercial.

Você pode ver a configuração completa nesse arquivo: activemq.xml