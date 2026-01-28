+++
title = "Investment House - Write-up"
date = 2025-01-22
draft = false
description = " "
tags = ["CTF", "Hacking", "HackingClub"]
+++

### Investment House - Write-up
===========================


![](/img/investment-house/image-48.png)

## INTRODUCTION
------------

Este será o primeiro write-up que estarei publicando de uma série que estão por vir, envolvendo máquinas retiradas tanto do Hacking Club quanto do Hack the Box. Estarei demonstrando algumas abordagens de como cheguei em determinadas conclusões em certas partes do desafio e dialogando alguns conceitos sobre as tecnicas utilizadas.

Investment House foi uma das máquinas mais interessantes de explorar porque ela aborda conceitos bastante realístico e intrigantes que forçam você a aprender e buscar por coisas novas (um dos principais benefícios do CTF). Sem mais delongas, vamos para o campo de batalha. XD

## RECON PHASE
-----------

Como de praxe começaremos com um mapeamento das portas abertas para identificar os possiveis vetores de ataques:

![](/img/investment-house/image-3.png)

Como pode-se notar no output do nmap temos apenas duas portas abertas. Porém a que mais nos interessa no momento será a porta 80, que está rodando um servidor web e que trás muitas possibilidades de exploração. Além disso também temos o hostname que iremos adicionar no nosso `/etc/hosts` para facilitar o acesso da aplicação.

Antes de mergulhar de cabeça na aplicação é sempre interessante verificar se não há outros possíveis subdominios rodando. Usando o tio gobuster logo foi identificado dois novos subs, `api` e `secrets`:

![](/img/investment-house/image-8.png)

Aqui as possibilidades já aumentaram e com isso precisamos analisar o que cada uma dessas aplicações nos reserva de interessante. Como toda mente curiosa é claro que comecei olhando logo o sub `secrets`:

![](/img/investment-house/image-9.png)

A principio não possui nada de interessante além de uma pagina de login. Não tem muita coisa a se fazer aqui. Porém como todo processo de recon, é claro que será feito aquele fuzzing nervoso, e pra isso usarei o tio ffuf:

![](/img/investment-house/image-10.png)

![](/img/investment-house/image-11.png)

Comecei analisando o `/api` e logo de cara nos deparamos com esse misconfig onde conseguimos listar/ler dois arquivos interessantes:

![](/img/investment-house/image-12.png)

![](/img/investment-house/image-13.png)

Aqui as coisas começam a ficar mais interessante ainda... no `list.php` conseguimos identificar um serviço de api (que encontramos no começo do recon com o gobuster) que fornece informações de Cryptos, e o `rotate.php` que parece ser um endpoint interessante a analisar.

![](/img/investment-house/image-14.png)

Fazendo o POST para esse endpoint ele nos retorna um response muito interessante nos dando a opção de gerar um novo secret caso o tenha perdido. E como não possuimos nenhum token, é claro que iremos tentar gerar.

![](/img/investment-house/image-15.png)

Depois ter gerado o token agora precisamos descobrir onde ele será usado. Pela lógica, com certeza deverá ter algum endpoint na api onde não possuimos permissão de acesso, por isso o tio ffuf já entrou em ação novamente nos dando o ouro:

![](/img/investment-house/image-17.png)

![](/img/investment-house/image-18.png)

Fazendo a requisição para esse endpoint perceba que precisamos do token para acessa-lo.

![](/img/investment-house/image-20.png)

O interessante de brincar com algumas api's é que na maioria dos casos ela sempre mostrará no corpo da resposta o que está faltando para a requisição funcionar como esperado. E como todo paramêtro suspeito, é claro que irá rolar uns testes básicos por vulnerabilidades. Como mostrado na imagem abaixo, foi identificado um LFI, que será uma de nossas portas de entrada para as proximas etapas da exploração. XD

![](/img/investment-house/image-21.png)

## EXPLOITATION PHASE
------------------

Aqui as possibilidades aumentaram bastante porque com um LFI em mãos podemos fazer vários testes para chegar a um possivel RCE e conseguir a shell na máquina. Antes de tentar qualquer testes a cegas, precisamos entender como as aplicações estão estruturadas dentro do servidor para ampliar nossos vetores de ataque. Além disso é interessante entender o objetivo de cada aplicação para não cair nos famosos buracos de coelhos, que são muito comums em máquinas hard e insanas. Como já sabemos, o objetivo da aplicação secrets era conseguir ter acesso ao token para usar na api e poder ter acesso a esse LFI cabuloso. E a aplicação principal que está rodando na porta 80, o que tem de interessante?

![](/img/investment-house/image-22.png)

![](/img/investment-house/image-23.png)

Acessando a aplicação web logo nos deparamos com uma web page bem simples que contém uma pagina de login e um formulário de registro.

![](/img/investment-house/image-6.png)

![](/img/investment-house/image-7.png)

Após a criação e autenticação do nosso usuário nos deparamos com uma pagina muito interessante contendo um upload de imagem e uma pagina de download de portfolio. Até aqui já podemos notar vários possiveis vetores de testes, e para nossa alegria temos o nosso LFI que possibilitará ler e entender como essa aplicação funciona fazendo aquele code review nervoso. XD

![](/img/investment-house/image-28.png)

Comecei o teste analisando o `download.php` porque parecia um endpoint interessante devido suas funcionalidades e parâmetros. Analisando seu código já notamos uma vulnerabilidade nesse bloco onde ele confia cegamente no caminho passado pelo usuário desde que ele comece com o `phar://`. O wapper `phar://` no PHP é usado para acessar arquivos dentro de um arquivo `phar://` (PHP archive). No entanto, muitas funções de sistema de arquivos (como,`is_file`,`filesize`,`readfile`, etc.) ao interagirem com um arquivo PHAR, desserializam automaticamente os metadados contidos nele. Ou seja, se conseguirmos uma forma de controlar o arquivo que será processado como um PHAR, podemos forçar a desserialização de um objeto malicioso, conseguindo triggar nosso sonhado RCE. Agora vêm os poréms, porque pra explorar essa gracinha precisamos descobrir algum gadget chain para poder fazer o trem funcionar. Depois de olhar até a alma da aplicação encontrei no `register.php` algo interessante, a classe que estavamos tanto buscando pra fazer nosso exploit funcionar:

![](/img/investment-house/image-29.png)

![](/img/investment-house/image-31.png)

Dando uma fuçada nesse código já da pra perceber que podemos controlar essas três propriedades `$path`, `$file`, `$content` na hora de criar nosso objeto serializado, podendo assim escrever arquivos no servidor. O método mágico `__destruct` é executado automaticamente pelo PHP quando o objeto é destruído (geralmente no final da execução do script). Ele não precisa ser chamado explicitamente. O rolê principal está dentro do `__destruct`, a função `file_put_contents()` pega três argumentos:

1.  O caminho completo do arquivo (`$this->path . $this->file`)
2.  O conteúdo a ser escrito (`$this->content`)

Como a gente tem controle sobre as três propriedades, a gente pode chegar no tio servidor e dizer: "Ei tio, escreva ESTE conteúdo, NESTE aquivo, NESTE diretório". XD

Com toda essa viajem teórica, na prática nosso exploit ficou mais ou menos assim:

![](/img/investment-house/image-36.png)

Depois de ter gerado nosso payload o proximo passo será encontrar uma forma de upar ele no servidor para acessar com o `phar://`. E, como já sabemos, nós temos essa funcionalidade no profile do usuário. Primeiro temos que renomear para `.jpg` para o upload funcionar como esperado, o phar não liga pra extensão, o papo dele é ter os objetos serializados dentro do metadados.

![](/img/investment-house/image-37.png)

![](/img/investment-house/image-38.png)

Depois de ter triggado nosso arquivo phar conseguimos com sucesso acessar nosso `mocoto.php`. A partir daqui não tem mistério, é só pegar a reverse shell e começar o processo de privilege escalation.

## POST-EXPLOITATION PHASE
-----------------------

![](/img/investment-house/image-39.png)

![](/img/investment-house/image-40.png)

![](/img/investment-house/image-42.png)

Começamos buscando por binários que possuem as permissões de SUID no sistema, e um que chamou bastante a atenção foi esse da imagem apontado, que parece ser uma tool de gerenciamento de banco de dados via CLI. Ao executar ele percebemos algumas funcionalidades interessantes:

![](/img/investment-house/image-43.png)

Depois de alguns testes percebemos que ele possui uma vulnerabilidades de SQLi após colocar um input incomum no Seach user. Além disso a mensagem "unrecognized token" indica que estamos lidando com um SQLite. Após alguns testes percebemos que a função `load_extension` está habilitado no SQLite, nos direcionando para a proxima fase da exploração que será criar uma biblioteca compartilhada maliciosa (.so) e tentar pegar uma revshell no root.

![](/img/investment-house/image-44.png)

![](/img/investment-house/image-46.png)

Acima mostro o código que utilizei para criar a biblioteca maliciosa e o comando para gera-la. Abaixo mostro o payload utilizado para chamar a biblioteca que foi gerada no `/tmp` e por fim nós temos a shell como root. XD

![](/img/investment-house/image-47.png)


Topic [CTF](https://lizardsec.xyz/tag/ctf/) [Hacking](https://lizardsec.xyz/tag/hacking/) [HackingClub](https://lizardsec.xyz/tag/hackingclub/)

