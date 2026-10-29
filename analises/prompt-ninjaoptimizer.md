Implemente na branch `main` do NinjaOptimizer as features descritas no arquivo anexado `features-para-ninjaoptimizer.md`. Ele é a especificação: veio da engenharia reversa de um .bat de otimização, já filtrado (só o que é útil e ainda não existe no app) e já mapeado para a nossa arquitetura. Siga o arquivo. Se algo nele contradisser o código atual, o código atual vence: me avise e adapte.

## Antes de começar
1. Leia `docs/02-arquitetura.md`, `docs/03-roteiro-de-etapas.md`, `src/NinjaOptimizer.Core/Optimization/Tweak.cs`, `TweakCatalog.cs`, a pasta `Actions/`, `Maintenance/MaintenanceTask.cs`, `src/NinjaOptimizer.Windows/Maintenance/WindowsMaintenance.cs`, `GameMode/GameMode.cs` e os testes em `tests/NinjaOptimizer.Core.Tests/` (principalmente `CatalogTests.cs` e `Fakes.cs`).
2. Confira se nenhum Id da especificação já existe no catálogo (pode ter mudado desde a análise). Se já existir, não duplique: só me avise.
3. Rode `dotnet test` e confirme que tudo passa antes de mexer em qualquer coisa.

## Regras
- **Nunca** chame o .bat nem copie código dele. Tudo em C#, pelas abstrações `IRegistryAccess`, `IServiceManager`, `ICommandRunner` (o Core continua sem conhecer o Windows; implementações reais ficam em `NinjaOptimizer.Windows`). PowerShell, `powercfg`, `netsh` etc. só pelo `ICommandRunner` e só quando não houver API melhor.
- Todo ajuste guarda o estado anterior no snapshot antes de alterar, reverte item a item e aparece na interface com toggle, nome, descrição, risco, reinício e reversível — igual aos que já existem.
- Risco alto: fora de qualquer perfil e com a confirmação extra do motor.
- Respeite as condições (SSD, notebook, fabricante da GPU, serviço existe). Nada de aplicar ajuste de notebook em desktop ou SysMain em HD.
- Textos da interface em português simples, no mesmo tom dos itens atuais.
- Não mude o comportamento dos ajustes que já existem, exceto o que a especificação pede (condição de SSD no `services.sysmain`). Não quebre snapshots já gravados no `journal.json` de usuários.

## Ordem — uma etapa por vez
A (condições) → B (serviços e privacidade) → C (visual) → D (energia e periféricos) → E (prioridade por jogo no Modo Gamer) → F (manutenção e reparos).
**Não implemente a Etapa G** (Integridade de Memória / VBS) sem eu confirmar explicitamente.

Para cada etapa:
1. Implemente.
2. Escreva testes no padrão atual: aplicar → reverter → estado idêntico, com valor ausente e com valor já existente; condições falsas bloqueiam; ações novas (`NetAdapterPowerAction`, `ProcessPriorityAction`) testadas com fakes, inclusive reverter quando a placa/jogo não existe mais e a validação do nome do `.exe` (recusar caminhos e qualquer valor fora de `PerfOptions`).
3. Rode `dotnet build` e `dotnet test`; só siga com tudo verde.
4. Atualize `docs/03-roteiro-de-etapas.md` (seção nova com tabela de Ids e "como testar no Windows") e o `README.md` se o estado mudar.
5. Faça **um commit por etapa** com mensagem clara (ex.: `Etapa 3A: condições de aplicabilidade e SysMain só em SSD`) e dê push na `main`.
6. Me mande um resumo curto da etapa (o que entrou, testes, o que testar num Windows real) e siga para a próxima.

## Pontos em aberto na especificação
- O padrão de início do `NlaSvc` no Windows 11 não foi confirmado: na tarefa `maintenance.restore-default-services`, se não tiver certeza, deixe esse serviço de fora e anote no roteiro.
- Os nomes de executáveis de jogos da especificação não foram conferidos um a um: use-os só como sugestão; o padrão é o executável que o Modo Gamer detectou rodando.
- Se para alguma feature faltar algo na infraestrutura (ex.: apagar chave vazia no `IRegistryAccess`), adicione da forma mais simples e testável e cite no resumo.

Se o CI (`.github/workflows/build.yml`) falhar depois de um push, corrija antes de seguir para a próxima etapa.
