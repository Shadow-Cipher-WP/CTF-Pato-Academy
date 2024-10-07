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

![Segunda Imagem Aqui](https://i.imgur.com/iK3K9Fo.png)

A porta `8888` estava aberta, rodando uma aplicação web nela. Tentei acessar redirecionando para essa porta:

![Terceira Imagem Aqui](https://i.imgur.com/5M76gUs.png)

Consegui acessar, porém a página principal não continha nada além de uma animação. O próximo passo foi fazer um scan de diretórios. Eu costumo usar a ferramenta `dirsearch`.

![Quarta Imagem Aqui](https://i.imgur.com/wotU3ox.png)

Acessei o diretório `/register` e fiz um cadastro. Notei um campo chamado **MFA** com um código. **MFA** é a sigla de "Multi-Factor Authentication", então anotei o código para usar posteriormente no login.

![Quinta Imagem Aqui](https://i.imgur.com/wDD48JB.png)

## Tentando Fazer Login

Fui até o diretório `/login`, inseri meu login e senha, porém havia um campo **CODE**, onde provavelmente seria usado o código da página de registro. No entanto, não funcionou. Eu já havia feito um outro CTF da comunidade onde precisei gerar 2FA a partir de um código OTP, então usei um script em Python que eu já tinha no Kali para gerar o código:

![Sexta Imagem Aqui](https://i.imgur.com/gt63YoX.png)
![Sétima Imagem Aqui](https://i.imgur.com/WisfjQk.png)

Consegui logar. A partir daí, tentei várias superfícies de ataque, a maioria sem sucesso, porém consegui identificar algumas falhas interessantes.

## Explorando IDOR no Token de Sessão

O token de sessão era um base64 que, ao ser decodificado, era composto por:

`<nome-do-usuário>:<hash-MD5-da-senha>`

Fiz alguns testes modificando apenas o `<nome-do-usuário>` do token e codificando novamente para base64. Percebi que o nome de usuário era alterado na mensagem da página principal.

![Oitava Imagem Aqui](https://i.imgur.com/JyFtv8S.png)
![Nona Imagem Aqui](https://i.imgur.com/Nl5LmqF.png)

Aqui temos um **IDOR**, pois como o servidor resgata as informações do banco de dados através do nome de usuário no token, se alterarmos o nome para o nome de outro usuário, o servidor nos devolve as informações dele, que nesse sistema é apenas a imagem de perfil, mas poderia ser mais coisas.

## Explorando IDOR no Upload de Imagem

No diretório `/settings`, percebi um campo de upload de imagem para alterar a imagem de perfil no `/dashboard`. Quando fiz o upload, percebi que a imagem era renomeada e salva da seguinte forma:

`<nome-do-usuário>.<extensão-da-imagem>`

No caminho `/uploads`. O servidor pegava o nome de usuário pelo token de sessão, então criei outro usuário chamado `wild2` e tentei trocar a imagem de perfil usando o usuário `wild` apenas alterando o nome do cookie para `wild2`. Funcionou!

![Décima Imagem Aqui](https://i.imgur.com/bDSIokE.png)

De bônus, percebi que a imagem padrão para um usuário recém-criado estava no caminho `/uploads/patonymous.jpg`. Alterei o nome de usuário no token para `patonymous` e consegui alterar a imagem padrão. Agora, todos os novos usuários criados terão uma nova imagem padrão da minha escolha xD!

## Explorando IDOR na Troca de Senha

Com o mesmo **IDOR** que permite trocar a imagem de perfil de um usuário, foi possível também trocar a senha do usuário `wild2` através das configurações do usuário `wild`.

Loguei com o usuário `wild`, alterei o token de sessão para `<wild2>:<hash-MD5-da-senha>`(lembrando que não alterei o hash, apenas o nome) e codifiquei novamente para base64. Fui em `/settings` e troquei a senha. Desloguei e loguei com o usuário `wild2` com a nova senha e obtive sucesso!

## Explorando Stored-XSS no diretório `/chat`

Explorando o diretório `/chat`, vi que se tratava de um chat com uma IA que falava diversas baboseiras. Notei que outros usuários mandavam mensagens ao mesmo tempo que eu, e as mensagens eram renderizadas no navegador. Resolvi testar um **XSS** no chat e ele aceitou a tag:

```html
<h1>TESTANDO...</h1>
```
![Décima Primeira Imagem Aqui](https://i.imgur.com/yt88tTn.png)

Não foi possível criar um XSS que roube os cookie de sessão pois eles foram setados com `HttpOnly`, impossibilitando que sejam lidos por javascript, porém nada impede de tentar um payload que faça o usuário que visitar `/chat` executar ações em endpoints da própria aplicação, como troca de senha por exemplo. Desenvolvi o seguinte payload que troca a senha do usuário para `987654321` e enviei no chat logado com o usuário `wild2`:

```html
<img src="invalid.jpg" onerror="fetch('/settings',{method:'POST',headers:{'Content-Type':'application/x-www-form-urlencoded'},body:new URLSearchParams({new_password:'987654321',otp_secret:'',serialized_data:{}})});">
```

Loguei com o usuário `wild`, acessei o chat, desloguei e tentei logar novamente com a senha antiga que era `12345`. Obtive o seguinte erro `Error during 2FA check, Please try again.`, que é o erro que dá quando o usuário erra o login e senha pois a aplicação primeiro verifica se ele contém 2FA.

![Décima Segunda Imagem Aqui](https://i.imgur.com/WOecLJW.png)

depois tentei logar com a senha do payload que é `987654321` e fui solicitado o 2FA, ou seja, a senha foi alterada com sucesso.

![Décima Terceira Imagem Aqui](https://i.imgur.com/osGa2CE.png)


## Explorando Subdomain Takeover nos Subdomains git e hub - 2 falhas!

Tentando acessar os subdomínios `git.tribopapaxota.org` e `hub.tribopapaxota.org` no navegador não obtive sucesso. Fiz scan com nmap e percebi que eles estavam respondendo para o mesmo endereço ip que o domínio principal e com as mesmas portas abertas. Resolvi verificar o subdomínio com o comando dig para ver se recebia mais informações e obtive o seguinte:

![Décima Quarta Imagem Aqui](https://i.imgur.com/IN5WLSo.png)
![Décima Quinta Imagem Aqui](https://i.imgur.com/DvthU5F.png)

Dois redirecionamentos para domínios do github.io. Fui verificar se os repositórios existem:

![Décima Sexta Imagem Aqui](https://i.imgur.com/xeA4XKT.png)
![Décima Sétima Imagem Aqui](https://i.imgur.com/RF9DTCI.png)

Os dois estão disponíveis para criação. Temos duas falhas de subdomain takeover.

Nesse CTF só consegui chegar até aqui, porém eu aprendi muitas coisas novas e pesquisei bastante. Ansioso para os próximos!
