## Tarefa 4 — Transparência de Relocação (Reflexão)

### 1. Diferença entre migração e relocação

A migração (vista na Tarefa 3) acontece quando o serviço é movido de um servidor para outro **entre requisições**, ou seja, não há uma operação ativa no momento da troca. Já a relocação é mais complexa porque o recurso é movido **enquanto ainda está sendo utilizado**, como no caso de uma conexão WebSocket ativa. Nesse cenário, é necessário manter a continuidade da comunicação durante a transição. 

---

### 2. O buffer garante semântica exactly-once? O que poderia causar entrega duplicada ou perda de mensagem mesmo com o buffer?

Não. O `_message_buffer` ajuda a evitar perda imediata de mensagens durante o estado `MIGRATING`, mas não garante semântica *exactly-once*. Ele apenas armazena temporariamente as mensagens até que a reconexão seja concluída. Pode ocorrer **duplicação** se uma mensagem for enviada ao servidor pouco antes da falha e o cliente não souber se ela foi processada.

---

### 3. A mudanca de estado `MIGRATING -> RECONNECTING -> CONNECTED` e uma maquina de estados. Por que modelar estados explicitamente em vez de uma flag booleana is_relocating?

Modelar explicitamente os estados (`MIGRATING`, `RECONNECTING`, `CONNECTED`) torna o fluxo de transição mais claro e controlado. Uma simples flag booleana como `is_relocating` não diferencia as fases distintas do processo de relocação.

---

### 4. Exemplo real de transparência de relocação

Um exemplo real é o **Kubernetes Pod rescheduling**. Quando um nó falha, o Kubernetes pode recriar o Pod em outro nó do cluster. O cliente continua acessando o serviço pelo mesmo endpoint lógico (Service), sem perceber que a instância física mudou.

# Bloco de Reflexao

### 1. Qual dos 7 tipos de transparencia voce considera mais dificil de implementar corretamente em um sistema real? Justifique com um argumento tecnico baseado nos exercicios realizados.

Acredito que a **Transparência de Relocação** seja a mais difícil de implementar corretamente. Diferente da migração vista na Tarefa 3, onde o estado é salvo para uma nova instância assumir depois, a relocação exige que o recurso se mova enquanto está em uso.  

No código de análise da Tarefa 4, vimos que é necessário um mecanismo complexo de *buffering* e máquinas de estado para garantir que as mensagens não se percam durante a troca de endpoints. Manter a continuidade da operação sem que o cliente perceba uma queda de conexão em tempo real exige um nível de coordenação e sincronismo extremamente alto entre as réplicas e o cliente.

---

### 2. Descreva um cenario concreto de um sistema que voce conhece (app, site, jogo) em que esconder completamente a distribuicao levaria a um sistema menos resiliente para o usuario final.

Um cenário concreto onde a transparência excessiva prejudica a resiliência é em aplicativos de transações bancárias. Se escondermos completamente que o sistema é distribuído e tratarmos uma chamada de rede como se fosse um método local, como criticado no `anti_pattern.py` da Tarefa 7, o usuário pode ficar olhando para uma tela travada sem saber se o problema é o sinal da internet ou o servidor do banco.  

---

### 3. Como o conceito de async/await explorado no Lab 02 se conecta com a decisao de quebrar a transparencia conscientemente, vista na Tarefa 7?

O uso de `async/await`, explorado no Lab 02, conecta-se diretamente à quebra deliberada de transparência vista na Tarefa 7. Ao marcar uma função como `async`, estamos explicitamente dizendo ao programador que aquela operação é não-bloqueante e, provavelmente, envolve I/O remoto ou latência.  

No `bom_pattern.py`, o uso de `await fetch_user_remote` sinaliza que o fluxo do programa pode ser suspenso enquanto aguarda a rede. Essa “transparência consciente” é fundamental para evitar que o desenvolvedor trate uma chamada externa custosa com a mesma leveza de uma operação em CPU, forçando o tratamento de exceções e *timeouts*.

---

### 4. Explique com suas palavras por que a Tarefa 6 usa multiprocessing em vez de threading. O que e o GIL e por que ele interfere na demonstracao de race conditions em Python?

A Tarefa 6 utiliza o módulo `multiprocessing` porque, no Python (CPython), existe o **GIL (Global Interpreter Lock)**, um mecanismo que impede que múltiplas *threads* executem bytecode simultaneamente no mesmo processador.  

Para demonstrar uma *race condition* real e reproduzível como a do saldo bancário no arquivo `sem_concorrencia.py`, precisamos de processos independentes, cada um com seu próprio espaço de memória e seu próprio GIL. Isso simula com fidelidade um ambiente distribuído, onde diferentes máquinas acessam um recurso compartilhado, como o Redis, tornando o uso de um **Distributed Lock** (exclusão mútua distribuída) estritamente necessário para manter a consistência dos dados.

---

### 5. Descreva uma dificuldade tecnica encontrada durante o laboratorio (incluindo o provisionamento do Redis Cloud), o processo de diagnostico e a solucao. Se nao houve dificuldade, descreva o exercicio mais interessante e explique por que.

A principal dificuldade que enfrentei foi no provisionamento do Redis Cloud. Inicialmente recebi erro de conexão (ConnectionError) porque o `REDIS_HOST` e `REDIS_PORT` estavam sendo acessados corretamente no .env.  Após essa dificuldade, O exercício mais interessante foi a Tarefa 6, especificamente a implementação do *lock* distribuído com o Redis Cloud. Fiquei muito surpreso ao observar como a latência de rede introduzida pelo `time.sleep(0.05)` causava a perda de consistência no saldo final de forma consistente quando não utilizávamos o *lock*.  