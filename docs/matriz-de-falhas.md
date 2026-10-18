# Matriz de Falhas e Compensações — TravelFlow

Este documento reúne as transações compensatórias, a ordem de compensação, os cenários de falha, a matriz de decisão para erros e timeouts, o tratamento especial do pagamento e a matriz de resiliência.

---

## Compensação e cenários de falha

### Quais operações serão compensadas?

| Operação original concluída | Transação compensatória |
|---|---|
| Reservar voo | Cancelar reserva de voo |
| Reservar hotel | Cancelar reserva de hotel |
| Reservar carro | Liberar reserva do veículo |
| Reservar restaurante | Cancelar reserva da mesa |
| Reservar passeio | Cancelar reserva ou ingresso |
| Capturar pagamento, se já ocorreu | Solicitar estorno financeiro |

Observe que, se o pagamento for recusado antes de qualquer captura, não existe estorno a executar. A autorização eventualmente obtida deverá ser anulada ou liberada, conforme o gateway.

No cenário principal, o pagamento é recusado e nenhuma cobrança é capturada. Portanto, a Saga cancela as cinco reservas e encerra o pedido como cancelado.

### Ordem da compensação

A compensação segue, preferencialmente, a ordem inversa das operações concluídas:

1. Cancelar passeio.
2. Cancelar reserva de restaurante.
3. Liberar carro.
4. Cancelar hotel.
5. Cancelar voo.

Essa ordem é uma política de implementação, não uma regra universal do padrão Saga. Ela é apropriada para este exemplo porque reverte as etapas na sequência oposta à execução original. Em uma operação real, a prioridade também deve considerar prazos de cancelamento, penalidades, expiração das reservas e dependências entre fornecedores.

> O diagrama de sequência da falha está no [`README.md`](../README.md) e em `diagrams/sequence-compensation-flow.png`.

### E se uma compensação também falhar?

Imagine que o hotel esteja indisponível para processar o cancelamento no momento em que o orquestrador enviar o comando.

O sistema não deve considerar a Saga concluída. Ele deve:

- Registrar a compensação como pendente.
- Repetir a operação quando for seguro fazê-lo.
- Manter o pedido em estado de cancelamento em andamento.
- Alertar a equipe se o limite de tentativas for excedido.
- Retomar a compensação a partir do estado persistido.
- Confirmar o cancelamento somente depois de resolver todas as etapas obrigatórias.

Se um fornecedor não aceitar o cancelamento, por exemplo por causa de uma regra tarifária, o sistema precisa aplicar a política comercial definida: negociar com o fornecedor, registrar uma pendência financeira ou encaminhar o caso para atendimento humano. Uma compensação de software não garante que um fornecedor externo permita o cancelamento.

### Um segundo cenário de falha: timeout durante a reserva

Há outra situação que merece ser documentada: o serviço de hotel recebe `ReserveHotel`, realiza a reserva no fornecedor, mas a resposta se perde na rede.

O orquestrador não sabe se a reserva foi criada. Ele não deve presumir que falhou nem enviar comandos contraditórios.

O procedimento é:

1. Registrar o timeout e manter a etapa em estado indeterminado.
2. Consultar o resultado usando o identificador de correlação ou a chave idempotente da operação.
3. Se a reserva existir, registrar a confirmação e continuar ou compensar, conforme o estado da Saga.
4. Se houver evidência confiável de que a operação não ocorreu, repetir o comando com a mesma chave idempotente.
5. Se não for possível determinar o resultado, manter a etapa pendente e encaminhar para reconciliação.


---

## Erros, timeouts e pagamento

### Matriz de decisão para erros e timeouts


| Situação | Decisão | Efeito na Saga |
|---|---|---|
| Reserva confirmada | Persistir confirmação e avançar | Próxima etapa |
| Reserva recusada definitivamente | Não repetir automaticamente | Iniciar compensação |
| Timeout após enviar reserva | Consultar estado antes de repetir | Manter etapa indeterminada |
| Erro HTTP 429 ou indisponibilidade temporária | Repetir com espera progressiva e limite | Manter etapa pendente |
| Erro de validação do pacote | Corrigir ou rejeitar a solicitação | Não iniciar ou interromper antes de reservar |
| Pagamento recusado | Não repetir a mesma cobrança como se fosse erro transitório | Iniciar compensação |
| Timeout na captura do pagamento | Consultar o gateway usando a referência da operação | Não cancelar reservas até esclarecer o resultado |
| Mensagem duplicada | Consultar o registro idempotente | Não repetir o efeito |
| Compensação temporariamente indisponível | Repetir com controle e persistir pendência | Manter `COMPENSATING` |
| Compensação permanentemente rejeitada | Aplicar política comercial e escalar | Manter pendência operacional |

**Regra essencial:** timeout não é sinônimo de falha definitiva. Uma requisição pode ter sido processada pelo destino mesmo que o remetente não tenha recebido a resposta.

### Um cuidado especial com o pagamento

A captura financeira exige tratamento mais rigoroso do que uma simples consulta.

Se o gateway receber uma solicitação de captura e a resposta se perder, repetir a captura com uma nova chave pode gerar uma cobrança duplicada. Por isso, o Payment Service deve:

1. Persistir a referência da operação financeira.
2. Enviar a solicitação com uma chave idempotente estável.
3. Consultar o estado da captura antes de tomar uma decisão baseada em timeout.
4. Não iniciar um segundo pagamento enquanto o primeiro estiver indeterminado.
5. Solicitar estorno se houver captura confirmada e a política exigir reversão.
6. Registrar e reconciliar operações que permaneçam sem resultado conhecido.

O sistema também precisa distinguir entre autorização, captura, cancelamento de autorização e estorno. São operações diferentes e não devem ser tratadas como sinônimos.

---

## Matriz de resiliência

| Mecanismo | Problema mitigado | Limitação |
|---|---|---|
| Retry com backoff e jitter | Falhas transitórias | Não resolve falhas permanentes |
| Circuit Breaker | Indisponibilidade persistente de fornecedor | Não conclui nem compensa uma Saga |
| DLQ | Mensagens que excedem a política de processamento | Exige análise e recuperação |
| Transactional Outbox | Perda de publicação após commit local | Pode gerar duplicidade de mensagens |
| Consumidor idempotente | Efeitos duplicados | Precisa de persistência e controle de concorrência |
| Timeout e reconciliação | Operações com resultado desconhecido | Pode exigir consulta ao fornecedor ou intervenção |
| Observabilidade | Falhas difíceis de localizar | Depende de instrumentação e alertas úteis |