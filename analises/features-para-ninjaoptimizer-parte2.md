# Features para o NinjaOptimizer — parte 2 (avisos, debloat e o resto do `.bat`)

> **Para quem vai implementar (sessão do NinjaOptimizer):** continuação de `features-para-ninjaoptimizer.md`.
> As etapas A–F daquele arquivo já estão no código (commits "Catálogo extra A…F"). Esta parte **substitui**
> a seção "Não implementar" de lá: o dono do projeto decidiu que **toda funcionalidade do .bat que tem efeito
> real entra no app**, inclusive as arriscadas — desde que cada uma deixe claro, antes de aplicar, que
> consequências pode trazer. Só fica de fora o que é placebo (lista no fim).
>
> Mesmas regras de antes: C# pelas abstrações do Core, nada de chamar ou copiar o .bat, snapshot antes de
> alterar, reverter item a item, um commit por etapa com testes, atualizar `docs/03-roteiro-de-etapas.md`.

## Ordem sugerida

| Etapa | Conteúdo |
|---|---|
| H | Sistema de avisos de consequências (infra) + preencher nos itens que já existem |
| I | Debloat com lista para o usuário escolher |
| J | Ajustes de risco que estavam fora (segurança, serviços, compatibilidade, Prefetch…) |
| K | Opções avançadas nas tarefas de manutenção + reparos de câmera, microfone e Xbox |
| L | Modo Gamer: timer de alta resolução e limpeza automática da memória em espera (o que o ISLC fazia) |
| M | DNS com escolha de provedor e teste de velocidade (o que o DNS Jumper fazia) |

---

## Etapa H — Avisos de consequências (infraestrutura)

**Objetivo:** todo item que pode trazer efeito colateral mostra, **antes de aplicar**, o que pode acontecer.

- `Tweak` ganha `IReadOnlyList<string> Consequences` (frases curtas, uma por consequência). O `PortableWarning` que já existe continua; ele é somado aos avisos quando o PC é notebook.
- `MaintenanceTask` ganha o mesmo `Consequences` (o `Note` atual continua como descrição curta). Cada entrada do debloat (Etapa I) também tem as suas.
- **Interface:**
  - Risco **Baixo** com consequências → linha discreta "O que muda" expansível.
  - Risco **Médio** → caixa amarela "Antes de aplicar" sempre visível no card.
  - Risco **Alto** → caixa vermelha + caixa de seleção **"Entendi as consequências"**; o botão de aplicar e a seleção em lote só aceitam o item depois de marcada. O diálogo de confirmação de risco alto do motor lista as consequências de novo.
- **Teste obrigatório:** todo `Tweak` com `Risk >= Medium` e toda `MaintenanceTask` que apaga algo sem volta ou reinicia serviço/PC tem `Consequences` não vazio. Assim nenhum item novo entra sem aviso.
- **Preencher nos itens que já existem** (conferir a lista atual no catálogo; sugestões de texto):

| Id | Consequências sugeridas |
|---|---|
| `cpu.ultimate-power-plan` | Mais consumo de energia e temperatura; em notebook a bateria dura menos e a ventoinha gira mais. |
| `system.background-apps` | Apps da Store (Telefone, Email, Teams da Store) param de mostrar notificações e de sincronizar até serem abertos. |
| `gaming.fullscreen-optimizations` | HDR automático, sobreposições do Windows e troca rápida Alt+Tab podem parar de funcionar em alguns jogos. |
| `gpu.hags` | Em poucos jogos antigos e alguns programas de captura pode causar engasgos ou tela preta; exige reiniciar. |
| `gpu.mpo-off` | Em vídeos e navegador o uso de GPU pode subir um pouco; em builds recentes do Windows pode não ter efeito. |
| `storage.hibernation-off` | O PC deixa de hibernar e a Inicialização Rápida é desligada (o boot a frio fica um pouco mais lento). |
| `memory.pagefile-auto` | Substitui o tamanho de arquivo de paginação que você configurou à mão. |
| `memory.compression-off` | Com menos de 32 GB de RAM o Windows pode usar mais o disco e ficar mais lento. |
| `services.sysmain` | Programas podem abrir um pouco mais devagar nas primeiras vezes. |
| `services.search-indexer` | A busca do Iniciar e do Explorador fica mais lenta e pode não achar conteúdo dentro de arquivos e e-mails do Outlook. |
| `services.print-spooler` | Nenhuma impressora funciona, nem "Imprimir em PDF". |
| `network.dns-cloudflare` | Em rede de empresa/escola, sites e pastas internas podem parar de abrir. |
| `network.adapter-power-saving` | A placa reinicia por alguns segundos ao aplicar; no notebook gasta mais bateria. |
| `gaming.process-priority` (Alta) | Áudio do Discord/OBS e outros programas podem engasgar enquanto o jogo roda. |
| `services.error-reporting` | A Microsoft e os desenvolvedores deixam de receber relatórios de travamento; soluções automáticas para travamentos param de aparecer. |
| `maintenance.repair-network` | Apaga IP fixo e DNS manuais e pode remover filtros de VPN/antivírus antigos; exige reiniciar. |
| `maintenance.repair-windows-update` | O histórico do Windows Update fica vazio. |
| `cleanup.recycle-bin` | Os arquivos da Lixeira são apagados para sempre. |

---

## Etapa I — Debloat (remover apps com escolha do usuário)

Nova página ou seção **"Apps do Windows"** (sugestão: dentro de Otimizar ou Limpeza). **Fora de todos os perfis prontos.**

### Como funciona
1. Lista **só os apps instalados** para o usuário atual: `Get-AppxPackage` (via `ICommandRunner`, saída em JSON com `ConvertTo-Json`), ignorando `IsFramework`, `NonRemovable`, `SignatureKind = System` e os da lista **Protegidos** abaixo.
2. Cada app aparece com: ícone/nome amigável, o que ele é, **o que acontece se remover** (consequências), selo de recomendação e se dá para reinstalar.
   - Selos: **Seguro remover** (verde) · **Cuidado** (amarelo) · **Não recomendado** (vermelho, exige "Entendi as consequências").
   - Apps que não estão no catálogo abaixo ficam escondidos atrás de "Mostrar outros apps instalados", com o aviso genérico: "O NinjaOptimizer não conhece este app. Remover pode quebrar o programa que depende dele."
3. Nada vem marcado. O usuário marca o que quer; botão "Remover selecionados" mostra um resumo final com todas as consequências juntas antes de executar.
4. **Remover:** `Remove-AppxPackage -Package <PackageFullName>` (só o usuário atual). Opção avançada, risco alto, desmarcada: **"Remover também para contas novas"** (`Remove-AppxProvisionedPackage -Online -PackageName <…>`), com o aviso: "Contas criadas depois neste PC não terão o app, e reinstalar pode exigir a Microsoft Store."
5. **Journal / desfazer:** antes de remover, gravar `Name`, `PackageFullName`, `PackageFamilyName`, se estava provisionado e o Id da Store (se conhecido). *Desfazer* tenta, nesta ordem:
   1. se o pacote ainda está provisionado no sistema → `Add-AppxPackage -Register "<InstallLocation do provisionado>\AppXManifest.xml" -DisableDevelopmentMode`;
   2. senão, se há Id da Store → `winget install --id <Id> --source msstore --accept-package-agreements --accept-source-agreements`;
   3. senão → abrir a página do app na Store com `ms-windows-store://pdp/?PFN=<PackageFamilyName>` e avisar que a reinstalação é manual.
   A interface marca esses itens como **"Reversível: reinstala (precisa de internet)"**.
6. **OneDrive não é um app da Store** (o `*onedrive*` do .bat não remove nada na maioria dos PCs). Tratar à parte: `winget uninstall --id Microsoft.OneDrive` (ou `%SystemRoot%\System32\OneDriveSetup.exe /uninstall`); desfazer com `winget install --id Microsoft.OneDrive`.
7. Testes com fakes: nunca lista/remove protegidos; removido entra no journal; desfazer escolhe o caminho certo (provisionado → winget → link); nome de pacote com caracteres inválidos é recusado (evitar injeção na linha do PowerShell — passar nomes só depois de validar contra `^[A-Za-z0-9._-]+$`).

### Catálogo de apps

Os Ids da Store foram anotados de memória — **conferir com `winget search --source msstore "<nome>"`** antes de usar; se não bater, deixar sem Id (cai no link da Store).

| Pacote (`Name`) | Nome amigável | Selo | Consequências ao remover | Id Store (conferir) | No .bat? |
|---|---|---|---|---|---|
| `Microsoft.BingNews` | Notícias | Seguro | Some o app Notícias. O widget de notícias da barra continua. | `9WZDNCRFHVFW` | Sim |
| `Microsoft.BingWeather` | Clima | Seguro | Some o app Clima. | `9WZDNCRFJ3Q2` | Sim |
| `Microsoft.GetHelp` | Obter Ajuda | Seguro | Alguns botões "Obter ajuda" do Windows param de abrir. | — | Extra |
| `Microsoft.Getstarted` | Dicas | Seguro | Some o app de dicas do Windows. | — | Sim |
| `Microsoft.MicrosoftOfficeHub` | Microsoft 365 (Office) | Seguro | Some o atalho/hub do Office. O Office instalado (Word, Excel…) **não** é afetado. | `9WZDNCRD29V9` | Sim |
| `Microsoft.MicrosoftSolitaireCollection` | Paciência | Seguro | Perde o jogo e o progresso local. | `9WZDNCRFHWD2` | Sim |
| `Microsoft.WindowsFeedbackHub` | Hub de Comentários | Seguro | Não dá mais para enviar comentários à Microsoft pelo app. | `9NBLGGH4R32N` | Sim |
| `Microsoft.3DBuilder`, `Microsoft.Microsoft3DViewer` | 3D Builder / Visualizador 3D | Seguro | Arquivos 3D (.3mf, .stl) deixam de abrir com esses apps. | — | Sim |
| `Microsoft.People` | Pessoas | Seguro | Some o app Pessoas (contatos do Email antigo). | — | Sim |
| `Microsoft.Messaging` | Mensagens | Seguro | App antigo, sem uso no Windows atual. | — | Sim |
| `Microsoft.WindowsMaps` | Mapas | Seguro | Some o app Mapas e os mapas offline baixados. | `9WZDNCRDTBVB` | Sim |
| `Clipchamp.Clipchamp` | Clipchamp | Seguro | Some o editor de vídeo. | `9P1J8S7CCWWT` | Extra |
| `Microsoft.ZuneVideo` | Filmes e TV | Cuidado | Vídeos comprados na Microsoft podem não abrir; arquivos de vídeo abrem em outro player. | `9WZDNCRFJ3P2` | Extra (`*zune*`/`*music*`) |
| `Microsoft.WindowsAlarms` | Relógio (Alarmes) | Cuidado | Alarmes, timer e **Sessões de foco** param de funcionar. | `9WZDNCRFJ3PR` | Sim |
| `microsoft.windowscommunicationsapps` | Email e Calendário | Cuidado | Contas de e-mail e calendário configuradas nele somem do PC (os e-mails continuam no servidor). Lembretes do calendário param. | `9WZDNCRFHVQM` | Sim |
| `Microsoft.YourPhone` | Vincular ao Celular | Cuidado | Para de mostrar notificações, fotos e mensagens do celular no PC. | `9NMPJ99VJBWV` | Sim (`*phone*`) |
| `MicrosoftCorporationII.QuickAssist` | Assistência Rápida | Cuidado | Não dá mais para receber ou dar suporte remoto por ele (golpistas usam muito — remover também protege). | `9P7BP5VNWKX5` | Sim |
| `Microsoft.549981C3F5F10` | Cortana | Cuidado | Some a Cortana (já descontinuada no Windows 11). | `9NFFX4SZZ23L` | Sim |
| `Microsoft.ZuneMusic` | **Media Player** (Win 11) / Groove (Win 10) | Não recomendado | No Windows 11 **remove o Media Player**: músicas e vídeos passam a abrir em outro programa ou não abrem. | `9WZDNCRFJ3PT` | Sim (`*music*`) |
| `Microsoft.WindowsCamera` | Câmera | Não recomendado | O app Câmera some; a webcam continua funcionando em outros apps (Teams, Discord, navegador). | `9WZDNCRFJBBG` | Extra |
| `Microsoft.Windows.Photos` | Fotos | Não recomendado | Imagens deixam de abrir com dois cliques até você escolher outro programa. | `9WZDNCRFJBH4` | Citado no texto do .bat |
| `Microsoft.GamingApp` / `Microsoft.XboxApp` | Xbox | Não recomendado | **Jogos do Game Pass para PC deixam de abrir e atualizar.** | `9MV0B5HZVK9Z` | Sim (`*Xbox*`) |
| `Microsoft.XboxGamingOverlay` | Xbox Game Bar | Não recomendado | Win+G, gravação de tela, FPS da Game Bar e widgets somem. Alguns jogos mostram erro ao tentar abrir a Game Bar. | `9NZKPSTSNW4P` | Sim (`*Xbox*`) |
| `Microsoft.XboxGameOverlay`, `Microsoft.XboxSpeechToTextOverlay` | Componentes da Game Bar | Não recomendado | Partes da Game Bar e da conversão de voz em texto param. | — | Sim (`*Xbox*`) |
| `Microsoft.XboxIdentityProvider`, `Microsoft.Xbox.TCUI` | Login Xbox | Não recomendado | **Jogos que usam conta Xbox (Minecraft, Forza, Halo, Sea of Thieves, jogos da Microsoft na Steam) não conseguem entrar na conta.** | — | Sim (`*Xbox*`) |
| OneDrive (Win32, ver item 6) | OneDrive | Cuidado | Para de sincronizar. **Arquivos que estão só na nuvem ("sob demanda") deixam de aparecer no PC** — eles continuam no site do OneDrive. Pastas como Documentos/Área de Trabalho podem estar dentro do OneDrive: conferir antes. | `Microsoft.OneDrive` (winget) | Sim |

### Protegidos — nunca listar nem remover
`Microsoft.WindowsStore`, `Microsoft.StorePurchaseApp`, `Microsoft.DesktopAppInstaller` (winget), `Microsoft.SecHealthUI` (Segurança do Windows), `Microsoft.Windows.ShellExperienceHost`, `Microsoft.Windows.StartMenuExperienceHost`, `MicrosoftWindows.Client.*`, `Microsoft.AAD.BrokerPlugin`, `Microsoft.AccountsControl`, `Microsoft.Windows.CloudExperienceHost`, `Microsoft.LockApp`, `Microsoft.WindowsTerminal`, `Microsoft.WindowsNotepad`, `Microsoft.Paint`, `Microsoft.ScreenSketch` (Ferramenta de Captura), `Microsoft.WindowsCalculator`, todas as extensões de mídia (`Microsoft.*Extension`, `Microsoft.HEVCVideoExtension`…), e todos os frameworks (`Microsoft.VCLibs.*`, `Microsoft.UI.Xaml.*`, `Microsoft.NET.Native.*`, `Microsoft.WindowsAppRuntime.*`).

### Não fazer (o que o .bat fazia de errado)
- Curingas (`*phone*`, `*music*`) — pegam apps de terceiros (ex.: `SpotifyAB.SpotifyMusic`).
- Remover provisionado por padrão.
- "Reverter" re-registrando todos os pacotes do sistema.

---

## Etapa J — Ajustes de risco que estavam fora

Todos **fora dos perfis prontos**, com `Consequences` preenchido (Etapa H).

### J1. `security.memory-integrity-off` — Desativar a Integridade de Memória (HVCI) e o VBS
- **Ações:** `LmDword(@"SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity", "Enabled", 0)` · `LmDword(@"SYSTEM\CurrentControlSet\Control\DeviceGuard", "EnableVirtualizationBasedSecurity", 0)`.
- **Risco:** Alto · **Reinício:** sim · **Impacto:** médio em CPU antiga (antes de Intel 11ª / Ryzen 3000), baixo em CPU nova.
- **Padrão:** ligado em instalação nova do Win 11; desligado em muitos upgrades.
- **Condição:** bloquear com motivo se imposto por política (`HKLM\SOFTWARE\Policies\Microsoft\Windows\DeviceGuard`) ou travado por UEFI. Estado real por `Win32_DeviceGuard` (`root\Microsoft\Windows\DeviceGuard`).
- **Consequências:** "O Windows fica mais exposto a drivers maliciosos e a ataques que usam falhas de driver." · "A Segurança do Windows mostra um alerta." · "Uma atualização do Windows pode religar sozinha."

### J2. `security.credential-guard-off` — Desativar o Credential Guard
- **Ação:** `LmDword(@"SYSTEM\CurrentControlSet\Control\Lsa", "LsaCfgFlags", 0)`.
- **Risco:** Alto · **Reinício:** sim · **Impacto de desempenho:** quase nenhum (deixar isso claro no texto: o benefício é só liberar o recurso, não FPS).
- **Condição:** só aparece se o Credential Guard estiver rodando (`Win32_DeviceGuard.SecurityServicesRunning` contém `1`); se travado por UEFI, bloquear com motivo.
- **Consequências:** "Senhas e tokens de login ficam menos protegidos contra programas que roubam credenciais da memória." · "Em PC de empresa, pode violar a política de segurança."

### J3. `security.hypervisor-off` — Não carregar o hipervisor do Windows
- **Ação nova `BcdSettingAction`:** captura com `bcdedit /enum {current}` (ler `hypervisorlaunchtype`; ausente = não definido), aplica `bcdedit /set {current} hypervisorlaunchtype off`, reverte com `/set … <valor anterior>` ou `/deletevalue {current} hypervisorlaunchtype` se não existia. Atenção: a saída do `bcdedit` é traduzida; o nome do campo `hypervisorlaunchtype` não é, os valores `Auto`/`Off` não são.
- **Risco:** Alto · **Reinício:** sim.
- **Consequências:** "**WSL2, Hyper-V, Windows Sandbox, Docker Desktop e emuladores Android baseados em Hyper-V param de funcionar.**" · "Desliga também o VBS e a Integridade de Memória." · "Se o disco usa BitLocker, tenha a chave de recuperação à mão por precaução."

### J4. `services.biometrics-off` — Desativar o serviço de Biometria
- **Ação:** `ServiceStartupAction("WbioSrvc", Disabled)` · **Padrão:** Manual · **Risco:** Médio.
- **Consequências:** "**O Windows Hello por digital ou rosto para de funcionar** — você vai precisar do PIN ou da senha." · "Ganho de desempenho praticamente nenhum (o serviço só roda quando é usado)."

### J5. `gaming.xbox-services-off` — Desativar os serviços Xbox
- **Ações:** `ServiceStartupAction` Disabled para `XblAuthManager`, `XblGameSave`, `XboxNetApiSvc`, `XboxGipSvc` · **Padrão:** Manual · **Risco:** Médio.
- **Consequências:** "Jogos do Game Pass e jogos que usam conta Xbox não entram na conta." · "Saves na nuvem do Xbox param de sincronizar." · "**Controles e acessórios Xbox (firmware, botões extras) podem parar** (`XboxGipSvc`)." · "Ganho de desempenho praticamente nenhum: esses serviços só rodam quando algo os usa."

### J6. `privacy.cortana-off` — Desativar a Cortana
- **Ações:** `LmDword(@"SOFTWARE\Policies\Microsoft\Windows\Windows Search", "AllowCortana", 0)` · `CuDword(@"Software\Microsoft\Windows\CurrentVersion\Search", "CortanaConsent", 0)`.
- **Condição:** `MaxWindowsBuild(21999)` (só Windows 10; criar a condição se não existir). **Risco:** Baixo.

### J7. `system.taskbar-search-hidden` — Esconder a pesquisa da barra de tarefas
- **Ação:** `CuDword(@"Software\Microsoft\Windows\CurrentVersion\Search", "SearchboxTaskbarMode", 0)` · **Padrão:** 1–3 conforme a versão (o snapshot guarda) · **Risco:** Baixo · preferência visual, sem ganho de desempenho (dizer isso). A pesquisa continua com a tecla Windows + digitar.

### J8. `disk.prefetcher-off` — Desativar o Prefetch
- **Ação:** `LmDword(@"SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management\PrefetchParameters", "EnablePrefetcher", 0)` · **Padrão:** 3 · **Risco:** Médio · **Reinício:** sim.
- **Consequências:** "**O Windows e os programas tendem a abrir mais devagar**, principalmente em HD." · "Só faz sentido para testar se o disco fica a 100% por causa disso." Aviso extra quando o disco do Windows é HD: "Num HD o Prefetch é o que mais ajuda — desligar costuma piorar."
- **Não gravar** `EnableSuperfetch` (o Windows 10/11 ignora).

### J9. `compat.pca-off` — Desativar o Assistente de Compatibilidade de Programas
- **Ações:** `ServiceStartupAction("PcaSvc", Disabled)` · `LmDword(@"SOFTWARE\Policies\Microsoft\Windows\AppCompat", "DisablePCA", 1)` · `LmDword(…AppCompat, "DisableInventory", 1)` · **Risco:** Médio.
- **Consequências:** "O Windows para de oferecer 'executar com configurações recomendadas' quando um programa antigo falha ao instalar ou abrir."

### J10. `compat.engine-off` — Desativar o motor de compatibilidade de aplicativos
- **Ação:** `LmDword(@"SOFTWARE\Policies\Microsoft\Windows\AppCompat", "DisableEngine", 1)` · **Risco:** Alto · **Reinício:** sim.
- **Consequências:** "**Programas e jogos antigos que dependem das correções de compatibilidade do Windows podem parar de abrir ou travar.**" · "Configurações de compatibilidade feitas em Propriedades → Compatibilidade deixam de valer."

### J11. `update.block-internet-locations` — Impedir o Windows Update de usar servidores da internet
- **Ação:** `LmDword(@"SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate", "DoNotConnectToWindowsUpdateInternetLocations", 1)` · **Risco:** Alto · **Categoria:** Sistema.
- **Descrição honesta:** "Política para PCs que recebem atualizações de um servidor da empresa (WSUS). **Não é telemetria** e não acelera o PC." (o .bat apresentava como telemetria).
- **Condição:** recomendar só quando existir `WUServer` em `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate`; sem isso, mostrar o aviso "Este PC não usa WSUS: este ajuste quase não muda nada e pode quebrar a Store".
- **Consequências:** "**A Microsoft Store e alguns downloads de recursos do Windows podem parar de funcionar.**"

---

## Etapa K — Manutenção: opções avançadas e reparos que faltam

### K1. Opções avançadas nas tarefas que já existem
Cada opção vem **desmarcada**, com suas consequências ao lado.

| Tarefa | Opção avançada | Consequências |
|---|---|---|
| `maintenance.repair-network` | "Também pedir um novo IP ao roteador" (`ipconfig /release` + `/renew`) | "A internet cai por alguns segundos. Não melhora o ping; só ajuda se houver conflito de IP." |
| `maintenance.check-disk` | "Verificação completa no próximo boot" (`chkdsk <unidade> /f /r`) | "**O PC fica inutilizável durante a verificação, que pode levar horas num HD grande.** Não desligue no meio." |
| `maintenance.repair-store` | "Re-registrar todos os apps do sistema" (todos os pacotes, todos os usuários) | "Pode levar vários minutos e mostrar muitos erros, que são normais." |
| `cleanup.recent-items` | "Também limpar listas de atalho e fixados" (`AutomaticDestinations` e `CustomDestinations`) | "**As pastas fixadas no Acesso Rápido e os itens fixados nas listas dos programas são apagados.**" |

### K2. `memory.working-sets` — Liberar a memória dos programas
- Reaproveitar o mesmo caminho do `memory.standby-list` (`NtSetSystemInformation` com `SystemMemoryListInformation`): comando `MemoryEmptyWorkingSets` (2) e, opcional, `MemoryFlushModifiedList` (3). Admin (privilégio `SeProfileSingleProcessPrivilege`).
- **Consequências:** "O uso de RAM cai na hora, mas os programas abertos ficam lentos por alguns segundos enquanto recarregam o que precisam do disco." · "Durante um jogo pode causar engasgos — não use com o jogo aberto."

### K3. `maintenance.repair-xbox-apps` — Reparar os apps Xbox
- Garantir os 4 serviços Xbox fora de "Desativado" (voltar a Manual) → reiniciar os que estiverem rodando → re-registrar só os pacotes Xbox instalados (`Microsoft.GamingApp`, `Microsoft.XboxGamingOverlay`, `Microsoft.XboxIdentityProvider`, `Microsoft.Xbox.TCUI`) com `Add-AppxPackage -Register … -DisableDevelopmentMode`.
- Se `gaming.xbox-services-off` estiver aplicado no journal, avisar e oferecer *Desfazer* em vez de mexer por fora.

### K4. `maintenance.repair-camera` — Reparar a câmera
Passos, cada um registrado no log:
1. **Programas que podem estar usando a câmera:** listar os que estão abertos entre Teams, Zoom, Discord, Skype, OBS, Câmera. **Perguntar** antes de fechar (lista com caixas de seleção); fechar com `CloseMainWindow` e só forçar se o usuário escolher "Forçar". Consequência: "Trabalho não salvo e chamadas em andamento nesses programas são perdidos."
2. Reiniciar `FrameServer` e `FrameServerMonitor` (sem mudar o tipo de início).
3. Desabilitar e reabilitar **só** dispositivos das classes `Camera` e `Image` (`Disable-PnpDevice`/`Enable-PnpDevice`), depois `pnputil /scan-devices`.
4. Re-registrar `Microsoft.WindowsCamera`, se instalado.
5. Mostrar o estado final dos dispositivos e abrir `ms-settings:privacy-webcam`.
6. **Opção avançada (desmarcada) — "Liberar a câmera para os apps":** grava `Value = "Allow"` em `HKCU\…\CapabilityAccessManager\ConsentStore\webcam` e `…\webcam\NonPackaged`; com uma segunda opção "para todos os usuários deste PC" também em `HKLM`. Guardar os valores anteriores no log/journal para poder voltar. Consequência: "**Todos os apps passam a poder usar sua câmera** — revise em Configurações → Privacidade."
7. **Opção avançada (desmarcada, risco alto) — "Remover bloqueios de política":** apaga `HKLM/HKCU\SOFTWARE\Policies\Microsoft\Camera\AllowCamera` e `…\Windows\AppPrivacy\LetAppsAccessCamera` (guardando antes). Consequência: "**Em PC de empresa ou escola, isso desfaz uma regra do administrador** e ela pode voltar na próxima sincronização."
8. Opcional: "Procurar drivers no Windows Update" (`UsoClient StartScan`).

### K5. `maintenance.repair-microphone` — Reparar o microfone
Igual à K4, trocando:
- serviços: `AudioEndpointBuilder` e `Audiosrv` (mesma ordem da `maintenance.restart-audio`);
- dispositivos: classe `AudioEndpoint` por padrão; **opção avançada** incluir a classe `MEDIA`, com a consequência "Inclui o áudio HDMI da placa de vídeo, placas de captura e interfaces de áudio: o som delas cai por alguns segundos";
- permissões: `ConsentStore\microphone` e política `LetAppsAccessMicrophone`;
- ao final abrir `ms-settings:sound`.

### K6. Reiniciar e desligar
- Quando um ajuste aplicado exigir reinício, mostrar um aviso com o botão **"Reiniciar agora"** (`shutdown /r /t 0` pelo `ICommandRunner`), pedindo para salvar o trabalho antes.
- Em Manutenção: **"Desligar à força"** (`shutdown /s /f /t 0`), para quando o PC não desliga normalmente. Consequências: "**Fecha todos os programas sem salvar nada.**" · Dica no texto: "Se o PC nunca desliga direito, tente desativar a Inicialização Rápida" (o .bat religava a hibernação, que não resolve isso).

---

## Etapa L — Modo Gamer: o que o ISLC fazia

### L1. Limpeza automática da memória em espera durante o jogo
- Configuração no Modo Gamer: "Limpar a memória em espera automaticamente" (desligado por padrão), com dois limites: lista em espera acima de **X MB** (padrão 1024) **e** memória livre abaixo de **Y MB** (padrão 1024). Verificar a cada 5 s só enquanto o Modo Gamer estiver ativo; reaproveitar a limpeza do `memory.standby-list`.
- Consequência: "Pode causar um engasgo curto no momento da limpeza; arquivos e programas usados há pouco recarregam do disco."

### L2. Timer de alta resolução durante o jogo
- Enquanto o Modo Gamer estiver ativo, pedir resolução de timer de 0,5 ms (`NtSetTimerResolution`, `ntdll`) e soltar ao desativar (e ao fechar/travar o app — o Windows solta sozinho quando o processo morre).
- **Windows 11:** a resolução pedida só vale para o próprio app, a menos que exista `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\kernel\GlobalTimerResolutionRequests = 1`. Criar o ajuste `gaming.global-timer-resolution` (`LmDword(…kernel, "GlobalTimerResolutionRequests", 1)`, condição `MinWindowsBuild(22000)`, Médio, reinício).
- Consequências: "Mais consumo de energia e CPU um pouco mais ativa; em notebook a bateria dura menos." · "O ganho de latência é pequeno e varia por jogo."

---

## Etapa M — DNS com escolha (o que o DNS Jumper fazia)

- Generalizar o `network.dns-cloudflare` sem quebrar o Id antigo (snapshots no journal): manter o item existente e adicionar uma escolha de provedor numa tela "DNS":

| Provedor | IPv4 |
|---|---|
| Cloudflare | `1.1.1.1`, `1.0.0.1` |
| Google | `8.8.8.8`, `8.8.4.4` |
| Quad9 (bloqueia sites maliciosos) | `9.9.9.9`, `149.112.112.112` |
| OpenDNS | `208.67.222.222`, `208.67.220.220` |
| Automático (do roteador) | — |

- Botão **"Testar velocidade"**: mede o tempo de resposta de cada provedor (algumas consultas DNS para domínios comuns, mediana em ms) e destaca o mais rápido. Fazer a consulta por UDP direto ao servidor (sem trocar o DNS do sistema para testar).
- Aplicar reaproveitando a `DnsServersAction` (que já guarda e restaura o DNS anterior).
- Consequências: as mesmas do `network.dns-cloudflare` (rede de empresa/escola).

---

## Fica de fora — só o que é placebo

| Item do .bat | Por que é placebo |
|---|---|
| `AllowAppDataCollection`, `DisableWindowsAdvertising`, `DisableMicrosoftConsumerExperience` | Chaves com nome errado; o Windows não lê |
| `EnableSuperfetch` | Ignorado no Windows 10/11 |
| `VisualFXSetting` em `Control Panel\Desktop` | Valor que não existe nessa chave |
| `SystemResponsiveness=0` | O Windows trata como 20 (o padrão); o app já usa 10 |
| `GPU Priority=8` | Já é o padrão |
| "Otimizar Energia" do .bat | Cria plano sem ativar; o resto é padrão (o app já tem o plano de verdade) |
| Menus AMD / Intel / NVIDIA | Idênticos entre si; tudo já coberto |
| "EcoQoS" | Mesma chave do Power Throttling (já existe) |
| `RequirePlatformSecurityFeatures=0` ao desligar o VBS | Só vale quando o VBS está ligado |
| `SystemRestorePointCreationFrequency=0` permanente | O motor já libera o limite na hora de criar o ponto |
| Serviços Xbox em Automático ("Fix Serviços Xbox") | Errado; `maintenance.restore-default-services` já volta ao padrão certo (Manual) |
| `UsoClient StartDownload`, `winget upgrade Microsoft.WindowsCamera` nos fixes | O Windows e a Store já fazem; a busca de drivers ficou como opção na K4 |
| Logo ASCII, cores, menus, `pause`/`cls` | Interface do .bat |
| Autoruns | Ferramenta externa; o módulo Inicialização já cobre |
