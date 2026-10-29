# ADR — Fronteira PostgreSQL/Redis no MVP

- **Status:** Proposto para revisão
- **Data:** 2026-10-28
- **Escopo:** MVP do Palácio Mental
- **Issue:** [#88 — ADR e fronteira PostgreSQL/Redis](https://github.com/DevTroli/palaciomental/issues/88)

## Contexto

O Palácio Mental usa PostgreSQL e Redis com responsabilidades diferentes. O PostgreSQL possui transações, relacionamentos, constraints e consultas necessárias para os dados do domínio. O Redis oferece baixa latência para leituras repetidas, sessões, contadores e projeções transitórias.

O risco principal é transformar o Redis em uma segunda fonte de verdade. Se isso acontecer, uma expiração, perda de dados ou indisponibilidade do Redis pode causar divergência de permissões, perda de conteúdo ou decisões incorretas de negócio.

## Decisão

O **PostgreSQL é a fonte oficial dos dados do domínio**. O Redis é uma camada complementar e reconstruível, usada somente para:

- cache de leituras e contexto institucional;
- aceleração de sessões com `cached_db`, mantendo a sessão persistida no PostgreSQL;
- projeções derivadas, como rankings, badges e contadores;
- eventos transitórios, Pub/Sub e coordenação de tarefas quando uma issue específica definir esse uso.

O Redis não decide autorização, visibilidade ou propriedade de um recurso. Uma resposta encontrada no cache nunca substitui a verificação de permissões no PostgreSQL e no código de domínio.

## Fronteira de responsabilidade

### PostgreSQL: fonte de verdade

Continuam exclusivamente no PostgreSQL:

- usuários e perfis;
- organizações, memberships e permissões;
- projetos e seus estados;
- comentários, respostas e histórico;
- votos;
- marcos;
- assinaturas;
- configurações cuja perda alteraria o domínio.

Toda criação, atualização ou exclusão desses dados deve ocorrer primeiro no PostgreSQL, dentro das transações necessárias. O código pode invalidar ou atualizar uma projeção Redis depois da confirmação da transação.

### Redis: projeção e aceleração

Podem usar Redis, com TTL e fallback seguro:

- resolução de hostname para organização;
- branding e configuração pública por organização;
- cache de consultas públicas já filtradas;
- sessões por meio de `cached_db`;
- rankings e contadores derivados;
- dados temporários e eventos que possam ser repetidos ou perdidos sem corromper o domínio.

Toda chave deve ter escopo explícito. Exemplos:

```text
tenant:hostname:<hostname>
tenant:<organizacao_id>:branding
tenant:<organizacao_id>:public-config
ranking:<escopo>:<periodo>
```

O prefixo global configurado pelo Django continua sendo aplicado para separar ambientes, por exemplo `pm-dev` e `pm-prod`.

## Regras de leitura e escrita

1. **Leitura com cache hit:** o Redis pode devolver uma projeção dentro do TTL, desde que a operação não use esse valor para conceder autorização.
2. **Cache miss:** o código consulta o PostgreSQL, aplica as regras de visibilidade e grava apenas o resultado derivado no Redis.
3. **Erro ou indisponibilidade do Redis:** a operação tenta continuar pelo PostgreSQL quando isso for seguro. A falha deve gerar log ou métrica, não erro 500 generalizado.
4. **Alteração de domínio:** depois de uma transação confirmada, as chaves relacionadas devem ser invalidadas. A próxima leitura reconstrói a projeção a partir do PostgreSQL.
5. **Dados privados:** não entram em cache público nem em vitrines públicas. O filtro de visibilidade deve ocorrer antes da gravação no Redis.
6. **Autorização:** deve ser recalculada a partir do usuário, da organização, do membership e das permissões oficiais. Redis não é uma lista de permissões.

## Sessões

As sessões usam `django.contrib.sessions.backends.cached_db`. O registro persistido permanece no banco relacional, enquanto o Redis acelera as leituras. Assim, uma indisponibilidade do Redis não deve desconectar todos os usuários nem eliminar os registros de sessão.

## Ranking e contadores

Rankings, badges, totais e posições são projeções derivadas. Podem ser armazenados em Sorted Sets ou outras estruturas Redis quando houver uma implementação específica.

A origem dos pontos e dos eventos continua no PostgreSQL. A projeção deve poder ser apagada e reconstruída por um comando ou rotina idempotente. Se o ranking estiver ausente ou desatualizado, o sistema deve sinalizar degradação ou reconstruí-lo; não deve inventar dados persistentes nem bloquear autorização.

Exemplo conceitual:

```text
PostgreSQL: votos e atividade registrados
Redis: ranking:relevancia:mvp como Sorted Set
Reconstrução: reler votos/atividade e recriar o Sorted Set
```

## Falhas e observabilidade

As integrações Redis devem registrar pelo menos:

- erro de leitura;
- erro de escrita;
- erro de invalidação;
- fallback para PostgreSQL;
- reconstrução de projeção.

Os logs não devem conter senhas, tokens de sessão ou conteúdo privado. A aplicação deve continuar preservando dados e decisões de autorização mesmo quando o Redis estiver indisponível.

## Alternativas consideradas

### Usar somente PostgreSQL

É a opção mais simples e continua válida como fallback, mas não atende por si só aos casos de baixa latência, sessões aceleradas e projeções transitórias previstos para o MVP.

### Usar Redis como banco principal

Rejeitado. Redis não deve substituir as transações, relacionamentos, constraints e histórico oficial do PostgreSQL.

### Introduzir MongoDB neste MVP

Rejeitado neste momento. Não há caso de uso que justifique uma terceira fonte de persistência. Uma decisão futura deve ter issue própria, critérios de migração e estratégia de consistência.

## Consequências

- O sistema mantém uma única fonte oficial de verdade.
- A indisponibilidade do Redis degrada desempenho, mas não deve causar perda de dados ou concessão indevida de acesso.
- Toda nova chave Redis precisa declarar escopo, TTL, origem dos dados e estratégia de invalidação.
- Rankings e contadores exigem rotina de reconstrução e testes de divergência.
- A implementação de multi-tenancy pode evoluir sem depender de uma decisão de persistência concorrente.

## Checklist para novas integrações Redis

Antes de adicionar uma nova chave ou fluxo, responder:

- Qual é a fonte oficial no PostgreSQL?
- O valor é derivado e reconstruível?
- Qual é o escopo da chave?
- Qual é o TTL?
- O que acontece em cache miss?
- O que acontece se Redis estiver indisponível?
- Como a chave é invalidada?
- O valor pode conter dados privados?
- O fluxo usa Redis para autorização? Se sim, ele deve ser redesenhado.
