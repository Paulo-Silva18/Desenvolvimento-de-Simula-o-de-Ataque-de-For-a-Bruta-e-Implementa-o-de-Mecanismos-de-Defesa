# Prova 04 — Simulação de Ataque de Força Bruta e Implementação de Mecanismos de Defesa

**Disciplina:** Segurança da Informação — IFTM Campus Patrocínio
**Aluno:** Paulo Henrique Marques Silva

## 1. Objetivo

Demonstrar, em ambiente estritamente isolado e controlado, a fragilidade de credenciais fracas frente a ataques automatizados de força bruta/dicionário, bem como a eficácia de mecanismos de defesa como bloqueio temporário por limite de tentativas (rate limiting) e autenticação multifator (2FA/TOTP).

## 2. Topologia do laboratório

| Máquina | Papel | SO | Rede |
|---|---|---|---|
| Kali Linux 2026.2 | Atacante | Kali Linux | Host-Only (VirtualBox) |
| Windows11 | Alvo (aplicação web vulnerável) | Windows 11 | Host-Only (VirtualBox) |

- As duas VMs se comunicam exclusivamente pela rede **Host-Only** do VirtualBox (`VirtualBox Host-Only Ethernet Adapter`), sem acesso à internet durante os testes de ataque — ambiente isolado conforme exigido.
- IP do alvo (Windows11) na rede Host-Only: `192.168.56.103`
- Serviço atacado: aplicação web Flask (porta `5000`), formulário de login (`POST /login`)

## 3. Aplicação alvo

Aplicação Flask desenvolvida especificamente para o laboratório, com três versões incrementais:

1. **v1 — Vulnerável:** login simples sem nenhuma proteção. Credencial de teste: `admin` / `123456` (senha fraca, presente em listas públicas de senhas comuns).
2. **v2 — Rate limiting:** adiciona `Flask-Limiter`, bloqueando o IP de origem após 3 tentativas de login por minuto (HTTP 429).
3. **v3 — 2FA (TOTP):** adiciona autenticação de dois fatores via `pyotp`, exigindo um código de 6 dígitos gerado pelo Google Authenticator além da senha correta.

Código-fonte completo das três versões está na pasta `/app/`.

## 4. Fase ofensiva — Ataque de força bruta

### 4.1 Ferramenta e wordlist

- Ferramenta: **Hydra v9.7**, módulo `http-post-form`
- Wordlist de teste: `wordlist.txt`, contendo a senha correta (`123456`) entre outras candidatas comuns

### 4.2 Comando executado

```bash
hydra -l admin -P wordlist.txt -s 5000 192.168.56.103 http-post-form "/login:username=^USER^&password=^PASS^:S=sucedido"
```

Observações sobre o comando:
- `-l admin`: fixa o usuário alvo
- `-P wordlist.txt`: testa cada senha da lista
- `-s 5000`: especifica a porta do serviço (Flask não roda na porta padrão 80)
- `S=sucedido`: informa ao Hydra o texto que aparece na resposta HTTP quando o login é bem-sucedido ("Login bem-sucedido!"), usado como critério de sucesso

### 4.3 Resultado

O Hydra identificou a credencial correta:

```
[5000][http-post-form] host: 192.168.56.103   login: admin   password: 123456
1 of 1 target successfully completed, 1 valid password found
```

<img width="1280" height="800" alt="ataque" src="https://github.com/user-attachments/assets/83449835-4581-4149-9fe0-21104abec2a2" />
<img width="1280" height="800" alt="kali" src="https://github.com/user-attachments/assets/f1886bba-5fc8-42c1-a7fe-d5bc93b1a5dd" />
<img width="1280" height="800" alt="kali2" src="https://github.com/user-attachments/assets/b1c784d5-fd87-4487-8158-4a8f9be43476" />

<img width="1024" height="768" alt="seguranca" src="https://github.com/user-attachments/assets/a563e839-8da3-409d-8616-7d9d439fcb05" />
<img width="1024" height="768" alt="seguranca2" src="https://github.com/user-attachments/assets/5081fd84-0850-4d8c-aefd-54d50ce7af0a" />
<img width="1024" height="768" alt="seguranca3" src="https://github.com/user-attachments/assets/f97e2d35-a78f-4ea6-9090-9bea2c9f4051" />

### 4.4 Validação do acesso obtido

O login foi validado manualmente no navegador com a credencial descoberta (`admin` / `123456`), confirmando o acesso não autorizado ao sistema desprotegido.

## 5. Fase defensiva

### 5.1 Bloqueio temporário por limite de tentativas (rate limiting)

**Implementação:** biblioteca `Flask-Limiter`, limitando a rota `/login` a 3 requisições por minuto por IP de origem. Ao exceder o limite, o servidor responde com HTTP 429 em vez de processar a tentativa de login.

```python
@app.route("/login", methods=["POST"])
@limiter.limit("3 per minute")
def login():
    ...
```

**Novo ataque após a implementação:**

```bash
hydra -l admin -P wordlist2.txt -s 5000 192.168.56.103 http-post-form "/login:username=^USER^&password=^PASS^:S=sucedido"
```

Resultado: todas as tentativas do ataque subsequente (realizado dentro da mesma janela de 1 minuto da tentativa anterior) foram bloqueadas, incluindo a senha correta — `0 valid password found`.

Log de evidência (`seguranca.log`):
```
... LOGIN FALHOU - usuario=admin ip=192.168.56.103
... LOGIN OK - usuario=admin ip=192.168.56.103
... BLOQUEADO POR LIMITE DE TENTATIVAS - ip=192.168.56.103
... BLOQUEADO POR LIMITE DE TENTATIVAS - ip=192.168.56.103
```

*[Inserir print do terminal do Kali mostrando os erros HTTP 429]*
*[Inserir print do conteúdo de seguranca.log]*

### 5.2 Autenticação de dois fatores (2FA / TOTP)

**Implementação:** biblioteca `pyotp`, gerando um segredo TOTP associado ao usuário e exigindo um código de 6 dígitos, válido por 30 segundos, além da senha correta. O segredo foi configurado no Google Authenticator via QR Code gerado pela própria aplicação (rota auxiliar `/qrcode`).

```python
otp_ok = totp.verify(otp) if otp else False
if senha_ok and otp_ok:
    # acesso liberado
```

**Teste manual com senha correta e código 2FA inválido:**

Resultado: "Senha correta, mas código 2FA inválido" — acesso negado mesmo com a credencial correta.

<img width="1501" height="1079" alt="image" src="https://github.com/user-attachments/assets/d98b47ca-1758-40d6-97d6-7f41f6deb87a" />

**Teste manual com senha correta e código 2FA correto:**

Resultado: "Login bem-sucedido!" — acesso concedido apenas com os dois fatores corretos.

<img width="1024" height="768" alt="login com cod" src="https://github.com/user-attachments/assets/58f7f5da-1a5f-44bc-b7df-16554f1e681c" />

**Novo ataque automatizado após a implementação do 2FA:**

```bash
hydra -l admin -P wordlist.txt -s 5000 192.168.56.103 http-post-form "/login:username=^USER^&password=^PASS^:S=sucedido"
```

Resultado: `0 valid password found`. Mesmo o Hydra conhecendo a senha correta, o ataque automatizado não é capaz de gerar o código TOTP necessário, sendo totalmente inviabilizado.

<img width="1918" height="956" alt="image" src="https://github.com/user-attachments/assets/27ab02a4-0de6-4f64-a5e3-e317b68ac2f9" />

## 6. Conclusão

O laboratório demonstrou de forma prática que:

- Senhas fracas, sem nenhum mecanismo de proteção, podem ser quebradas rapidamente por ferramentas automatizadas de força bruta/dicionário (Hydra).
- A implementação de bloqueio temporário por limite de tentativas (rate limiting) já reduz drasticamente a viabilidade de um ataque de força bruta tradicional, barrando a maior parte das tentativas em sequência.
- A adição de um segundo fator de autenticação (2FA/TOTP) inviabiliza por completo o ataque automatizado, já que o código TOTP não pode ser previsto ou obtido por dicionário — mesmo que a senha seja descoberta, o acesso permanece bloqueado.
- A combinação de ambas as defesas representa uma estratégia robusta e de baixo custo de implementação para proteger serviços de autenticação expostos.

## 7. Vídeo demonstrativo

Link (YouTube "Não listado"): https://youtu.be/nEup6g0qPiU
