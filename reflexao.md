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