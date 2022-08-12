---
title: "Portfolio"
date: 2022-06-28T09:33:29-03:00
draft: false
---
 
### Myridian: The Last Stand

{{<figure src="/images/portfolio/myridian_wallpaper-small.png" >}}

**Myridian: The Last Stand** é um jogo de plataforma móvel da velha escola com impressionantes pixelarts 2D, uma trilha sonora orquestral e ação intensa.

Impulsionado por uma história que exige a interação dos jogadores para evoluir, **The Last Stand** é o primeiro capítulo da missão de Nettor de despertar os Verges e dar vida a heróis lendários para que possam enfrentar seu maior desafio.

{{<figure src="/images/portfolio/Carlo_Slash_AttackA.webp" caption="Carlo attacking">}}

No final de cada capítulo, o progresso dos jogadores ao atingir seus objetivos ditará como novos heróis, missões e eventos serão inseridos no enredo. Os jogadores mais experientes serão convidados a participar de missões individuais ou cooperativas e seus resultados conduzirão a história.

#### Lessons Learned

O projeto exigiu uma extensa pesquisa em muitas áreas. Apesar de todos os recursos e funcionalidades disponíveis, **Unreal Engine** não é otimizado para jogos 2D, faltando algumas ferramentas essenciais.

###### 1. Animações

{{<figure src="/images/portfolio/Neeno_Carlo_Donna_RunCycle.webp" alt="Heroes running" title="Heroes running">}}

Devido à falta de uma versão do controlador de animação para **Paper2D**, encontramos um plugin chamado **PaperZD**, expondo uma máquina de estado para controlar a transição entre animações da mesma forma que está disponível para malhas de esqueleto.

###### 2. Paletas de Cores & Skins

{{<figure src="/images/portfolio/Neeno-Alternate_Skins.png" alt="Neeno skins" title="Neeno skins" width="450">}}

As paletas de cores foram implementadas usando uma combinação de cálculos de materiais e preparações de textura. As cores da textura atuam como índices para selecionar cores de uma paleta nos materiais.

Skins foram implementadas sobrepondo sprites contendo as armas e peças de armadura, utilizando o mesmo mecanismo de paleta para os personagens.

###### 3. Navegação & IA

Como o **Unreal Engine** não possui navegação 2D, a equipe implementou uma lógica de navegação personalizada usando pontos de caminho vinculados entre eles, com instruções adicionais no personagem deve pular ou cair para o próximo ponto.

Para controlar os personagens, o projeto utilizou controladores de IA usando pontuação de gol, que é muito mais simples do que a funcionalidade de Behavior Trees existente.

###### 4. Multiplayer

{{<figure src="/images/portfolio/Fay-KillingSpree.webp" alt="Multiplayer fight" title="Multiplayer fight" width="500">}}

A lógica multiplayer foi usada para movimentação de heróis e inimigos, projéteis e plataformas móveis, bem como para replicar o dano causado e recebido entre todos os personagens.

Implementamos técnicas avançadas para reduzir a transmissão de dados entre os peers e o servidor e aumentar a capacidade de resposta percebida pelos jogadores.

O jogo utilizou servidores dedicados hospedados no **AWS** **GameLift**, e foi implementado com o mínimo de código possível, recebendo todas as informações sobre a partida do matchmaker.

###### 5. Matchmaking e Backend

{{<figure src="/images/portfolio/Myridian_teste_escolha_de_time.webp" alt="Team selection" title="Team selection" width="500">}}

O matchmaker exigia um aplicativo que utilizasse serviços em nuvem para comunicação rápida, grande disponibilidade e custo reduzido. Ele usa uma combinação de **DynamoDB**, **Simple Queue Service** e **WebSockets** controlados pelo **API Gateway**.

Para gerenciar as contas dos jogadores, utilizamos serviços prontos para uso com armazenamento de dados unificado, single sign on e leaderboards, permitindo que clientes de diferentes plataformas acessem os mesmos dados.

#### Future roadmap

Atualmente estamos trabalhando em um modo de histórico de um jogador, permitindo que outros jogadores experimentem o jogo sem a necessidade de se conectar a um servidor multiplayer e encontrar pessoas para jogar.