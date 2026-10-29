Continue a implementação do NinjaOptimizer na branch padrão do repositório (a mesma onde estão os commits "Catálogo extra A…F"), seguindo o arquivo anexado `features-para-ninjaoptimizer-parte2.md`. Ele substitui a seção "Não implementar" da especificação anterior: decidi que **toda funcionalidade do .bat com efeito real entra no app**, inclusive as arriscadas, desde que cada uma mostre claramente as consequências antes de aplicar. Só o que é placebo fica de fora (lista no fim do arquivo).

## Antes de começar
1. Releia o código atual (catálogo, `Conditions.cs`, `MaintenanceTask.cs`, `RepairTasks.cs`, `WindowsMaintenance.cs`, Modo Gamer, testes) — ele vence a especificação se houver conflito; me avise quando acontecer.
2. Confira se algum Id da especificação já existe. Se existir, não duplique.
3. `dotnet test` verde antes de mexer.

## Regras (as mesmas de antes, mais uma)
- Nada de chamar ou copiar o .bat. C# pelas abstrações do Core; comandos externos só pelo `ICommandRunner`, com argumentos validados (nunca concatenar texto vindo do sistema numa linha de PowerShell sem validar).
- Snapshot antes de alterar, reverter item a item, log de tudo.
- Risco alto: fora dos perfis + confirmação extra. Itens novos desta parte ficam **fora de todos os perfis**.
- Respeitar condições (SSD/HD, notebook/desktop, build do Windows, serviço existe, política/UEFI).
- **Nova:** nenhum item com risco Médio ou Alto, nenhuma tarefa que apague algo sem volta e nenhuma entrada do debloat pode existir sem a lista de consequências (Etapa H). Garanta isso com um teste.
- Textos em português simples, no tom dos itens atuais. Consequências escritas para leigo: o que deixa de funcionar, não jargão.
- Não quebrar snapshots já gravados no `journal.json` (principalmente ao mexer no DNS, Etapa M).

## Ordem — uma etapa por vez
H (avisos de consequências, e preencher nos itens que já existem) → I (debloat com lista para o usuário escolher) → J (ajustes de risco) → K (opções avançadas e reparos de câmera/microfone/Xbox, reiniciar/desligar) → L (Modo Gamer: limpeza automática da memória em espera e timer) → M (DNS com escolha e teste de velocidade).

Para cada etapa:
1. Implemente.
2. Testes no padrão atual (aplicar → reverter → estado idêntico, com valor ausente e existente; condições bloqueiam; ações novas com fakes). No debloat: nunca lista nem remove protegidos, nome de pacote inválido é recusado, desfazer escolhe o caminho certo (provisionado → winget → link da Store). Na `BcdSettingAction`: captura com valor ausente e presente, reverte com `/deletevalue` quando não existia.
3. `dotnet build` e `dotnet test` verdes.
4. Atualize `docs/03-roteiro-de-etapas.md` (tabela de Ids, riscos e "como testar no Windows") e o `README.md`.
5. **Um commit por etapa** com mensagem clara (ex.: `Catálogo extra H: avisos de consequências`) e push na branch padrão.
6. Resumo curto da etapa para mim (o que entrou, testes, o que testar num Windows real) e siga para a próxima.

## Pontos em aberto
- Os Ids da Microsoft Store da tabela do debloat foram anotados de memória: confira com `winget search --source msstore` (ou deixe sem Id; o desfazer cai no link `ms-windows-store://pdp/?PFN=`).
- Se faltar condição (ex.: `MaxWindowsBuild`) ou método na infraestrutura, crie da forma mais simples e testável e cite no resumo.
- Se o CI falhar depois de um push, corrija antes de seguir.
