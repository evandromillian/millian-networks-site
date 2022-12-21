---
title: "Desenvolvendo um sistema de matchmaker em Myridian: The Last Stand - Parte 1"
date: 2022-08-11T18:30:46-03:00
description: ""
draft: false
author: "Evandro Millian"
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image-preview.webp"
  
tags: ["AWS", "Serverless", "Matchmaker"]
categories: ["System Development"]
lightgallery: true
---

Um dos pontos mais complexos no desenvolvimento do nosso jogo foi o **matchmaker**. Para quem não conhece, este é o sistema que coloca os usuários em salas, baseado em personagens, habilidade entre outras informações. 

Existem vários sistemas de **matchmaker** no mercado, porém eles são muito simples, apenas recebem as informações e organizam jogadores baseado em níveis de habilidade, quantidade de vitórias, etc. Nós queríamos um sistema que coordenasse todo o fluxo de informações, preparasse as salas, para em seguida alocar o servidor e que receberia as informações da partida. Geralmente em jogos como **League of Legend** ou **Fortnite**, entre outros, o matchmaker apenas conecta os jogadores com o servidor de jogo, que por sua vez interage com os jogadores para criar os times e configurar os personagens, entre outras coisas, 

Nosso planejamento era que o servidor de jogos fosse o mais simples possível, então toda a  lógica descrita foi implementada no matchmaker. Adicionalmente, é muito mais simples, fácil e barato contratar pessoas para trabalhar com provedores de nuvem e desenvolver em **Typescript**, do que trabalhar com **Blueprints** na **Unreal Engine**. Então essa foi nossa primeira decisão de arquitetura em alto nível.
Dividimos então o desenvolvimento em duas partes, uma tratando a conexão do cliente com o sistema (vamos chamar de **front-end**) e a outra processando as requisições dos jogadores (vamos chamar de **back-end**).

### Front-End

Desenvolvemos três versões do **front-end**, sendo a primeira criada como uma API REST chamada pelo cliente, com endpoints **REST** no **API Gateway** invocando funções **Lambda**. Essa versão funcionou muito bem, porém o cliente acessava o sistema via pooling de chamadas **HTTP**, então havia um delay de alguns segundos para a informação chegar ao cliente, além de múltiplas chamadas ao sistema durante todo o tempo que o jogador estava conectado. Embora a **API** fosse muito simples, existia este problema de performance, mas o que motivou a evolução foi o fato de que o **matchmaker** não conseguia se comunicar ativamente com o cliente.

No desenvolvimento da segunda versão, utilizamos um sistema de publish-subscribe de uma empresa chamada **PubNub**, porém durante os testes tivemos problemas de delay e performance, com as mensagens demorando demais para atingir os clientes, com isso esta versão foi rapidamente desprezada.

Em seguida nós pensamos então na versão atual, que utiliza **WebSocket** gerenciado também pelo **API Gateway**. Desta forma o cliente apenas envia uma mensagem solicitando uma partida, e o sistema envia ativamente para os clientes as seguintes mensagens:

1. Partida criada pelo sistema
2. Time escolhido por cada jogador
3. Herói escolhido por cada jogador
4. Confirmação de cada jogador
5. Informações de conexão ao servidor de jogo

Desta forma, não existe mais pooling, reduzindo a quantidade de chamadas ao sistema, diminuindo o custo, além da simplificação da lógica do lado do cliente.

### Back-End

A lógica do **back-end** teve duas versões. A primeira era uma aplicação **Typescript** executando consultas complexas no **DynamoDB**, rodando periodicamente em um servidor físico alocado também na **AWS**. Embora funcionasse perfeitamente, ter uma máquina física executando uma aplicação tão pequena era um desperdício de dinheiro. 

Nós queríamos que a lógica toda fosse executada no ambiente **Lambda**, porém havia uma questão técnica importante: o tempo mínimo entre execuções para tarefas agendadas no **AWS Lambda** é de 1 minuto, o que é inaceitável, por os usuários teriam que aguardar bem mais de 1 minuto para entrar em uma partida, considerando todo o tempo de escolhas de jogadores, alocação de servidor e conexão dos clientes.
Tínhamos três opções para reduzir o tempo entre execuções do matchmaking:

* Uma função **Lambda** executando a lógica várias vezes durante o minuto de execução, aguardando alguns segundos entre cada execução;
* Uma função **Lambda** enviando mensagens agendadas para uma fila do **Simple Queue Service**, com a lógica executada pelo handler da fila.
* **DynamoDB Stream** tratando elementos com TTL (time-to-live) do **DynamoDB**

Decidimos utilizar a segunda opção devido ao custo menor (pelo menor tempo de execução), simplicidade e também por ser uma opção ainda não analisada por outros blogs especializados no assunto. Configuramos a função agendada para enviar 6 mensagens com delay de 10 segundos entre elas, desta forma a lógica de matchmaking seria executada a cada 10 segundos.

Uma outra tarefa foi utilizada para tratar o comportamento da sala após sua criação, aguardando as escolhas dos jogadores ou o timeout configurado, em que escolhas aleatórias são  forçadas ao jogador. Esta tarefa é executada periodicamente por sala criada, e dura apenas entre o momento que os jogadores fazem as escolhas para a partida até a conexão ao servidor da partida.

### Fluxo de Chamadas

Com o sistema descrito, o fluxo que a aplicação segue hoje é o seguinte:

{{< figure src="/images/myridian-matchmaker/Myridian_Matchmaker.webp" >}}

1. **Cliente** faz login no **PlayFab** e recebe um ticket de sessão
2. Utilizando o ticket de sessão, o cliente conecta à **API WebSocket**, que então verifica o ticket
3. **Cliente** solicita uma nova partida; sistema armazena a solicitação no banco
4. Periodicamente, a **Matchmaking Task** obtém as solicitações dos clientes, e agrupa em uma sala
5. **Matchmaking Task** notifica o cliente que a sala foi criada
6. **Matchmaking Task** cria a task que monitora a sala e recebe as escolhas dos jogadores
7. **Cliente** envia as escolhas de time, personagem e build; sistema faz o broadcast das informações para os outros jogadores
8. Após receber as escolhas dos jogadores (ou após um tempo determinado), a **Waiting Room Task** solicita que o **GameLift** aloque um servidor para a partida com as escolhas dos jogadores
9. **Waiting Room Task** envia para os **Clientes** as informações de conexão ao servidor da partida
10. **Clientes** se conectam ao servidor e a partida inicia

Concentramos neste artigo sobre a arquitetura de alto nível e os serviços AWS utilizados. No próximo post falaremos um pouco sobre código, otimizações de consultas ao **DynamoDB** e a funcionalidade de escolha de times pelos jogadores, que foi a última adição ao sistema. Até lá.


