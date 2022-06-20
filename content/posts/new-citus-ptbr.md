+++
title = "Citus 11.0 (PT-BR)"
date = "2022-06-20T13:06:20-03:00"
author = "Bruno Panuto"
authorTwitter = "panuto_"
cover = ""
tags = ["citus", "postgres", "dev"]
keywords = ["database"]
description = "Uma nova e incrível versão de extensão do Postgres que vale a pena!"
showFullContent = false
readingTime = false
hideComments = false
+++

Nova versão do Citus a caminho! A versão 11 dessa extensão do Postgres me pareceu tão legal que gostaria de falar um pouco sobre ela.

Para quem não a conhece, o Citus é uma extensão do Postgres que permite criar tabelas distribuídas em um cluster de Postgres. É um problema de escala: bancos relacionais tendem a ser máquinas grandes, que eventualmente atingem um limite tanto de processamento quanto de capacidade de armazenamento. Poder distribuir esses dados permite uma escalabilidade maior sem precisar sair do SQL (embora ainda seja necessário arquitetar sua solução com isso em mente).

Duas novas features me chamaram a atenção, e assim como uma mudança do Citus em relação ao modelo de negócios, por assim dizer.

A primeira feature: Citus não mais bloqueia escritas e leituras no rebalanceamento dos shards na versão 11.0. Isso era algo que já existia, mas que agora vem para a versão open source da extensão. Bloquear nem sempre é ruim, mas quando a sua escala é grande um bloqueio pode gerar latência que impacta o seu negócio. Shard rebalancing agora usa replicação lógica para evitar o bloqueio de escritas e leituras!

A segunda feature: agora, qualquer nó Citus pode fazer queries distribuídas de escrita e leitura. Em versões antigas, o Citus requer que queries passem por um "nó coordenador". É um padrão comum em bancos de dados distribuídos, como Cassandra ou DynamoDB: um ponto centralizador que conhece e monitora os outros nós do sistema é necessário para distribuir as queries ou encontrar os nós relevantes de uma shard key. O nó coordenador do Citus ainda existe e ainda é necessário para mudanças no schema, mas como o schema agora é sincronizado entre os nós, qualquer nó permite leituras e escritas distribuídas. Isso é bom pois evita a sobrecarga de apenas um nó, enquanto remove o SPOF (single point of failure) do seu cluster.

E agora, a mudança interessante que não é técnica mas é bacana de comentar: o Citus é 100% open source software! Shard rebalancing com replicação lógica era algo que já existia, como comentei, porém apenas na versão "enterprise" da extensão. Agora, essa e todas as outras features que apenas existiam para clientes com bolso fundo são 100% abertas e disponíveis. Isso é uma mudança interessante, pois pode alavancar o uso da ferramenta e permitir que mais pessoas e empresas a usem. Eu sou um grande fã de open source, então vejo essa mudança com ótimos olhos.

E você? Conhecia essa extensão? Já precisou de algo desse tipo? Tem alguma dúvida? Me manda uma mensagem ou no Twitter ou no Linkedin!
