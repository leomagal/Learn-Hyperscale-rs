# Guia de Aprendizado Detalhado - Hyperscale-rs (v2)

## Módulo 2: Consenso BFT e Segurança

### 2.3 Bloqueio de Voto e Desbloqueio HotStuff-2 (CORREÇÃO CRÍTICA)

**O que é Bloqueio de Voto?**

O bloqueio de voto é um mecanismo de segurança que impede um validador de votar em blocos conflitantes na mesma altura **na mesma rodada**.

**Como funciona**:
1. Quando um validador vota em um bloco na altura H, rodada R, ele cria um bloqueio de voto: `voted_heights[H] = (block_hash, R)`
2. Se outro bloco chegar na altura H na mesma rodada R, o validador verifica seu bloqueio de voto e **não pode votar novamente**.

**O Erro Crítico de Compreensão (v1)**:

Na v1 deste guia, afirmamos incorretamente que os bloqueios de voto persistem através das rodadas. Isso **não é verdade**.

**O Comportamento Correto (v2)**:

Os bloqueios de voto são **limpos** quando uma rodada avança sem a formação de um QC na altura atual. Este é o **mecanismo de desbloqueio HotStuff-2**.

**Mecanismo de Desbloqueio HotStuff-2**:

Quando uma rodada avança (por exemplo, de R para R+1):
1. O sistema verifica se um QC foi formado na altura atual H.
2. **Se NENHUM QC foi formado**: O bloqueio de voto para a altura H é **limpo**.
3. **Se SIM, um QC foi formado**: O bloqueio de voto é **mantido**.

**Por que isso é seguro?**
- Se nenhum QC foi formado, o bloco anterior não alcançou consenso.
- É seguro votar em um bloco diferente na nova rodada.
- Esta é uma característica chave do HotStuff-2 que garante **liveness** (o sistema não fica travado).

**Referência de Código**:
- `state.rs`, linha 4331: `self.voted_heights.remove(&height)` - É aqui que o bloqueio de voto é limpo.

**Segurança vs Liveness**:
- **Segurança**: O bloqueio de voto previne a equivocação dentro de uma rodada.
- **Liveness**: A limpeza dos bloqueios de voto no avanço da rodada evita que o sistema fique travado.

### 2.4 Avanço de Rodada e Limpeza de Bloqueio de Voto (CORREÇÃO CRÍTICA)

**O que é Avanço de Rodada?**

O avanço de rodada é o processo de mover para a próxima rodada na mesma altura quando um bloco não consegue alcançar consenso.

**Como funciona**:
1. Ocorre um timeout (por exemplo, `on_proposal_timer`).
2. O número da rodada é incrementado (por exemplo, `self.view += 1`).
3. O **mecanismo de desbloqueio HotStuff-2** é acionado.

**Limpeza de Bloqueio de Voto durante o Avanço de Rodada**:

Quando `advance_round()` é chamado:
1. Ele verifica se um QC foi formado na altura atual.
2. Se nenhum QC foi formado, ele **limpa o bloqueio de voto** para essa altura.
3. Isso permite que o validador vote em um novo bloco na nova rodada.

**Referência de Código**:
- `state.rs`, linha 4304: `advance_round()` - A função principal para o avanço de rodada.
- `state.rs`, linha 4331: `self.voted_heights.remove(&height)` - A limpeza do bloqueio de voto.

**O que acontece a seguir?**
- **Se o bloqueio de voto for limpo**: O novo líder pode propor um novo bloco, e os validadores podem votar nele.
- **Se o bloqueio de voto for mantido**: O novo líder deve repropor o mesmo bloco ao qual o validador está bloqueado.

### 2.5 Sincronização de Visão e Bloqueios de Voto (Atualizado)

**O que é Sincronização de Visão?**

A sincronização de visão é como os validadores se mantêm em sincronia com o número da rodada atual.

**Como funciona**:
- Quando um validador recebe um QC com um número de rodada maior, ele atualiza seu número de rodada local (`self.view`).

**Bloqueios de Voto durante a Sincronização de Visão**:
- Quando um validador sincroniza para uma nova visão, ele **não limpa automaticamente seus bloqueios de voto**.
- Os bloqueios de voto são limpos apenas durante o **avanço de rodada** (quando ocorre um timeout).

**Ponto Chave**:
- A sincronização de visão e a limpeza de bloqueio de voto são **mecanismos separados**.
- A sincronização de visão mantém os validadores na mesma rodada.
- O avanço de rodada (com limpeza de bloqueio de voto) permite o progresso quando uma rodada falha.
