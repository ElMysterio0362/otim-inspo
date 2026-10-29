# Features para o NinjaOptimizer — extraídas do `iGust_Windows_Boost.bat`

> **Para quem vai implementar (sessão do NinjaOptimizer):** este arquivo é a fonte. Ele lista só as
> funcionalidades do .bat que **valem a pena** e **ainda não existem** no app, já traduzidas para a
> arquitetura atual (`Tweak` + `TweakAction` no `TweakCatalog`, `MaintenanceTask` na Manutenção, página
> Modo Gamer). O .bat **não** deve ser chamado nem copiado: tudo é reimplementado com
> `IRegistryAccess`, `IServiceManager` e `ICommandRunner`.
>
> Base analisada: `ElMysterio0362/ninjasoptimizer@ec295a4`. A análise completa (87 itens, com
> justificativas) está em `analises/iGust_Windows_Boost.md` no repositório `otim-inspo`.

## Regras gerais (valem para todas as features)

- Ler e guardar o valor atual **antes** de alterar (snapshot no journal), para desfazer item a item.
- Registrar tudo no log do app.
- Na interface, cada item mostra: toggle, nome, descrição clara, risco, se exige reinício e se é reversível.
- Risco **alto** → fora de qualquer perfil pronto + confirmação extra (o motor já faz isso).
- Respeitar as condições de cada item (SSD, notebook, fabricante da GPU, serviço existe).
- Um commit por etapa, com testes (aplicar → reverter → estado idêntico, com valor ausente e com valor existente).
- No fim, atualizar `README.md` e `docs/03-roteiro-de-etapas.md`.

## Ordem sugerida

| Etapa | Conteúdo |
|---|---|
| A | Condições de aplicabilidade (infra) + SysMain só em SSD |
| B | Processos, serviços e privacidade |
| C | Visual |
| D | Energia e periféricos |
| E | Jogos (prioridade por jogo) |
| F | Manutenção e reparos |
| G | Risco alto (opcional, só com aprovação do dono do projeto) |

---

## Etapa A — Condições de aplicabilidade (infraestrutura)

**Problema:** o `Tweak` não tem como dizer "só em SSD" ou "só em notebook". Hoje `services.sysmain` pode ser aplicado num HD, onde ele piora o desempenho.

**Proposta:**

- Novo tipo `TweakCondition` com `Check(contexto) → (bool aplicável, string motivo)`.
- `Tweak` ganha `IReadOnlyList<TweakCondition> Conditions` (vazio = sempre aplicável).
- O contexto ganha um resumo de hardware (de `WmiHardwareProvider` / `HardwareModels`, que o app já tem):
  - tipo do disco onde o Windows está instalado (`DiskMediaType`, reaproveitando `DiskTypeHeuristics`);
  - se é portátil: `Win32_SystemEnclosure.ChassisTypes` ∈ {8, 9, 10, 11, 12, 14, 18, 21, 30, 31, 32} **ou** existe `Win32_Battery`;
  - fabricante das GPUs pelo PCI Vendor ID: NVIDIA `10DE`, AMD `1002`, Intel `8086`;
  - build do Windows.
- Condições prontas: `SystemDriveIsSsd`, `IsDesktop`, `IsPortable`, `HasGpuVendor(vendor)`, `ServiceExists(nome)`, `MinWindowsBuild(n)`.
- Interface: item não aplicável aparece desabilitado com o motivo (ex.: "Só para SSD — o Windows está num HD").
- Perfis prontos pulam itens não aplicáveis sem avisar.
- Quando não dá para saber (ex.: tipo de disco desconhecido), o item fica **bloqueado** com o motivo "não foi possível confirmar".
- Mudar `services.sysmain`: `Conditions = [SystemDriveIsSsd]`.

**Testes:** condição falsa → o motor não aplica; perfil com item não aplicável → item pulado; tipo desconhecido → bloqueado.

---

## Etapa B — Processos, serviços e privacidade

### B1. `services.error-reporting` — Desativar o Relatório de Erros do Windows

| Campo | Valor |
|---|---|
| Descrição | Quando um programa trava, o Windows deixa de gerar o relatório e de enviá-lo à Microsoft. O aviso "o programa parou de funcionar" continua aparecendo. |
| Benefício | Evita o `WerFault.exe` gastando CPU e disco depois de um travamento; mais privacidade. |
| Categoria | `ProcessesServices` |
| Risco / Impacto | Baixo / Baixo |
| Reinício | Não |
| Perfis | Equilibrado, Máximo, Gamer |
| Ações | `ServiceStartupAction("WerSvc", Disabled)` · `LmDword(@"SOFTWARE\Microsoft\Windows\Windows Error Reporting", "Disabled", 1)` · `ServiceStartupAction("dmwappushservice", Disabled)` |
| Padrão do Windows | `WerSvc` Manual · `Disabled` ausente · `dmwappushservice` Manual |
| Observação | `dmwappushservice` é usado por PCs gerenciados por empresa (Intune/MDM). Citar isso na descrição. Se preferir, separar em um item próprio (`privacy.wap-push`). |

### B2. `privacy.app-launch-tracking` — Não registrar os apps que você abre

| Campo | Valor |
|---|---|
| Descrição | O Windows para de anotar quais programas você abre. A lista "Mais usados" do menu Iniciar fica vazia. |
| Benefício | Mais privacidade. Sem efeito no desempenho. |
| Categoria | `SystemPrivacy` |
| Risco / Impacto | Baixo / Baixo |
| Reinício | Não |
| Perfis | Todos |
| Ação | `CuDword(ExplorerAdvanced, "Start_TrackProgs", 0)` |
| Padrão do Windows | `1` ou ausente |

### B3. `gpu.nvidia-telemetry` — Desativar a telemetria da NVIDIA

| Campo | Valor |
|---|---|
| Descrição | Desliga o serviço "NVIDIA Telemetry Container", que envia dados de uso à NVIDIA. O driver e o painel da NVIDIA continuam funcionando. |
| Benefício | Um processo a menos em segundo plano e mais privacidade. |
| Categoria | `Gpu` |
| Risco / Impacto | Baixo / Baixo |
| Reinício | Não |
| Perfis | Máximo, Gamer |
| Ação | `ServiceStartupAction("NvTelemetryContainer", Disabled)` |
| Condições | `HasGpuVendor(NVIDIA)` **e** `ServiceExists("NvTelemetryContainer")` |
| Padrão | Automático |
| Observação | O serviço não existe nos drivers atuais (NVIDIA App). Sem a condição, o item apareceria e não faria nada. |

---

## Etapa C — Visual

### C1. `system.visual-best-performance` — Efeitos visuais: melhor desempenho

| Campo | Valor |
|---|---|
| Descrição | Equivale a "Ajustar para obter um melhor desempenho" nas Opções de desempenho do Windows: desliga sombras, animações de controles, mostrar o conteúdo da janela ao arrastar, fade de menus e dicas. A suavização das fontes (ClearType) não muda. |
| Benefício | Interface mais leve, principalmente com GPU integrada ou PC mais fraco. |
| Categoria | `SystemPrivacy` (onde já estão animações e transparência) |
| Risco / Impacto | Baixo / Baixo (Médio em PC fraco) |
| Reinício | Sair e entrar na conta |
| Perfis | Máximo |
| Ações | `CuDword(@"Software\Microsoft\Windows\CurrentVersion\Explorer\VisualEffects", "VisualFXSetting", 2)` · `new RegistryValueAction(RegHive.CurrentUser, @"Control Panel\Desktop", "UserPreferencesMask", new RegValue(RegKind.Binary, new byte[] { 0x90, 0x12, 0x03, 0x80, 0x10, 0x00, 0x00, 0x00 }))` |
| Padrão do Windows | `VisualFXSetting` = `0` ou ausente (o Windows escolhe). `UserPreferencesMask` varia por PC (no Win 10 costuma ser `9E 1E 07 80 12 00 00 00`), por isso o snapshot precisa guardar o binário original — `RegKind.Binary` já é suportado. |
| Não fazer | O .bat também grava `VisualFXSetting` em `Control Panel\Desktop`. Esse valor não existe lá (placebo). |
| Observação | Não conflita com `system.animations` (valores diferentes). Para aplicar sem sair da conta, dá para chamar `SystemParametersInfo(SPI_SETUIEFFECTS…)`, mas é opcional. |

---

## Etapa D — Energia e periféricos

### D1. `peripherals.usb-selective-suspend` — Não suspender dispositivos USB

| Campo | Valor |
|---|---|
| Descrição | O Windows para de desligar portas USB ociosas para economizar energia. Vale no plano de energia ativo, com o PC na tomada. |
| Benefício | Corrige mouse, teclado, headset ou controle USB que "dorme", demora a responder ou desconecta. |
| Categoria | `Cpu` (onde estão os ajustes de plano de energia) ou uma nova categoria de energia/dispositivos |
| Risco / Impacto | Baixo / Baixo |
| Reinício | Não |
| Perfis | Gamer |
| Ação | `new PowerSettingAction("2a737441-1930-4402-8d77-b2bebba308a3", "48e6b7a6-50f5-4782-a5d4-53bb8f07e226", 0, "suspensão seletiva de USB desligada")` (subgrupo USB / USB selective suspend) |
| Padrão do Windows | `1` (ligado) na tomada e na bateria |
| Condição | Só mexe no valor "na tomada" (AC), que é o que a `PowerSettingAction` já faz. **Não mexer no valor da bateria (DC)** — o .bat mexe nos dois, o que gasta bateria em notebook. |

### D2. `network.adapter-power-saving` — Não desligar a placa de rede para economizar

| Campo | Valor |
|---|---|
| Descrição | Desmarca "Permitir que o computador desligue este dispositivo para economizar energia" nas placas de rede físicas (Wi-Fi e cabo). A placa reinicia por alguns segundos ao aplicar. |
| Benefício | Corrige internet caindo ou demorando a voltar depois que o PC fica parado. |
| Categoria | `Network` |
| Risco / Impacto | Médio / Baixo |
| Reinício | Não (a placa reinicia sozinha) |
| Perfis | Nenhum |
| Condição | Em notebook, aviso extra: gasta mais bateria. |
| Ação nova | `NetAdapterPowerAction` |
| Captura | `Get-NetAdapter -Physical` → para cada placa, guardar `InterfaceGuid` (o nome muda, o GUID não) e o valor atual de `AllowComputerToTurnOffDevice` de `Get-NetAdapterPowerManagement`. |
| Aplicar | `Set-NetAdapterPowerManagement -Name <nome> -AllowComputerToTurnOffDevice Disabled` só nas placas que estão `Enabled`. |
| Reverter | Para cada GUID guardado que ainda existe, voltar o valor anterior. Placa que não existe mais → ignorar. |
| `IsApplied` | Todas as placas físicas com `AllowComputerToTurnOffDevice` ≠ `Enabled`. |
| Não fazer | O .bat usa `Disable-NetAdapterPowerManagement`, que desliga **tudo**, inclusive o Wake-on-LAN. Mexer só nessa opção. |
| Observação | Placas cujo driver não aceita a opção: pular e registrar no log. Alternativa por registro: `PnPCapabilities` na chave da classe de rede `{4d36e972-e325-11ce-bfc1-08002be10318}\00XX`, mas o cmdlet é mais claro e já valida o driver. |

---

## Etapa E — Jogos

### E1. `gaming.process-priority` — Prioridade de CPU por jogo

| Campo | Valor |
|---|---|
| Onde | Página **Modo Gamer**: um toggle por jogo (jogos detectados por `ProcessRules.IsGame` + `GameModeSettings.UserGames`). **Não** usar a lista fixa do .bat. |
| Descrição | Faz o Windows abrir sempre esse jogo com prioridade de CPU maior que a dos outros programas. Vale na próxima vez que o jogo abrir. |
| Benefício | Ajuda em jogos limitados pela CPU quando há coisas pesadas em segundo plano. Em jogo limitado pela placa de vídeo, quase nada muda. |
| Risco / Impacto | Médio / Baixo |
| Reinício | Não (vale ao abrir o jogo) |
| Perfis | Nenhum |
| Ação nova | `ProcessPriorityAction(exe, prioridade)` |
| Chave | `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\<exe>\PerfOptions`, valor `CpuPriorityClass` (DWORD) |
| Valores | `6` Acima do normal (**recomendado**, padrão do toggle) · `3` Alta (opção avançada, com aviso). **Nunca** `4` (Tempo real). Outros: `1` Ociosa, `5` Abaixo do normal, `2` Normal. |
| Padrão do Windows | Chave ausente |
| Captura | Se `PerfOptions` existia e o valor anterior de `CpuPriorityClass`. |
| Reverter | Volta o valor anterior ou apaga o valor. Se a ação criou `PerfOptions` e ela ficou vazia, apagar a chave — hoje `IRegistryAccess` só tem `DeleteValue`; adicionar `DeleteKeyIfEmpty` (ou aceitar a chave vazia, que é inofensiva). |
| Segurança | Validar o nome: só o nome do arquivo, terminado em `.exe`, sem `\` ou `/`. **Nunca** escrever nada fora de `PerfOptions` na chave IFEO (`Debugger`, `GlobalFlag` etc. são usados por malware). Não mexer em outros valores que já existam em `IFEO\<exe>`. |
| Avisos na interface | Prioridade Alta pode fazer áudio do Discord/OBS engasgar. Alguns anti-cheats ignoram ou rebaixam a prioridade. `javaw.exe` vale para **todos** os programas Java, não só o Minecraft. |

**Lista de executáveis do .bat** (só como sugestão para pré-preencher / detectar; já corrigida):

| Jogo | Executável |
|---|---|
| Fortnite | `FortniteClient-Win64-Shipping.exe` |
| GTA V | `GTA5.exe` |
| FiveM | `FiveM_b2372_GTAProcess.exe` (o número muda com a versão do jogo) |
| CS2 | `cs2.exe` |
| Minecraft (Java) | `javaw.exe` (afeta todo Java) |
| Valorant | `VALORANT-Win64-Shipping.exe` |
| League of Legends | `League of Legends.exe` (o .bat usa `LeagueClient.exe`, que é o launcher — errado) |
| Warzone / CoD | `cod.exe` |
| Apex Legends | `r5apex.exe` |
| Roblox | `RobloxPlayerBeta.exe` |
| God of War / Ragnarök | `GoW.exe`, `GoWRagnarok.exe` |
| MTA / GTA SA | `Multi Theft Auto.exe`, `gta_sa.exe` |
| Euro Truck Simulator 1/2 | `eurotrucks.exe`, `ets2.exe` |
| Rainbow Six Siege | `RainbowSix.exe` |
| Resident Evil 2 / 4 / Village | `re2.exe`, `re4.exe`, `re8.exe` |
| Free Fire (BlueStacks) | `HD-Player.exe` |
| Battlefield 4 / 2042 / 6 | `bf4.exe`, `BF2042.exe`, `BF6.exe` |
| The Last of Us 1 / 2 | `tlou-i.exe`, `tlou-ii.exe` |
| PUBG | `tslgame.exe` |
| Rocket League | `RocketLeague.exe` |
| Cyberpunk 2077 | `Cyberpunk2077.exe` |
| Terraria | `Terraria.exe` |
| Red Dead Redemption 2 | `RDR2.exe` |
| Rust | `RustClient.exe` |
| DayZ | `DayZ.exe` |
| Escape from Tarkov | `EscapeFromTarkov.exe` |
| Marvel Rivals | `MarvelRivals.exe` |
| Dead by Daylight | `DeadByDaylight-Win64-Shipping.exe` |
| Palworld | `Palworld-Win64-Shipping.exe` |
| Genshin Impact | `GenshinImpact.exe` |
| Street Fighter 6 | `StreetFighter6.exe` |
| Skyrim SE | `SkyrimSE.exe` |
| Outros do .bat | `ArenaBreakout.exe`, `BloodStrike.exe`, `Charles.exe`, `Chivalry2-Win64-Shipping.exe`, `CultOfTheLamb.exe`, `Cuphead.exe`, `DaysGone.exe`, `Deadlock.exe`, `ds.exe`, `DeathStranding2.exe`, `FC26.exe`, `FarmingSimulator2022.exe`, `FarmingSimulator2025.exe`, `GhostOfTsushima.exe`, `HLL.exe`, `MecchaChameleon.exe`, `PES2017.exe`…`PES2020.exe`, `eFootball.exe`, `Playtime_Multiplayer.exe`, `PoppyPlaytimeChapter4.exe`, `PoppyPlaytimeChapter5.exe`, `Poppy_Playtime.exe`, `ProjectPlaytime.exe`, `PointBlank.exe`, `REMATCH.exe`, `SnowRunner.exe`, `Subnautica.exe`, `TheIsleClient-Win64-Shipping.exe`, `ULTRAKILL.exe`, `Warface.exe`, `Warframe.x64.exe`, `crossfire.exe`, `ffxiv_dx11.exe`, `hollow_knight.exe`, `left4dead.exe`, `left4dead2.exe`, `mysummercar.exe` |

> Os nomes vieram do .bat e não foram conferidos um a um. Preferir sempre o executável que o Modo Gamer detectou rodando.

---

## Etapa F — Manutenção e reparos (`MaintenanceTask`)

São ações de "rodar agora": não têm estado para reverter. Cada uma diz antes o que vai fazer (`Note`), quanto demora (`Duration`) e se precisa de administrador.

### F1. `maintenance.repair-system-files` — Verificar e reparar arquivos do Windows
- **Faz:** `DISM.exe /Online /Cleanup-Image /RestoreHealth` e depois `sfc.exe /scannow`.
- **Admin:** sim · **Duração:** 10 a 30 minutos · **Precisa de internet** (o DISM baixa os arquivos do Windows Update).
- **Resultado:** ler o código de saída de cada um e resumir: "nenhum problema", "problemas corrigidos" ou "problemas que não puderam ser corrigidos" (apontar `%windir%\Logs\CBS\CBS.log`).
- **Armadilha:** a saída do `sfc` vem em UTF-16 — ler com `Encoding.Unicode` ou confiar só no código de saída.
- Diferente da tarefa `cleanup.components` que já existe (`StartComponentCleanup`): esta repara, aquela só limpa.

### F2. `maintenance.restart-explorer` — Reiniciar o Explorer
- **Faz:** encerra o `explorer.exe` da sessão atual e abre de novo. Corrige barra de tarefas, menu Iniciar ou área de trabalho travados.
- **Admin:** não · **Duração:** segundos.
- **Garantir** que o Explorer sempre volta, mesmo se algo falhar no meio (`try/finally`).
- **Armadilha:** se o app estiver elevado, iniciar o `explorer.exe` direto cria um Explorer como administrador. Iniciar sem elevação (pelo token do shell / processo não elevado).

### F3. `cleanup.recent-items` — Limpar a lista de itens recentes
- **Faz:** apaga os atalhos `*.lnk` **só no primeiro nível** de `%APPDATA%\Microsoft\Windows\Recent`.
- **Admin:** não · **Duração:** segundos · **Categoria:** `SystemPrivacy`.
- **Note:** "Os arquivos recentes somem do Acesso Rápido e do menu Iniciar. Seus arquivos não são apagados."
- **Não fazer:** o .bat usa `del /s`, que apaga também `AutomaticDestinations` — isso **remove as pastas fixadas no Acesso Rápido** e as listas de atalho dos programas. Não descer em subpastas.
- Pode reaproveitar `FolderCleanupTask`/`FolderCleaner` se ganhar a opção "sem subpastas, só `*.lnk`".

### F4. `maintenance.repair-windows-update` — Reparar o Windows Update
- **Faz:** para `wuauserv`, `bits` e `cryptsvc` → renomeia `%SystemRoot%\SoftwareDistribution` e `%SystemRoot%\System32\catroot2` para `<nome>.bak-AAAAMMDD-HHMMSS` → inicia os 3 serviços de novo.
- **Admin:** sim · **Duração:** 1 minuto · recomenda reiniciar depois.
- Usar data/hora no nome evita o erro do .bat quando já existe um `.old`.
- Se a renomeação falhar (pasta em uso), religar os serviços mesmo assim e informar.
- Não mudar o tipo de início dos serviços.
- **Note:** "O histórico de atualizações mostrado no Windows Update fica vazio. As atualizações instaladas não são afetadas."

### F5. `maintenance.repair-network` — Reparar a rede
- **Faz:** `netsh winsock reset`, `netsh int ip reset "<pasta de logs do app>\netsh-ip-reset.log"` e `ipconfig /flushdns`.
- **Admin:** sim · **Reinício:** obrigatório.
- **Note forte:** "Apaga IP fixo e DNS configurados manualmente e pode desinstalar filtros de VPN/antivírus antigos. Use só se a internet não funciona."
- Antes de rodar, salvar no log a configuração atual (`Get-NetIPConfiguration` / `ipconfig /all`) para o usuário poder digitar o IP fixo de novo.
- Se o journal tiver `network.dns-cloudflare` ou `network.nagle-off` aplicados, avisar que o reset pode desfazer esses ajustes por fora.
- **Não incluir** `ipconfig /release` + `/renew` como "melhorar ping" (placebo).

### F6. `maintenance.check-disk` — Verificar erros no disco do Windows
- **Faz:** `chkdsk C: /scan` (verificação online: não reinicia e não trava o disco). Usar a unidade do Windows, não `C:` fixo.
- **Admin:** sim · **Duração:** alguns minutos · só NTFS.
- **Códigos de saída:** `0` sem erros · `1` erros corrigidos · `2` limpeza feita · `3` erros que precisam de correção offline.
- Se der `3`, oferecer (com confirmação) `chkdsk <unidade> /spotfix`, que roda no próximo boot e é rápido.
- **Não fazer:** `chkdsk /f /r` como padrão — num HD grande pode deixar o PC inutilizável por horas.
- **Armadilha:** a saída do `chkdsk` usa a página de código OEM do console.

### F7. `maintenance.restart-audio` — Reiniciar o áudio do Windows
- **Faz:** para `Audiosrv` e depois `AudioEndpointBuilder`, inicia `AudioEndpointBuilder` e depois `Audiosrv`. Se algum estiver Desativado, volta para Automático (padrão do Windows).
- **Admin:** sim · **Duração:** segundos.
- **Note:** "O som some por alguns segundos." Corrige "nenhum dispositivo de saída de áudio instalado" e som que parou do nada.

### F8. `maintenance.restore-default-services` — Restaurar serviços essenciais ao padrão
- **Faz:** para cada serviço da tabela abaixo que estiver **Desativado**, volta ao tipo de início padrão do Windows. Não mexe nos que já estão Manual ou Automático. Registra no log cada mudança (de → para).
- **Admin:** sim · **Duração:** segundos.
- **Não mexer** em serviços que o próprio app desativou por um ajuste ativo no journal (para isso o usuário usa *Desfazer*).
- Substitui os "Fix Bluetooth / Wi-Fi / Modo avião / Serviços Xbox" do .bat, que forçavam tudo para Automático (errado).

| Serviço | Para que serve | Padrão do Windows |
|---|---|---|
| `bthserv` | Bluetooth | Manual |
| `WlanSvc` | Wi-Fi | Automático |
| `Dhcp` | Obter IP da rede | Automático |
| `Netman` | Conexões de rede | Manual |
| `NlaSvc` | Detecção de rede | Automático no Win 10, Manual no Win 11 (confirmar num Windows real) |
| `RmSvc` | Modo avião / rádios | Manual |
| `Audiosrv` | Áudio | Automático |
| `AudioEndpointBuilder` | Dispositivos de áudio | Automático |
| `BITS` | Downloads do Windows Update e da Store | Manual |
| `wuauserv` | Windows Update | Manual |
| `FrameServer` | Câmera | Manual |
| `XblAuthManager`, `XblGameSave`, `XboxNetApiSvc`, `XboxGipSvc` | Xbox / Game Pass / controles Xbox | Manual |

### F9. `maintenance.repair-store` — Reparar a Microsoft Store
- **Faz:**
  1. garante que `BITS`, `wuauserv`, `ClipSVC` e `InstallService` não estão Desativados (se estiverem, volta ao padrão);
  2. roda `wsreset.exe` (limpa o cache da Store; ela abre sozinha no fim);
  3. re-registra **só** `Microsoft.WindowsStore` e `Microsoft.DesktopAppInstaller` do usuário atual: `Get-AppxPackage <nome> | ForEach-Object { Add-AppxPackage -DisableDevelopmentMode -Register "$($_.InstallLocation)\AppXManifest.xml" }`.
- **Admin:** só para o passo 1 · **Duração:** 1 a 2 minutos.
- **Não fazer:** o .bat re-registra **todos** os pacotes de **todos** os usuários — demora muito, gera centenas de erros e não é necessário.

---

## Etapa G — Risco alto (opcional — só com aprovação explícita do dono)

### G1. `security.memory-integrity-off` — Desativar a Integridade de Memória (HVCI) e o VBS

| Campo | Valor |
|---|---|
| Descrição | Desliga a "Integridade de Memória" (Segurança do Windows → Segurança do dispositivo → Isolamento de núcleo) e a segurança baseada em virtualização. **Reduz a proteção do Windows contra drivers maliciosos.** |
| Benefício | Alguns % de FPS em CPUs mais antigas (antes de Intel 11ª geração / Ryzen 3000). Em CPUs novas o ganho é quase nulo. |
| Categoria | `Cpu` |
| Risco / Impacto | **Alto** / Médio em CPU antiga, Baixo em CPU nova |
| Reinício | **Sim** |
| Perfis | **Nenhum** + confirmação extra (diálogo vermelho do motor) |
| Ações | `LmDword(@"SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity", "Enabled", 0)` · `LmDword(@"SYSTEM\CurrentControlSet\Control\DeviceGuard", "EnableVirtualizationBasedSecurity", 0)` |
| Padrão do Windows | Ligado em instalação nova do Win 11; desligado em muitos PCs que vieram do Win 10 |
| Condição | Bloquear com motivo se o VBS estiver imposto por política (`HKLM\SOFTWARE\Policies\Microsoft\Windows\DeviceGuard`) ou travado por UEFI — aí o registro não tem efeito. Para mostrar o estado real, ler `Win32_DeviceGuard` (namespace `root\Microsoft\Windows\DeviceGuard`): `VirtualizationBasedSecurityStatus` e `SecurityServicesRunning`. |
| Não fazer | O .bat também desliga o Credential Guard (`Lsa\LsaCfgFlags=0`), `RequirePlatformSecurityFeatures=0` e roda `bcdedit /set hypervisorlaunchtype off`. **Não incluir:** só reduzem segurança, e o `bcdedit` quebra WSL2, Hyper-V, Windows Sandbox, Docker Desktop e emuladores Android. |
| Avisos | A Segurança do Windows vai mostrar um alerta amarelo. Algumas atualizações do Windows podem religar a Integridade de Memória — o item vai aparecer como "não aplicado" de novo. |

---

## Já existe no app — não duplicar

| Item do .bat | Id no NinjaOptimizer |
|---|---|
| Ponto de restauração e backup do registro | motor (`OptimizationEngine`, `WmiRestorePointService`) |
| MPO | `gpu.mpo-off` |
| Game DVR / captura | `gaming.gamedvr` |
| Painel e botão Xbox da Game Bar | `gaming.game-bar-shortcuts` |
| Modo de Jogo | `gaming.game-mode` |
| Otimizações de tela cheia | `gaming.fullscreen-optimizations` |
| Perfil "Games" do MMCSS | `gaming.scheduler-priority` |
| `SystemResponsiveness` | `network.throttling` (o .bat usa `0`, que o Windows trata como 20; o app usa `10`, correto) |
| Plano de energia Desempenho Máximo | `cpu.ultimate-power-plan` |
| `Win32PrioritySeparation` | `cpu.foreground-priority` |
| Power Throttling / "EcoQoS" | `cpu.power-throttling` |
| PCI Express ASPM | `gpu.pcie-power` |
| Transparência | `system.transparency` |
| Apps em segundo plano | `system.background-apps` |
| Telemetria (`AllowTelemetry`, DiagTrack) | `privacy.telemetry` |
| Bing na busca | `privacy.bing-search` |
| Spooler | `services.print-spooler` |
| Indexação (WSearch) | `services.search-indexer` |
| SysMain | `services.sysmain` (falta a condição de SSD → Etapa A) |
| Hibernação | `storage.hibernation-off` |
| Compressão de memória | `memory.compression-off` |
| Aceleração do mouse | `gaming.mouse-acceleration` |
| Temporários, cache do Windows Update, Lixeira, DNS | `cleanup.temp`, `cleanup.windows-update`, `cleanup.recycle-bin`, `maintenance.dns-flush` |
| Esvaziar a standby list | `memory.standby-list` |
| Autoruns | módulo Inicialização (`StartupManager`) |
| DNS Jumper | `network.dns-cloudflare` |

## Não implementar (placebo ou perigoso)

| Item do .bat | Motivo |
|---|---|
| `ipconfig /release` + `/renew` "para ping" | Placebo |
| Esvaziar working sets / modified page list | Mito; causa engasgos |
| Desligar Prefetch (`EnablePrefetcher=0`) / `EnableSuperfetch=0` | Mito / valor ignorado; deixa boot e apps mais lentos |
| `CortanaConsent`, `AllowCortana`, remover Cortana | Sem efeito no Win 11 |
| `AllowAppDataCollection`, `DisableWindowsAdvertising`, `DisableMicrosoftConsumerExperience` | Chaves com nome errado (não existem) |
| `DoNotConnectToWindowsUpdateInternetLocations=1` | Política de WSUS; pode quebrar a Store |
| Desativar `WbioSrvc` | Quebra o Windows Hello |
| Desativar serviços Xbox / remover apps Xbox | Sem ganho; quebra Game Pass e login Xbox |
| Debloater com curingas (`*music*`, `*phone*`…) | Remove Media Player, Spotify, Email/Calendário; sem volta |
| `AppCompat DisableEngine/DisablePCA/DisableInventory` + `PcaSvc` | Sem ganho; programas antigos podem parar de abrir |
| `LsaCfgFlags=0`, `RequirePlatformSecurityFeatures=0`, `hypervisorlaunchtype off` | Só reduzem segurança / quebram WSL2 e Hyper-V |
| Timer resolution / ISLC | Ferramenta externa; mito no Win 11 |
| "PC não desliga" (`shutdown /f /t 0`) | Fecha tudo sem salvar; não corrige nada |
| `taskkill /F` em Teams/Zoom/Discord/OBS | Perde trabalho sem perguntar |
| Forçar `Allow` em câmera/microfone e apagar políticas | Passa por cima da privacidade e de políticas da empresa |
| Menus "AMD / Intel / NVIDIA" | São idênticos entre si e já cobertos |
| `SearchboxTaskbarMode=0` | Preferência visual, não otimização (pode virar item depois, se quiserem) |
| Executáveis externos (`DnsJumper`, `EmptyStandbyList`, `ISLC`, `Autoruns`, `debloater.bat`) | Não verificáveis; o app já cobre o que eles fazem |
| `SystemRestorePointCreationFrequency=0` permanente | O motor já libera o limite só durante a criação do ponto |
| "Otimizar Energia" do .bat (`-duplicatescheme` sem ativar, `IdleDisable 0`) | Não faz nada (bug); coberto por `cpu.ultimate-power-plan` |
| `GPU Priority=8` | Já é o padrão do Windows |
| Re-registrar os apps Xbox ("Fix Xbox app") | Baixo valor |
| Câmera/Mic: desabilitar e reabilitar todos os dispositivos `Camera`/`Image`/`MEDIA`/`AudioEndpoint` | `MEDIA` pega áudio HDMI da GPU e placas de captura; adiar para um futuro "Reparar câmera" |
| Câmera/Mic: `pnputil /scan-devices`, `UsoClient StartScan/StartDownload`, `winget upgrade Microsoft.WindowsCamera` | O Windows já faz sozinho |
| Câmera/Mic: parar `PlugPlay`/`DeviceInstall`/`DsmSvc` e mudar o tipo de início | `PlugPlay` não pode ser parado; muda padrões. Só `FrameServer` faz sentido (futuro "Reparar câmera") |
| Re-registrar **todos** os pacotes Appx (Fix Loja / reverter debloater) | Substituído pela F9, que re-registra só a Store |
| `chkdsk C: /f /r` | Substituído pela F6 (`/scan` + `/spotfix`) |
| Botão "Reiniciar PC" | Só interface (e no .bat nem funciona) |
| Logo ASCII, cores, menus | Só interface |
