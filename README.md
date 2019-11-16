# Hyperledger Composer - ICOLAB

## Pré-requisitos

The following are prerequisites for installing the required development tools:

- Operating Systems: Ubuntu Linux 18.04
- Docker Engine: Version 17.03 ou +
- Docker-Compose: Version 1.8 ou +
- Node: 8.9 or + (versão 9 não é suportada)
- NPM: v5.x
- GIT: 2.9.x ou +
- Python: 2.7.x
- IDE de sua escolha (usaremos VSCode).

Usando Ubuntu, execute:
```
curl -O https://hyperledger.github.io/composer/latest/prereqs-ubuntu.sh

chmod u+x prereqs-ubuntu.sh
```

Depois, execute o script:
```
./prereqs-ubuntu.sh
```

## Components

Utilização dos componentes composer-cli, generator, rest-server e Yeoman

Use os comandos (sem utilizar *su* ou *sudo*)

1 - Composer CLI
```
npm install -g composer-cli@0.20
```

2 - Helper para execução de REST server
```
npm install -g composer-rest-server@0.20
```

3 - Utility para geração de aplicações composer
```
npm install -g generator-hyperledger-composer@0.20
```

4 - Yeoman, para criação genérica de aplicações
```
npm install -g yo
```

## Instalar Playground
Criaremos modelo similar ao visto na WEB
```
npm install -g composer-playground@0.20
```

## Instale (ou use) sua IDE
Como exemplo, usaremos VSCode

https://linuxize.com/post/how-to-install-visual-studio-code-on-ubuntu-18-04/

## Instalar Hyperledger Fabric

Agora faremos o download dos scrip files para downoad de Hyperledger Fabric
```
mkdir ~/fabric-dev-servers && cd ~/fabric-dev-servers

curl -O https://raw.githubusercontent.com/hyperledger/composer-tools/master/packages/fabric-dev-servers/fabric-dev-servers.tar.gz
tar -xvf fabric-dev-servers.tar.gz
```
Depois de carregar os scripts, vamos partir para o download dos binários e docker files para Hyperledger Fabric:
```
cd ~/fabric-dev-servers
export FABRIC_VERSION=hlfv12
./downloadFabric.sh
```

## Iniciando uma rede Hyperledger Fabric
Toda vez que for iniciar uma nova runtime, será necessário inicializar o script, e então gerar um PeerAdmin card:
```
cd ~/fabric-dev-servers
export FABRIC_VERSION=hlfv12
./startFabric.sh
./createPeerAdminCard.sh
```
É possível então parar e iniciar o runtime usando os scripts **stopFabric.sh** e **startFabric.sh**.

Após finalizar todo process, pode-se finalizar os docker files rodando **tearDownFabric.sh**.

## Iniciando o "Playground" local
Para dar start no app, execute:
```
composer-playground
```
O app deve então estar acessível no endereço https://localhost:8080/login

Para literalmente destruir todo detup feito até reference a docker images, pode-se executar:
```
docker kill $(docker ps -q)
docker rm $(docker ps -aq)
docker rmi $(docker images dev-* -q)
```

## Desenvolvedor - Criar uma BNS
Primeiramente, vamos criar uma business network structure:

1 - Crie uma instalação modelo
```
yo hyperledger-composer:businessnetwork
```

2 - Digite **icolab-network** para *network-name*, e qualquer nome para *descrition*, *author name* e *author email*

3 - Escolha **Apache-2.0** como licença

4 - Escolha **org.example.mynetwork** como namespace

5 - Escolha **No** quando perguntado se queres gerar o modelo vazio (empty)

## Desenvolvedor - Definindo uma BNS
Utilizando sua IDE, abra o recém-criado pacote do Composer

1 - Abra o aquivo model **org.example.mynetwork.cto**

2 - Substitua o conteúdo por:
```
/**
 * My commodity trading network
 */
namespace org.example.mynetwork
asset Commodity identified by tradingSymbol {
    o String tradingSymbol
    o String description
    o String mainExchange
    o Double quantity
    --> Trader owner
}
participant Trader identified by tradeId {
    o String tradeId
    o String firstName
    o String lastName
}
transaction Trade {
    --> Commodity commodity
    --> Trader newOwner
}
```

3 - Salve as alterações

4 - Abra o arquivo **logic.js**

5 - Altere o conteúdo para seja:
``` javascript
/**
 * Track the trade of a commodity from one trader to another
 * @param {org.example.mynetwork.Trade} trade - the trade to be processed
 * @transaction
 */
async function tradeCommodity(trade) {
    trade.commodity.owner = trade.newOwner;
    let assetRegistry = await getAssetRegistry('org.example.mynetwork.Commodity');
    await assetRegistry.update(trade.commodity);
}
```

6 - Salve as mudanças

7 - Altere o conteúdo do arquivo **permission.acl** para:
```
/**
 * Access control rules for tutorial-network
 */
rule Default {
    description: "Allow all participants access to all resources"
    participant: "ANY"
    operation: ALL
    resource: "org.example.mynetwork.*"
    action: ALLOW
}

rule SystemACL {
  description:  "System ACL to permit all access"
  participant: "ANY"
  operation: ALL
  resource: "org.hyperledger.composer.system.**"
  action: ALLOW
}
```

8 - Salve as alterações.

## Desenvolvedor - Gerando a nova BSN
1 - Acesse a pasta **icolab-network**

2 - Nela, execute o comando:
```
composer archive create -t dir -n .
```
Então o arquivo bna **icolab-network@0.0.1.bna** deve estar criado na pasta.

## Desenvolvedor - Deployando sua business network
1 - Primeiro, vamos instalar nosso bna
```
composer network install --card PeerAdmin@hlfv1 --archiveFile icolab-network@0.0.1.bna
```

2 - Agora, iniciaremos nossa bna
```
composer network start --networkName icolab-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file networkadmin.card
```

3 - Importaremos agora o network administrator como uma business network card utilizável, executando:
```
composer card import --file networkadmin.card
```

4 - Para verificarmos se tudo está OK, faremos um PING para nossa network:
```
composer network ping --card admin@icolab-network
```

## Desenvolvedor - Gerando um servidor REST
Para facilitar a acesso ao modelo, faremos agora uma REST API:
1 - Uma vez dentro da pasta **icolab-network**, execute:
```
composer-rest-server
```

2 - Digite **admin@icolab-network** como card name

3 - Selecione **Never use namespace**

4 -Somente selecione **Yes** para seção sobre *enable event publication*

## Desenvolvedor - Gerando aplicação Angular
1 - Para criar uma alicaão-base sobre nossa REST API, execute:
```
yo hyperledger-composer:angular
```

2 - Selecione **Yes** para *connect to running network*

3 - Digite os dados para o arquivo *package.json*

4 - Digite **admin@icolab-network** como business card

5 - Selecione **Connect to to existing REST API**

6 - Digite **http://localhost** para REST server address

7 - Digite **3000** como porta

8 - Selecione **Namespace are not used**

9 - Para rodar a aplicação, entre na *pasta do projeto Angular* e execute **npm starrtt**. Isto executará a aplicação, que estará rodando sobre a REST API no endereço *https://localhost:4200*

## Desafio 

Vamos definir outro modelo, resolvendo o seguinte problema:

1 - Uma empresa de containers que alugar espaço de seu navio

2 - Essa mesma empresa deseja fazer com que as outras que querem alugar espaço sejam servidas no modelo FIFO, mas hoje dia tudo é feito manualmente, via telefone, a se alguém esquece uma empresa que pediu previamente a outa pode ser prejudicada

3 - Cada navio suporta 3 espaços para reserva

4 - Os agentes envolvidos são: **Empresa de container**, que aluga espaços; **Empresa dona do navio**, que fornece 3 espaços por navio; e as **Empresas que solicitam espaço** via empresade container.