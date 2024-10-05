# CTF Pato Academy - Setembro 2024

Este é o relato da minha experiência com o CTF da Pato Academy. O objetivo deste CTF era encontrar o máximo de falhas possíveis e documentar, não há flags.

## Domínios e Subdomínios

Nos foi dado os seguintes domínios e subdomínios:

- `http://tribopapaxota.org`
- `http://ctf.tribopapaxota.org`
- `http://git.tribopapaxota.org`
- `http://hub.tribopapaxota.org`

*(Nome escolhido carinhosamente pelos moderadores xD)*

Tentei acessá-los, mas não carregaram:

![Primeira Imagem Aqui](https://i.imgur.com/5jLf5V3.png)

Resolvi fazer um scan com o nmap para ver as portas que estavam abertas:

![Segunda Imagem Aqui](./Imagens/2.jpg)

A porta `8888` estava aberta, rodando uma aplicação web nela. Tentei acessar redirecionando para essa porta:

![Terceira Imagem Aqui](./Imagens/3.jpg)

Consegui acessar, porém a página principal não continha nada além de uma animação. O próximo passo foi fazer um scan de diretórios. Eu costumo usar a ferramenta `dirsearch`.

![Quarta Imagem Aqui](#)

Acessei o diretório `/register` e fiz um cadastro. Notei um campo chamado **MFA** com um código. **MFA** é a sigla de "Multi-Factor Authentication", então anotei o código para usar posteriormente no login.

![Quinta Imagem Aqui](./Imagens/5.jpg)

## Tentando Fazer Login

Fui até o diretório `/login`, inseri meu login e senha, porém havia um campo **CODE**, onde provavelmente seria usado o código da página de registro. No entanto, não funcionou. Eu já havia feito um outro CTF da comunidade onde precisei gerar 2FA a partir de um código OTP, então usei um script em Python que eu já tinha no Kali para gerar o código:

![Sexta Imagem Aqui](./Imagens/6.jpg)
![Sétima Imagem Aqui](./Imagens/7.svg)

Consegui logar. A partir daí, tentei várias superfícies de ataque, a maioria sem sucesso, porém consegui identificar duas falhas interessantes.

## Primeira Exploração - Token de Sessão

O token de sessão era um base64 que, ao ser decodificado, era composto por:

`<nome-do-usuário>:<hash-MD5-da-senha>`


Fiz alguns testes modificando apenas o nome do token e codificando novamente para base64. Percebi que o nome de usuário era alterado na mensagem da página principal.

![Oitava Imagem Aqui](#)
![Nona Imagem Aqui](#)

## Segunda Exploração - Upload de Imagem

No diretório `/settings`, percebi um campo de upload de imagem para alterar a imagem de perfil no `/dashboard`. Quando fiz o upload, percebi que a imagem era salva da seguinte forma:

`<nome-do-usuário>.<extensão-da-imagem>`


no caminho `/uploads`. O servidor pegava o nome de usuário pelo token de sessão, então criei outro usuário chamado "wild2" para tentar trocar a imagem dentro do perfil do usuário "wild". Funcionou!

![Décima Imagem Aqui](#)

Com o mesmo **IDOR**, foi possível trocar a senha do usuário "wild2" através das configurações do usuário "wild".

De bônus, percebi que a imagem padrão para um usuário recém-criado estava no caminho `/uploads/patonymous.jpg`. Alterei o nome de usuário no token para "patonymous" e mudei a imagem padrão. Agora, todos os novos usuários criados terão uma nova imagem padrão!

## Explorando o Diretório `/chat`

Explorando o diretório `/chat`, vi que se tratava de um chat com uma IA que falava diversas baboseiras. Notei que outros usuários mandavam mensagens ao mesmo tempo que eu, e as mensagens eram renderizadas no navegador. Resolvi testar um **XSS** no chat e ele aceitou a tag:

```html
<h1>TESTANDO...</h1>
```
![Décima Primeira Imagem Aqui](#)
