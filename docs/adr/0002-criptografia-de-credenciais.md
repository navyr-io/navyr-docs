# 0002 — Criptografia de credenciais com chave derivada por organização

**Status:** Aceita · **Data:** 2026-05

## Contexto

A plataforma guarda segredos de terceiros: chaves de provedor de IA, segredos
de webhook, semente de TOTP e credenciais de cluster. Guardá-los em claro
transforma um dump de banco — ou um backup mal protegido — em comprometimento
total de todos os clientes de uma vez.

O requisito não é só "cifrar". É que o vazamento do material de uma organização
não sirva para as outras.

## Decisão

AES-256-GCM, com **chave derivada por organização**:

```
chave = SHA-256( seed + ":" + orgID )
```

O `seed` vem de `AI_PROVIDER_SECRET_KEY`, com fallback para `JWT_SECRET`.

GCM foi escolhido por ser autenticado: ciphertext adulterado falha na
decifragem em vez de produzir plaintext silenciosamente errado.

Organizações em plano que suporta BYOK podem apontar um KMS externo
(`UpdateOrgKMSConfig`), e aí a chave não é derivada localmente.

## Consequências

**O que ganhamos.** Um dump de banco não entrega segredo legível. Ciphertext de
uma organização não decifra com a chave de outra — o raio do vazamento fica
contido por tenant.

**O que custou, e é dívida reconhecida:**

1. **A derivação é `SHA-256` de concatenação, não HKDF.** HKDF é o primitivo
   correto para derivar chave a partir de material de chave, e é o que
   deveríamos usar. A construção atual funciona, mas não é a que se defende
   numa auditoria sem explicação adicional.

2. **O fallback para `JWT_SECRET` faz um segredo acumular duas funções**:
   assinar token de sessão e derivar chave de cifragem. Vazamento de
   `JWT_SECRET` compromete as duas coisas ao mesmo tempo, e a rotação de um
   obriga a rotação do outro. Definir `AI_PROVIDER_SECRET_KEY` explicitamente
   evita isso — o fallback existe para não quebrar instalação antiga.

3. **Não há rotação de chave.** Trocar o `seed` torna todo o ciphertext
   existente indecifrável. Rotacionar exige decifrar com a chave antiga e
   recifrar com a nova, e esse caminho não está implementado.

## Alternativas descartadas

**Chave única para toda a instalação.** Mais simples, mas um vazamento entrega
todas as organizações. O isolamento por tenant é justamente o que se quer
comprar aqui.

**KMS externo obrigatório.** É mais forte, e continua disponível para quem quer.
Como obrigatório inviabiliza a instalação self-hosted simples e a edição OSS,
que precisam funcionar sem dependência de nuvem.

**Cifrar no nível do banco (TDE).** Protege o disco, não o dump lógico nem o
acesso via aplicação — que são os vetores que preocupam aqui.
