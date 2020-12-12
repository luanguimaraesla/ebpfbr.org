---
author: "Lucas Severo"
date: 2020-10-24T18:51:01+02:00
title: "As principais razões pelas quais você deve dar uma chance ao eBPF"
title: "As principais razões pelas quais você deve dar uma chance ao eBPF"
description: This tutorial will show you how to create a simple theme in Hugo.
weight: 10
categories:
- development
tags:
- ebpf
- cilium
- hubble
- falco
- prometheus
- infrastructure
menu: main
---


Com este _post_, queremos mostrar as razões que fazem do eBPF uma importante ferramenta e os motivos que começam a despertar tanto interesse dentro da comunidade do Linux.

<!--more-->

*Adaptado de https://blog.container-solutions.com/the-top-reasons-why-you-should-give-ebpf-a-chance*

Antes de falarmos sobre essas tais razões, gostaria de lhe apresentar uma explicação básica sobre o que é o eBPF. Fique à vontade para pular esta seção se você já tem um conhecimento geral sobre o assunto.

A principal ideia por trás do eBPF é a possibilidade de enviar códigos arbitrários para serem executados em uma máquina virtual rodando dentro do Sistema Operacional. Podemos pensar essa máquina virtual como um conjunto de pseudo-instruções e recursos que compõem os blocos elementares de uma linguagem comum aos Linux. Esse conjunto de instruções será usado para criar programas que interagem com diversos recursos e eventos do Sistema Operacional.

Vamos descomplicar!

Para facilitar a compreensão, podemos enxergar essa máquina virtual como um pseudodispositivo programável dentro do Kernel. Agora imagine que esse dispositivo possui "sensores", registradores, memória, etc. Além disso, ele é capaz de executar programas que você escreve e que o instruirão sobre o que fazer: o que observar, modificar, descartar, armazenar, relatar e muito mais. No entanto, este dispositivo é **virtual** e roda dentro do Kernel. É basicamente isso!


Mas você poderia pensar: com esse crachá VIP vários programadores fariam estragos inimagináveis no sistema operacional! Entretetanto, o código eBPF criado e enviado do espaço de usuário para o espaço Kernel é executado com segurança no nosso pseudodispositivo programável porque este comporta um verificador muito robusto e restritivo. Esse verificador será executado sempre que tentarmos carregar novos códigos e garantirá as seguintes condições antes de compilar/executar:

* O programa não possui loops;
* Não há instruções inacessíveis;
* Os estados de registradores e pilha são válidos;
* Registradores com conteúdo não inicializado não são lidos;
* O programa só acessa estruturas apropriadas para seu tipo de programa BPF.

Portanto, essa simples máquina virtual está conectada a alguns *hooks* específicos dentro do kernel. A quantidade de *hooks* para eBPF está aumentando a cada dia por causa de suas [capacidades](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md). Existem muitos casos de uso possíveis para o eBPF, mas vamos nos concentrar em observabilidade para simplificar a explicação.

Aqui está um exemplo de fluxo BPF, do [blog do Brendan Gregg](http://www.brendangregg.com/ebpf.html), arquiteto de desempenho sênior da Netflix e um dos principais porta-vozes dessa tecnologia.

![](images/linux_ebpf_internals.png)

Se o código BPF que você escreve for aceito pelo verificador, ele pode ser anexado a algumas fontes de eventos como as da imagem acima:

* kprobes: rastreamento dinâmico do Kernel.
* uprobes: rastreamento dinâmico no nível do usuário.
* tracepoints: rastreamento estático do Kernel.
* perf_events: amostragem com timestamps e PMCs.

Depois de medir quaisquer dados dessas fontes de eventos, o programa pode então retornar esses dados ao espaço do usuário para que sejam mostrados ou processados. E isso é eBPF em poucas palavras!

Agora que você tem uma compreensão geral do que é o eBPF, vamos voltar ao porquê de dar uma chance a ele.

## Razão nº 1: _Às vezes, apenas o eBPF resolve o problema_

Embora possa parecer um exagero, o eBPF pode ser a melhor solução para diagnosticar e resolver vários problemas, devido à especificidade ou sobrecarga de desempenho.

**Rastreamento muito específico**

Esta anedota é um ótimo exemplo de eBPF resolvendo algo muito específico. Tudo começa com o seguinte tweet da Engenheira de Software Julia Evans:

{{< tweet 765666624968003584 >}}

Ela estava procurando uma ferramenta CLI que pudesse mostrar durações de conexão em uma determinada porta. O raciocínio inicial era verificar de forma fácil e rápida se algumas requisições HTTP eram realmente lentas. Nesta mesma thread de tweets, podemos ver Brendan Gregg sugerindo algumas perf-tools alternativas para obter essas informações.

Mas o que realmente empolgou foi [esta publicação](http://www.brendangregg.com/blog/2016-11-30/linux-bcc-tcplife.html) do Brendan Gregg sobre o lançamento do tcplife na coleção [BCC](https://github.com/iovisor/bcc) no final daquele ano.

Este é um exemplo de saída do tcplife:

![](https://lh6.googleusercontent.com/wUr0mIFU0NEcJ6HTaq9-nJLL5kNuFBTEW85vgtOFsDKNty4LoV_fSt3fHINYRldK_VtkVJ287vO9BSugcY5pSFTUptS3E3Vj11xk7EfmMtvzJ3e2yqdNyZSCdwx8uN02JUgAzO7l)


Eu gostaria de apontar algumas coisas interessantes sobre isso. O Tcplife mostra alguns dados interessantes, como ID e nome do processo; informações que você não obterá de uma saída do tcpdump, por exemplo. Para evitar sobrecarga de desempenho, essa ferramenta não rastreia pacotes - ela apenas instrumenta eventos que alteram o estado TCP, obtendo timestamps deles. E, claro, a funcionalidade que Evans pediu está presente, e você pode simplesmente verificar as durações de conexão na última coluna!

Como Brendan Gregg disse em algumas de suas palestras, construir coisas em torno do eBPF é como ter superpoderes ocultos. Você só precisa fazer as perguntas certas para fazer algo interessante sair disso.

**eBPF provê rastreamento (*tracing*) sem interromper produção.**

Quem um dia sentiu curiosidade de se aventurar na compreensão Linux, e mais especificamente sobre a depuração em *user space*, certamente conhece a ferramenta _strace_. Este era um comando muito interessante para executar em todos os lugares, visualizando quais chamadas de sistema disparadas para determinadas ações.

![](https://lh3.googleusercontent.com/V1q2hd_FzteIRSuwtRs9YkYmQQEjEM3c3otjUhbFoDfRnoWJPrUifvdk0ufWUo4SgE-12Q5rTufqpDEIjhLjYXd0XCgh9N_2A-vXlQ-4naPM8RCZbzuDo5jHJkgKJsRR-zgntn3-)

O que as pessoas se esquecem de dizer (ou talvez até digam, mas a gente simplesmente esquece) é que o _strace_ cria uma enorme sobrecarga no sistema. Até o Brendan Gregg já escreveu sobre isso em seu blog [strace Wow Much Syscall](http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html). Executar _strace_ para entender algo que está acontecendo em um ambiente de produção com alta carga pode ser catastrófico e pode simplesmente jogar seu servidor no chão. Claro, pode ser OK fazer isso se você tiver redundância ou se o seu servidor estiver inativo de qualquer maneira, mas nem sempre é o caso.

Em sua postagem, Brendan Gregg sugere algumas outras ferramentas no final, listando *perf_events, Sysdig, Ktap, SystemTap* e, em uma edição posterior da postagem, *perf_trace*. Mas esta postagem é de 2014, antes do merge do eBPF no Kernel. *perf_trace* é uma ferramenta completamente aceitável para evitar sobrecarga e fazer rastreamento, mas existe uma ferramenta de [*tracing*](https://github.com/iovisor/bcc/blob/master/tools/trace.py) no [BCC](https://github.com/iovisor/bcc) que considero mais interessante por causa do ecossistema eBPF, ainda voltaremos nesse assunto.

Vamos verificar um exemplo de uso semelhante de _strace_ e _trace.py_ do BCC. A primeira captura de tela aqui mostra o uso de strace para um comando _cat_ simples; o segundo mostra o uso do _trace.py_ do BCC também para um comando _cat_ simples.

![](https://lh3.googleusercontent.com/p4GHY6RAGmM19Chiuli30sIG6tmo6j1FlGd34ft9hfECcxcZHlpp-q4ihxGhigvOq3-NQ4s62q5wxNsmLmi5Gs7SCjFxSi7mzkNkuQ09C0vTAhP8VhKTkHqa35t9wyvbVWHNRlK0)

![](https://lh5.googleusercontent.com/iqaGobzk_pnP-xEFZAJtSLKMkHrcdGXtiaTjBw9Y3YRQ_5-edvl6t95pF0ALCKRieTj0aFoRfWrsoSXNtkHJAZfN6ACVwZ15v6PHeKwAKkdKswo7wUAhogLatEfmj5rgqCJasGrR)

Usar trace.py do BCC adiciona um pouco de complexidade de uso porque seu Kernel precisa ser recente, você precisa ter as ferramentas BCC instaladas e precisa entender seus argumentos CLI. Todavia, terá muito mais desempenho e gerará menos sobrecarga. Além disso, é mais flexível e poderoso, considerando o que investigar enquanto também considera filtros, pagers de buffer e outras configurações de *tracing*.

## Razão nº 2: _Importantes entidades estão usando o eBPF_

Certamente é um motivo para ficar de olho no eBPF agora. Cilium foi provavelmente a primeira equipe a construir um produto completo em torno de eBPF, investindo principalmente nas áreas de observabilidade e controle de rede, mas muitos jogadores de observabilidade têm caminhado nessa direção ultimamente.

Em 2017, Brendan Gregg deu muitas palestras sobre o eBPF e, como vimos em sua citação anterior, ele disse que o eBPF estava em um estágio em que tinha muitos superpoderes ocultos e que precisávamos fazer as perguntas certas para extrair grandes funcionalidades dele. Mas, em 2020, acho que é seguro dizer que, com tantas ferramentas prontas para produção utilizando eBPF, estamos descobrindo seus poderes com maestria!

**Cilium e Hubble**

[Cilium](https://cilium.io/) é um popular plug-in CNI do Kubernetes baseado inteiramente em eBPF. Fornece e protege de forma transparente a conectividade de rede e o balanceamento entre as cargas de trabalho de aplicações. Este diagrama em seu [repositório Github](https://github.com/cilium/cilium) apresenta uma idéia sobre como isso é feito.

![](https://lh5.googleusercontent.com/XXMBRvWcsn7zs4hH0Ck_99X1hiB1uWPJ0RbDYWm8zJ9lU7ImxaTfG_hKU5ZK0qqtQSX8vBlo3_aPt7cmfbGequTIz4XzohpJFiV6iY40zznC5Uq6Dpf1pTwxyPS75c7oLhSYND8G)

Hubble é a solução da Isovalent, empresa por trás do Cilium, para observação de rede. Ele expõe mapas de comunicação, políticas de segurança sendo aplicadas/acionadas e provê monitoramento e alertas. Ele também oferece um diagrama simples para mostrar sua arquitetura em seu repositório.

![](https://lh6.googleusercontent.com/AK4HVPZTa45FEgCkcNGWir4eblkyjmbWXTjHa7lkO73_0ISVjXo-ukj0EmTsWVmncIzTEyYMsxpIA5ZXebcT6oumvb_-CnblMZ1LUaVLrAtvhKbHAFEG51980OFDCNObAwBhpa9j)

Este é um exemplo do mapa de serviço do Hubble:

![](https://lh5.googleusercontent.com/roeyFXqoa1VPnfshQ5S4u4j5ufPT9hX8CV-2T3PofZFc2u7RhlhVU963ajJg-iBnMkVRmJvzlgc3K1siE9OgMnr-BU7nXFQKfMvfrrpMGieJx2_WKjUf1akW4Cpz1clQDneKiO-_)

**Prometheus eBPF Exporter**

CloudFlare abriu o código em 2018 e esse exportador é basicamente o BCC como métricas do Prometheus. Então, em vez de ferramentas CLI que exigiriam que você se conectasse (*ssh*) a uma máquina para rastrear algo, você escreveria programas eBPF e os transformaria em métricas do Prometheus para verificar nos gráficos do Grafana.

Exemplos prontamente disponíveis seriam uso de CPU, uso de memória ou uso de CPU Cgroup para verificar as métricas "por serviço" em sua infraestrutura. (A primeira imagem a seguir é uma visão geral do *node_exporter*; a segunda é uma visualização "por serviço" do cAdvisor.)

![](https://lh4.googleusercontent.com/Lbjpc779julGtVwkcaQkyAZJXu1RSIUhe6CXq6N_jhC0tcFYiSOwR9suSGc5_yzvVs3FJ6Sym_IiWYWDurZJF6cWTLl4LMOsuFbIcwORuciJ_Y19Dwhff94dpAgYGZzvFgW2nAfj)

![](https://lh4.googleusercontent.com/-h7l045RmB81BBl-F9jP8ZcJy8SOUmDxDF55D3MY8NVznq2Lq4sUQ677SzNM2hUM3FXIdLL8bTLgIETABI9xymr9CW_QmyxE1N8SCANXcmbdC8s10hpv_6MYyL5xuFxEoySnztGU)


**Falco**

Falco é uma ferramenta de segurança de *runtime*. Tem como objetivo detectar anomalias de atividades em nós e _containers_. Ele monitora brotamento de processos, por exemplo se um contêiner do MongoDB, que deveria ter apenas o processo do MongoDB rodando nele, também tivesse um segundo processo estranho e inesperado em execução; ou até mesmo atividade de rede, tal como se um Nginx abrisse uma porta inesperada em um servidor. Sysdig também fornece um diagrama de arquitetura útil para entendermos melhor a arquitetura do Falco.

![](https://lh5.googleusercontent.com/b5zHsYkdXrvsvj-PM1RQm1kNTcFE9LrJ1BKmY1FMp9xFCJk_UM6fjbW1JWrYdktNKLSCmu0EKhZ7rBW_CZxhDA__dFt99eP24jF6SVlPZdrAk1Ltj9_vBsfIz1bGZHiYZfbWpKFS)

**Algumas outras entidades**

A lista continua com:

* O fork que a Datadog fez do rastreador TCP da Weaveworks (Datadog Performance Monitoring) 
* Flowmill
* Scope Weave
* ntopng
* Inspektor Gadget
* Instana (processos de SO)

Existem algumas outras soluções bem **fechadas** usando eBPF internamente em empresas como Facebook e Netflix, especialmente para firewall de borda XDP, que substitui o _iptables_ por uma série de razões. A maioria dos projetos se concentra em observabilidade e controle de rede, mas alguns tentam instrumentar outros dados, como Prometheus eBPF Exporter, Falco e Instana.

## Razão nº 3: _Flexibilidade e velocidade em ter novas ferramentas de *Kernel tracing*_

Este é provavelmente o motivo mais empolgante para o hype em torno do eBPF. Podemos argumentar que o eBPF é simplesmente o mais recente vencedor do rastreamento de Kernel.

Como Daniel Borkmann, engenheiro da Isovalent e mantenedor do eBPf, disse em sua palestra [eBPF and Kubernetes: Little Helper Minions for Scaling Microservices](https://www.youtube.com/watch?v=99jUcLt3rSk): Linus Torvalds, criador do kernel do Linux, tem uma regra para aceitar e incluir novas ideias malucas no kernel. Cada nova ideia maluca deve ser embrulhada em um pacote de presente que a torne obviamente boa à primeira vista. A ideia é maluca, mas seus argumentos são razoáveis e nada malucos. Esse foi o caso do eBPF, quando eles estavam tentando emplacar o eBPF no Kernel.

Temos `ftrace` e` perf_events` como exemplos de ferramentas de rastreamento que foram integradas ao Kernel, mas esperar que os _merge requests_ sejam aceitos pode ser tedioso e demorar muito. Ter uma maneira de apenas enviar dinamicamente programas para serem anexados aos componentes internos do Kernel acelera imensamente o desenvolvimento das ferramentas de *kernel tracing*, além de reduzir consideravelmente a complexidade do SO ao longo do tempo, já que menos coisas farão parte do código principal.

Outra opção é escrever *Kernel modules* personalizados. Isso pode até ser viável em diversos cenários, entretanto o mundo super-dinâmico que vivemos hoje, às vezes com milhares de _containers_ sendo executados, nascendo e morrendo, com regras de roteamento, políticas de acesso e controle de tráfego pode ser demais para os conhecidos módulos do Kernel. Ter a VM do eBPF projetada para executar programas com segurança, sem congelar ou travar seu sistema operacional, é claramente uma opção melhor.

## Conclusão

Neste post, tentamos mostrar brevemente o que empolga no eBPF e algumas das razões para o hype contínuo no cenário de observabilidade. Acho importante mostrar que já temos ferramentas para usá-lo em produção e que eBPF está maduro o suficiente para isso. E para finalizar temos outra frase do Brendan Gregg:

> "Você não precisa necessariamente saber tudo sobre o eBPF para ficar animado com ele, mas é bom entender os recursos que ele traz para à mesa".

E é extraordinário termos tantas pessoas criando abstrações em torno dele para torná-lo mais fácil de usar!
