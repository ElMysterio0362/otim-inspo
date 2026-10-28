# Engenharia reversa — `iGust_Windows_Boost.bat`

Arquivo analisado: `iGust_Windows_Boost.bat` (2.447 linhas, 109 KB, UTF-8, CRLF).
Lido inteiro como texto. **Nunca foi executado.**
Comparado com o catálogo do NinjaOptimizer em `ElMysterio0362/ninjasoptimizer@ec295a4`.

---

## 1. Segurança

### Veredito: **não é malicioso**, mas tem itens perigosos que não devem ser copiados

| Verificação pedida | Resultado |
|---|---|
| Código ofuscado / codificado (`-enc`, base64, comandos montados por variáveis) | **Não encontrado.** Os únicos caracteres de controle são 54 `ESC` usados para cores ANSI no terminal. As variáveis (`%m%`, `%w%`…) só guardam cores. |
| Downloads / URLs (`curl`, `bitsadmin`, `certutil`, `Invoke-WebRequest`) | **Não encontrado.** Nenhuma URL no arquivo. Única ação de rede: `winget upgrade --id Microsoft.WindowsCamera` (loja oficial) dentro do "Fix Câmera". |
| Execução de arquivos externos | ⚠️ **Sim, 5 executáveis que não vieram junto** e que não pude verificar: `DnsJumper.exe`, `EmptyStandbyList.exe`, `ISLC v1.0.4.6\…ISLC.exe`, `Autoruns.exe`, `debloater.bat` (todos em `%~dp0`, a pasta do script). Os nomes batem com ferramentas conhecidas (Sordum, wj32, Wagnardsoft, Sysinternals), mas qualquer arquivo com esse nome na pasta seria executado como administrador. |
| Desativar Defender / Firewall | **Não encontrado.** |
| Desativar Windows Update de forma permanente | Não diretamente. ⚠️ A opção "Telemetria" grava a política `DoNotConnectToWindowsUpdateInternetLocations=1`, que **não é telemetria**: é uma política de WSUS e a Microsoft avisa que pode quebrar a Microsoft Store. Está com o nome errado no menu. |
| Arquivo hosts, tarefas agendadas, criação de usuários, permissões (`icacls`/`takeown`) | **Não encontrado.** |
| Comandos destrutivos sem restrição de caminho | `del`/`rd` estão sempre limitados a `%windir%\Temp`, `%temp%`, `Recent` e `SoftwareDistribution\Download` — OK. ⚠️ Mas há: remoção em massa de apps com curingas (`*music*`, `*phone*`, `*Xbox*`…, inclusive pacotes provisionados), `shutdown /f /t 0` (desliga na hora sem salvar nada), `taskkill /F` em Teams/Zoom/Discord/OBS sem perguntar e `chkdsk C: /f /r`. |
| Redução de segurança | ⚠️ **Sim:** duas opções desligam VBS, HVCI (Integridade de Memória), Credential Guard (`LsaCfgFlags=0`) e o hipervisor (`bcdedit /set hypervisorlaunchtype off`). |
| Privacidade | ⚠️ "Fix Câmera" e "Fix Microfone" gravam `Allow` em `HKLM` (todos os usuários) e apagam políticas que bloqueiam câmera e microfone — passam por cima da escolha do usuário e de políticas da empresa/escola. |

Como nada indica intenção maliciosa, a análise continuou. **Nada foi implementado** — aguardando aprovação.

### Bugs do próprio .bat (bom saber antes de "confiar" no que ele promete)

1. **Menus AMD / NVIDIA / Intel:** `reg add … /v GPU Priority …` sem aspas (linhas 299–302, 334–337, 364–367). O `reg` recusa a linha, então `GPU Priority`, `Scheduling Category` e `SFIO Priority` **nunca são gravados** por esses menus. Fora isso, os 3 menus são **idênticos** (o único item de fabricante é o serviço de telemetria da NVIDIA).
2. **"Otimizar Energia" não faz nada útil:** `-duplicatescheme` cria uma cópia do Desempenho Máximo mas **não a ativa**; `IdleDisable 0` já é o padrão do Windows; `setactive SCHEME_CURRENT` reativa o plano que já estava. A cada execução sobra mais um plano duplicado na lista.
3. **`SystemResponsiveness=0` é placebo:** o MMCSS trata valores abaixo de 10 como 20 (o padrão). O catálogo já usa 10, que é o menor valor que vale.
4. **Opção 34 "Reiniciar PC"** vai para `:opcao29` (Power Throttling). O PC nunca reinicia por ela.
5. Opção 69 de jogos usa `goto exit`, rótulo que não existe.
6. Dois rótulos `:opcao1`: funciona por acaso (o `goto` procura pra frente a partir de onde está).
7. "Fix Câmera" e "Fix Microfone" escrevem em `"%LOG%"`, variável que nunca é definida. O "Fix Microfone" termina com `exit /b 0` e fecha o script.
8. League of Legends prioriza `LeagueClient.exe` (o launcher), não `League of Legends.exe` (o jogo).
9. Prioridade por jogo **não tem opção de reverter** no script.
10. "Fix Serviços Xbox" põe os serviços em **Automático** (`Start=2`); o padrão do Windows é Manual (`3`).
11. "Desat. EcoQoS" grava exatamente a mesma chave do Power Throttling (o próprio script admite).
12. Três chaves de "telemetria" não existem com esse nome no Windows: `AllowAppDataCollection`, `DisableWindowsAdvertising` (a real é `AdvertisingInfo\DisabledByGroupPolicy`) e `DisableMicrosoftConsumerExperience` (a real é `CloudContent\DisableWindowsConsumerFeatures`, e só vale no Enterprise/Education).

---

## 2. Análise comando a comando

Legenda de status:
**✅ Já existe** (não duplicar) · **🆕 Útil** (candidato) · **🟡 Placebo/mito** · **⛔ Perigoso** · **➖ Ignorado** (interface/ferramenta externa)

### 2.1 Menu principal

| # | Comando original | O que faz de verdade | Categoria | Risco | Dificuldade | Impacto | Reversível / padrão Windows | Compatibilidade | Status |
|---|---|---|---|---|---|---|---|---|---|
| 1 | `reg add …SystemRestore /v SystemRestorePointCreationFrequency /d 0` + `Checkpoint-Computer` | Tira o limite de 1 ponto por 24 h **para sempre** e cria um ponto de restauração | manutenção | Baixo | — | — | Sim / valor ausente (= 1440 min) | Win 10/11 com Proteção do Sistema ligada | ✅ Já existe no motor (`WmiRestorePointService`), e melhor: libera o limite só durante a criação |
| 2 | `reg export` de 12 chaves para `.\Backup` | Backup das chaves que o script mexe | manutenção | Baixo | — | — | — | Todas | ✅ Já existe (backup `.reg` automático antes de cada aplicação) |
| 3 | `ipconfig /flushdns` | Limpa o cache de DNS | rede | Baixo | Fácil | Leve (não muda ping) | Não precisa | Todas | ✅ Já existe (`maintenance.dns-flush`) |
| 4 | `ipconfig /release` + `/renew` | Derruba a conexão e pede IP novo ao roteador | rede | Baixo | Fácil | 🟡 **Placebo para ping** — só ajuda com conflito de IP | Não precisa | Todas; quebra IP fixo? não, mas derruba a rede por segundos | 🟡 Ignorar como "otimização" (entra só no reparo de rede, item 89) |
| 5 | `start DnsJumper.exe` | Abre ferramenta externa de troca de DNS | rede | Médio (binário não verificado) | — | — | — | — | ➖ Ignorar. O app já tem `network.dns-cloudflare` com reversão exata |
| 6 | `EmptyStandbyList.exe standbylist` | Esvazia a lista de espera (cache) da RAM | memória | Baixo | — | Leve | Não precisa | Todas | ✅ Já existe (`memory.standby-list`, via `ntdll`, sem binário externo) |
| 7 | `EmptyStandbyList.exe workingsets` / `modifiedpagelist` | Força todos os programas a devolver RAM e grava páginas modificadas no disco | memória | Médio | — | 🟡 **Mito/prejudicial**: o Windows recarrega tudo do disco logo depois, causando engasgos | Não precisa | Todas | 🟡 Ignorar |
| 8 | `start Autoruns.exe` | Abre o Autoruns (Sysinternals) | processos | Médio (binário não verificado) | — | — | — | — | ➖ Ignorar. O app já tem o módulo Inicialização (`StartupManager`) |

### 2.2 Menu "Otimizar AMD / Intel / NVIDIA" (os três são iguais)

| # | Comando original | O que faz de verdade | Categoria | Risco | Dificuldade | Impacto | Reversível / padrão Windows | Compatibilidade | Status |
|---|---|---|---|---|---|---|---|---|---|
| 9 | `HKLM\…\Dwm /v OverlayTestMode /d 5` | Desliga o MPO (sobreposições multiplano) | GPU | Médio | — | Leve; só corrige tela piscando | Sim / ausente | Win 10/11; ignorado em builds recentes do Win 11 | ✅ Já existe (`gpu.mpo-off`, fora dos perfis — correto) |
| 10 | `GameBar /v ShowStartupPanel /d 0` | Esconde o painel de boas-vindas da Game Bar | jogos | Baixo | — | Nenhum (visual) | Sim / ausente | Win 10/11 | ✅ Já existe (`gaming.game-bar-shortcuts`) |
| 11 | `GameBar /v AllowAutoGameMode /d 1`, `AutoGameModeEnabled /d 1` | **Liga** o Modo de Jogo (o menu diz "desativando Game Bar") | jogos | Baixo | — | 🟡 Já vem ligado desde o Win 10 1703 | Sim / ausente (= ligado) | Win 10/11 | ✅ Já existe (`gaming.game-mode`) |
| 12 | `GameConfigStore /v GameDVR_Enabled /d 0` + política `AllowGameDVR=0` | Desliga a gravação em segundo plano | jogos | Baixo | — | Moderado em GPUs fracas | Sim / `1` ou ausente | Win 10/11 | ✅ Já existe (`gaming.gamedvr`) |
| 13 | `GameDVR_FSEBehaviorMode=2`, `GameDVR_HonorUserFSEBehaviorMode=1` | Desliga otimizações de tela cheia (global) | jogos | Médio | — | Leve; varia por jogo | Sim / ausente | Win 10/11, DirectX 9–11 | ✅ Já existe (`gaming.fullscreen-optimizations`, com as 4 chaves) |
| 14 | `SystemProfile /v SystemResponsiveness /d 0` | Tenta reservar 0% de CPU para tarefas de fundo | CPU | Baixo | — | 🟡 **Placebo**: abaixo de 10 o Windows usa 20 | Sim / `20` | Todas | ✅ Já existe com o valor correto (`network.throttling` usa 10) |
| 15 | `Tasks\Games`: `GPU Priority=8`, `Priority=6`, `Scheduling Category=High`, `SFIO Priority=High` | Sobe a prioridade do perfil "Games" do MMCSS | jogos | Baixo | — | Leve (só vale para jogos que se registram no MMCSS). `GPU Priority=8` **já é o padrão** | Sim / `8`, `2`, `Medium`, `Normal` | Todas | ✅ Já existe (`gaming.scheduler-priority`). Obs.: no .bat esses valores falham por falta de aspas |
| 16 | `sc stop/config NvTelemetryContainer start= disabled` | Desliga o contêiner de telemetria da NVIDIA | GPU / privacidade | Baixo | Fácil | Leve (um processo a menos) | Sim / Automático | **Só NVIDIA** com driver antigo (o serviço não existe nos drivers atuais / NVIDIA App) | 🆕 Útil, com condição "GPU NVIDIA + serviço existe" |
| 17 | Reverter: `reg delete`/`reg add` dos itens acima, `NvTelemetryContainer start= demand` | Volta aos padrões (mas põe a telemetria em Manual, não no Automático original) | — | — | — | — | — | — | ✅ O motor do app já reverte pelo snapshot, com o valor exato anterior |

### 2.3 Menu "Otimizar Windows"

| # | Comando original | O que faz de verdade | Categoria | Risco | Dificuldade | Impacto | Reversível / padrão Windows | Compatibilidade | Status |
|---|---|---|---|---|---|---|---|---|---|
| 18 | `powercfg -duplicatescheme e9a42b02…` + `IdleDisable 0` + `setactive SCHEME_CURRENT` + abre `powercfg.cpl` | Cria uma cópia do Desempenho Máximo **sem ativar**; o resto é o padrão | CPU | Baixo | — | 🟡 **Nenhum** (bug, ver seção 1) | — | Todas | ✅ Já existe e funciona de verdade (`cpu.ultimate-power-plan`) |
| 19 | `VisualEffects /v VisualFXSetting /d 2` + `Control Panel\Desktop /v UserPreferencesMask /d 9012038010000000` | "Ajustar para melhor desempenho": desliga sombras, animações, arrastar janela com conteúdo, miniaturas animadas etc. | visual | Baixo | Média (valor binário; precisa ler e guardar o original) | Leve; moderado em PC fraco / GPU integrada | Sim / `VisualFXSetting=0` (deixar o Windows escolher), máscara `9E1E078012000000` (varia) | Todas; vale após sair e entrar | 🆕 Útil (complementa `system.animations`) |
| 20 | `Control Panel\Desktop /v VisualFXSetting /d 2` | Valor que não existe nessa chave | visual | — | — | 🟡 Placebo | — | — | 🟡 Ignorar |
| 21 | `Personalize /v EnableTransparency /d 0` | Desliga transparência | visual | Baixo | — | Leve | Sim / `1` | Win 10/11 | ✅ Já existe (`system.transparency`) |
| 22 | `BackgroundAccessApplications /v GlobalUserDisabled /d 1` | Bloqueia apps da Store em segundo plano | processos | Médio | — | Moderado | Sim / ausente | Win 10 (no Win 11 o valor é ignorado em builds novos) | ✅ Já existe (`system.background-apps`) |
| 23 | `Explorer\Advanced /v Start_TrackProgs /d 0` | Para de registrar quais apps você abre (lista "mais usados" do Iniciar) | privacidade | Baixo | Fácil | Leve (privacidade, não desempenho) | Sim / `1` ou ausente | Win 10/11 | 🆕 Útil (privacidade) |
| 24 | `WerSvc` desativado + `Windows Error Reporting /v Disabled /d 1` (opções 4 e 7) | Desliga o Relatório de Erros do Windows | privacidade / processos | Baixo | Fácil | Leve (evita o `WerFault` ao travar um programa) | Sim / serviço Manual, valor ausente | Todas | 🆕 Útil |
| 25 | `DiagTrack` desativado | Serviço de telemetria | privacidade | Baixo | — | Leve | Sim / Automático | Todas | ✅ Já existe (`privacy.telemetry`) |
| 26 | `dmwappushservice` desativado | Roteador de mensagens WAP usado pela telemetria/MDM | privacidade | Baixo (Médio em PC gerenciado por empresa — Intune usa) | Fácil | Leve | Sim / Manual | Todas | 🆕 Útil, junto do item 24, com aviso para PC de empresa |
| 27 | `WbioSrvc` desativado | Serviço de Biometria | processos | **Alto** | — | 🟡 Nenhum ganho | Sim / Manual | — | ⛔ **Quebra o Windows Hello** (digital e rosto). Ignorar |
| 28 | `Spooler` desativado | Spooler de impressão | processos | Médio | — | Leve | Sim / Automático | Todas | ✅ Já existe (`services.print-spooler`, fora dos perfis) |
| 29 | Game Bar: `UseNexusForGameBarEnabled=0`, `AppCaptureEnabled=0` | Botão Xbox não abre a Game Bar; desliga captura | jogos | Baixo | — | Leve | Sim | Win 10/11 | ✅ Já existe (`gaming.game-bar-shortcuts`, `gaming.gamedvr`) |
| 30 | `Get-AppxPackage *Xbox* \| Remove-AppxPackage` + `Remove-AppxProvisionedPackage` | **Desinstala** todos os apps com "Xbox" no nome, inclusive para usuários futuros | jogos | **Alto** | — | 🟡 Nenhum ganho de FPS | **Não** (o "reverter" só re-registra a Game Bar; os provisionados somem) | — | ⛔ Quebra Game Pass, login Xbox em jogos (Minecraft, Forza, Sea of Thieves…) e a Game Bar. Ignorar |
| 31 | `XblAuthManager`, `XblGameSave`, `XboxNetApiSvc`, `XboxGipSvc` desativados | Desliga serviços Xbox | jogos | Médio | Fácil | 🟡 Nenhum (são Manuais: só rodam quando algo usa) | Sim / Manual | — | 🟡 Ignorar. `XboxGipSvc` desligado pode quebrar acessórios/controles Xbox |
| 32 | `AllowTelemetry=0` (política) | Telemetria mínima | privacidade | Baixo | — | Leve | Sim / ausente | Todas | ✅ Já existe (`privacy.telemetry`) |
| 33 | `AllowAppDataCollection`, `DisableWindowsAdvertising`, `DisableMicrosoftConsumerExperience` | Chaves com nome errado | privacidade | — | — | 🟡 **Placebo** (não existem) | — | — | 🟡 Ignorar. O efeito real já é coberto por `privacy.ads-tips` e `privacy.advertising-id` |
| 34 | `WindowsUpdate /v DoNotConnectToWindowsUpdateInternetLocations /d 1` | Política de WSUS; não é telemetria | outra | **Alto** | — | Nenhum | Sim / ausente | — | ⛔ Pode quebrar a Microsoft Store e atualizações em PCs com WSUS. Ignorar |
| 35 | `powercfg -h off` | Desliga hibernação e apaga o `hiberfil.sys` | disco | Médio | — | Leve (libera GB de disco) | Sim / ligado | Todas | ✅ Já existe (`storage.hibernation-off`) |
| 36 | `Disable-MMAgent -MemoryCompression` | Desliga compressão de memória | memória | Médio | — | Leve; o script indica para "mais de 8 GB", o catálogo exige 32 GB (mais correto) | Sim / ligado | Win 10/11 | ✅ Já existe (`memory.compression-off`) |
| 37 | `WSearch` desativado | Desliga a indexação | disco | Médio | — | Moderado em HDD, leve em SSD | Sim / Automático (atraso) | Todas | ✅ Já existe (`services.search-indexer`) |
| 38 | `BingSearchEnabled=0`, `DisableSearchBoxSuggestions=1` | Tira a web da busca | privacidade | Baixo | — | Leve | Sim | Win 10/11 | ✅ Já existe (`privacy.bing-search`) |
| 39 | `CortanaConsent=0` | Consentimento da Cortana | privacidade | Baixo | — | 🟡 Nenhum no Win 11 (Cortana foi descontinuada) | Sim | Win 10 | 🟡 Ignorar |
| 40 | `SearchboxTaskbarMode=0` | Esconde a caixa/ícone de pesquisa da barra de tarefas | visual | Baixo | Fácil | Nenhum (preferência visual) | Sim / `1`–`3` conforme a versão | Win 10/11 | 🟡 Preferência, não otimização. Ignorar (dá pra fazer depois, se quiser) |
| 41 | `AllowCortana=0` + remover app Cortana | Desliga/desinstala a Cortana | privacidade | Baixo | — | 🟡 Nenhum no Win 11 | Remoção não é reversível pelo script | Win 10 | 🟡 Ignorar |
| 42 | `PrefetchParameters /v EnablePrefetcher /d 0` | Desliga o Prefetch (carregamento antecipado de apps e boot) | disco | Médio | — | 🟡 **Mito**: o script diz "indicado para HDD", mas é no HDD que o Prefetch mais ajuda; desligar deixa o boot e a abertura de apps mais lentos | Sim / `3` | Todas | 🟡 Ignorar |
| 43 | `EnableSuperfetch=0` | Valor que o Windows 10/11 ignora | memória | — | — | 🟡 Placebo | — | — | 🟡 Ignorar |
| 44 | `SysMain` desativado | Desliga o Superfetch | memória / disco | Médio | — | Leve em SSD; **prejudicial em HDD** | Sim / Automático | Deveria ser só SSD | ✅ Já existe (`services.sysmain`), **mas sem a condição "só em SSD"** → melhoria proposta (seção 4, etapa A) |
| 45 | Opção 15: mesmo bloco dos itens 14 e 15 | — | — | — | — | — | — | — | ✅ Já existe |
| 46 | `PriorityControl /v Win32PrioritySeparation /d 38` | Fatias curtas e reforço maior ao programa em primeiro plano | CPU | Baixo | — | Leve | Sim / `2` | Todas | ✅ Já existe (`cpu.foreground-priority`, mesmo valor `0x26`) |
| 47 | `DeviceGuard /v EnableVirtualizationBasedSecurity /d 0` + `HypervisorEnforcedCodeIntegrity /v Enabled /d 0` (opções 17 e 23) | Desliga VBS e Integridade de Memória (HVCI) | CPU / segurança | **Alto** | Média | **Real**: alguns % de FPS em CPUs mais antigas (pré-Intel 11ª / pré-Ryzen 3000), quase nada nas novas | Sim / Win 11 instalação nova: ligado; upgrades: desligado | Win 10/11; exige reinício | 🆕 / ⛔ Candidato **só com sua aprovação explícita** (risco alto, fora dos perfis, confirmação extra) |
| 48 | `Lsa /v LsaCfgFlags /d 0` | Desliga o Credential Guard (proteção das senhas na memória) | segurança | **Alto** | — | Nenhum ganho de desempenho | Sim / ausente | — | ⛔ Ignorar (só reduz segurança) |
| 49 | `bcdedit /set hypervisorlaunchtype off` | Não carrega o hipervisor no boot | CPU / segurança | **Alto** | — | Igual ao item 47 (é o que desliga o VBS de fato) | Sim / `Auto` ou ausente | Quebra **WSL2, Hyper-V, Windows Sandbox, Docker Desktop** e emuladores Android baseados em Hyper-V | ⛔ Ignorar como item separado |
| 50 | `DeviceGuard /v RequirePlatformSecurityFeatures /d 0` | Tira a exigência de DMA protection/Secure Boot do VBS | segurança | Alto | — | Nenhum | Sim | — | ⛔ Ignorar |
| 51 | Debloater: 19 × `Get-AppxPackage *nome* \| Remove-AppxPackage` | Desinstala apps com curingas (`*phone*`, `*music*`, `*maps*`, `*news*`, `*calendar*`, `*onedrive*`…) | processos | **Alto** | — | Leve (apps parados não gastam nada) | **Não** — o "reverter" re-registra *todos* os pacotes do sistema e não traz de volta o que foi removido | — | ⛔ Curingas pegam o que não devem: `*music*` remove o **Media Player** do Win 11 e o **Spotify** da Store; `*calendar*`/`*windowscommunicationsapps*` remove Email e Calendário; `*phone*` remove o Link to Phone. Ignorar |
| 52 | Opção 19: Modo de Jogo | — | jogos | — | — | — | — | — | ✅ Já existe (`gaming.game-mode`) |
| 53 | Opção 20: FSO | — | jogos | — | — | — | — | — | ✅ Já existe (`gaming.fullscreen-optimizations`) |
| 54 | `Control Panel\Mouse`: `MouseSpeed=0`, `MouseThreshold1=0`, `MouseThreshold2=0` | Desliga aceleração do mouse | jogos | Baixo | — | Moderado em FPS competitivo | Sim / `1`, `6`, `10` | Todas | ✅ Já existe (`gaming.mouse-acceleration`) |
| 55 | "Timer Resolution": abre `ISLC.exe` | Ferramenta externa (limpa standby + timer) | memória | Médio (binário não verificado) | — | 🟡 Leve; o timer de 0,5 ms é mito no Win 11 (cada processo tem o seu) | — | — | ➖ Ignorar (a limpeza da standby já existe) |
| 56 | `PcaSvc` desativado + `AppCompat`: `DisablePCA=1`, `DisableEngine=1`, `DisableInventory=1` | Desliga o Assistente de Compatibilidade **e o motor de compatibilidade (shims)** | outra | **Alto** | — | 🟡 Nenhum ganho mensurável | Sim / ausente; serviço Manual (11) ou Automático (10) | — | ⛔ `DisableEngine` pode fazer jogos e programas antigos pararem de abrir. Ignorar |
| 57 | `taskkill /f /im explorer.exe` / `start explorer.exe` | Fecha / abre o Explorer | manutenção | Baixo | Fácil | Nenhum (útil para destravar a barra de tarefas) | — | Todas | 🆕 Útil como ação "Reiniciar o Explorer" (junta as duas, nunca deixa o Explorer fechado) |
| 58 | `del %windir%\Temp`, `del %temp%` | Limpa temporários | disco | Baixo | — | Leve | Não (apaga) | Todas | ✅ Já existe (`cleanup.temp`, e com idade mínima) |
| 59 | `del %APPDATA%\Microsoft\Windows\Recent\*.*` | Apaga a lista de "itens recentes" | privacidade | Baixo | Fácil | Nenhum (privacidade) | Não | Todas | 🆕 Útil como limpeza de privacidade (avisando que o Acesso Rápido perde os recentes) |
| 60 | `net stop wuauserv` + `del SoftwareDistribution\Download` | Limpa o cache de downloads do Windows Update | disco | Baixo | — | Leve | Não | Todas | ✅ Já existe (`cleanup.windows-update`) |
| 61 | `Clear-RecycleBin -Force` | Esvazia a Lixeira sem perguntar | disco | Médio | — | Leve | **Não** | Todas | ✅ Já existe (`cleanup.recycle-bin`, com aviso) |
| 62 | `DISM /Online /Cleanup-Image /RestoreHealth` + `sfc /scannow` | Repara a imagem do Windows e os arquivos de sistema | manutenção | Baixo | Fácil | Nenhum em desempenho; corrige erros | Não precisa | Todas; DISM precisa de internet | 🆕 Útil (o app só tem `StartComponentCleanup`) |
| 63 | `PowerThrottling /v PowerThrottlingOff /d 1` | Desliga a limitação de energia por processo | CPU | Baixo | — | Leve | Sim / ausente | Win 10 1709+ | ✅ Já existe (`cpu.power-throttling`) |
| 64 | "EcoQoS": mesma chave do item 63 | — | CPU | — | — | 🟡 Duplicado | — | — | 🟡 Ignorar |
| 65 | `powercfg SUB_USB USBSELECTIVE 0` (AC e DC) | Desliga a suspensão seletiva de USB | periféricos / energia | Baixo | Fácil (`PowerSettingAction` já existe) | Leve; corrige mouse/headset USB que "dorme" ou desconecta | Sim / ligado (`1`) em AC e DC | Todas. **No notebook, só na tomada (AC)** — na bateria gasta mais | 🆕 Útil (aplicar só AC; DC só em desktop) |
| 66 | `Disable-NetAdapterPowerManagement` em todas as placas físicas | Impede o Windows de desligar a placa de rede para economizar | rede | Médio | Média (precisa ler e guardar as opções de cada placa) | Leve; corrige Wi-Fi/Ethernet caindo depois de ocioso | Sim / ligado (depende do driver) | Todas; **desliga também o Wake-on-LAN**. No notebook gasta mais bateria | 🆕 Útil, com aviso de notebook |
| 67 | `powercfg SUB_PCIEXPRESS ASPM 0` (AC e DC) | Desliga economia do PCI Express | GPU | Baixo | — | Leve | Sim / Equilibrado: economia moderada | Todas; DC só em desktop | ✅ Já existe (`gpu.pcie-power`, só AC — mais correto para notebook) |
| 68 | `shutdown /r /t 0` | Reinicia na hora | — | — | — | — | — | — | ➖ Ignorar (no .bat nem funciona, ver bug 4) |

### 2.4 Menu "Otimizar Jogos"

| # | Comando original | O que faz de verdade | Categoria | Risco | Dificuldade | Impacto | Reversível / padrão Windows | Compatibilidade | Status |
|---|---|---|---|---|---|---|---|---|---|
| 69 | 85 × `IFEO\<jogo>.exe\PerfOptions /v CpuPriorityClass /d 3` (68 jogos) | Faz o Windows abrir aquele `.exe` sempre com prioridade **Alta** de CPU | jogos | Médio | Média | Leve: só ajuda em jogo limitado por CPU com coisas pesadas em segundo plano; em jogo limitado por GPU, nada. Prioridade Alta pode fazer áudio do Discord/OBS engasgar | Sim / chave ausente (o .bat **não** reverte) | Todas. Alguns anti-cheats ignoram ou rebaixam a prioridade. `javaw.exe` afeta todos os programas Java, não só o Minecraft | 🆕 Útil, mas **sem lista fixa**: usar os jogos que o Modo Gamer já detecta + os que o usuário adicionar, um toggle por jogo, reversível |

### 2.5 Menu "Fix de Erros"

| # | Comando original | O que faz de verdade | Categoria | Risco | Dificuldade | Impacto | Reversível / padrão Windows | Compatibilidade | Status |
|---|---|---|---|---|---|---|---|---|---|
| 70 | Bluetooth: `bthserv start= auto` + start | Força o serviço Bluetooth para Automático | manutenção | Baixo | Fácil | Corrige se alguém desativou | Sim / **Manual** | Com Bluetooth | 🆕 Útil **reimaginado**: "Restaurar serviços essenciais ao padrão" (volta ao padrão do Windows só o que estiver Desativado, sem forçar Automático) |
| 71 | Wi-Fi: `WlanSvc`, `Dhcp`, `NlaSvc`, `Netman` → auto + `netsh winsock reset` + `netsh int ip reset` | Religa serviços de rede e reseta a pilha TCP/IP | rede | Médio | Fácil | Corrige internet quebrada por programas/VPNs | O reset **não** é reversível: apaga IP fixo, DNS manual e ajustes TCP (inclusive os do app) | Todas; exige reinício | 🆕 Útil como "Reparar rede", com aviso claro |
| 72 | Áudio: `Audiosrv`, `AudioEndpointBuilder` → auto + reinício | Reinicia os serviços de áudio | manutenção | Baixo | Fácil | Corrige "nenhum dispositivo de áudio" | Sim / Automático (já é o padrão) | Todas | 🆕 Útil ("Reiniciar o áudio do Windows") |
| 73 | Loja: `BITS` → auto + re-registrar **todos** os pacotes Appx de todos os usuários | Tenta consertar a Store | manutenção | Médio | Média | Às vezes resolve, demora e gera centenas de erros | BITS: padrão é Manual | Win 10/11 | 🆕 Útil **reimaginado**: `wsreset` + re-registrar só a Store e o App Installer |
| 74 | Modo avião: `RmSvc`, `WlanSvc` → auto | Religa serviço de rádio | manutenção | Baixo | Fácil | Corrige modo avião travado | `RmSvc` padrão Manual | Notebooks | 🆕 Entra no item 70 |
| 75 | "PC não desliga": `powercfg -h on` + `shutdown /f /t 0` | Religa hibernação e **desliga na hora, fechando tudo sem salvar** | manutenção | **Alto** | — | Não corrige nada (religar hibernação até piora o "não desliga" com Inicialização Rápida) | — | — | ⛔ Ignorar |
| 76 | `sfc /scannow` + DISM | Igual ao item 62 | manutenção | — | — | — | — | — | 🆕 Mesmo item 62 |
| 77 | Rede: flushdns + release/renew + `int ip reset` + `winsock reset` | Igual ao item 71 | rede | — | — | — | — | — | 🆕 Mesmo item 71 |
| 78 | `chkdsk C: /f /r` | Agenda verificação **completa** no próximo boot | disco | Médio | Fácil | Corrige erros de sistema de arquivos | — | `/r` em HD grande pode levar **horas** com o PC inutilizável | 🆕 Útil **reimaginado**: `chkdsk C: /scan` (online, sem reiniciar) e só oferecer `/spotfix` se achar erro |
| 79 | Menu Iniciar: `taskkill explorer` + `start explorer` | Reinicia o Explorer | manutenção | Baixo | — | — | — | — | 🆕 Mesmo item 57 |
| 80 | Xbox: serviços → `Start=2` (Automático) | Põe 3 serviços Xbox em Automático | jogos | Baixo | — | — | Padrão: Manual | — | 🟡 Errado (bug 10). Coberto pelo item 70 com o valor certo |
| 81 | Windows Update: para `wuauserv`/`bits`/`cryptsvc`, renomeia `SoftwareDistribution` e `catroot2` | Reset clássico do Windows Update (procedimento da própria Microsoft) | manutenção | Médio | Média | Corrige atualização travada/erro | Sim (as pastas `.old` ficam) | Todas; falha se já existir `.old` | 🆕 Útil ("Reparar Windows Update"), tratando o `.old` existente |
| 82 | Xbox app: para serviços Xbox + re-registra `*Xbox*` | Re-registra os apps Xbox | jogos | Baixo | Média | Às vezes resolve | — | — | 🟡 Baixo valor. Ignorar nesta rodada |
| 83 | Câmera: `taskkill /F` Teams/Zoom/Discord/OBS… | Fecha apps sem perguntar | manutenção | **Alto** | — | — | — | — | ⛔ Perde trabalho não salvo / chamada em andamento. Ignorar |
| 84 | Câmera/Mic: `ConsentStore\webcam|microphone /v Value /d Allow` em HKLM e HKCU + apaga políticas `AllowCamera`/`LetAppsAccess*` | **Libera câmera e microfone para todos os apps e usuários** e remove bloqueios de empresa | privacidade | **Alto** | — | — | — | — | ⛔ Passa por cima da escolha do usuário e de políticas. Ignorar (no máximo abrir `ms-settings:privacy-webcam`) |
| 85 | Câmera/Mic: desabilitar/habilitar todos os dispositivos `Camera`, `Image`, `MEDIA`, `AudioEndpoint` | Reinicia drivers | manutenção | Médio | Média | Às vezes resolve | — | `MEDIA` inclui o áudio HDMI da GPU, placas de captura etc. | 🟡 Ignorar nesta rodada |
| 86 | Câmera/Mic: `pnputil /scan-devices`, `UsoClient StartScan/StartDownload`, `winget upgrade Microsoft.WindowsCamera` | Procura hardware e atualizações | manutenção | Baixo | Fácil | Leve | — | — | 🟡 Ignorar (o Windows faz sozinho) |
| 87 | Câmera/Mic: `net stop/start` de `PlugPlay`, `DeviceInstall`, `DsmSvc`, `FrameServer` com `start= demand` | Reinicia serviços de dispositivos (e muda o tipo de início) | manutenção | Médio | — | — | `PlugPlay` não pode ser parado; muda o padrão de outros | — | 🟡 Ignorar (só `FrameServer` faz sentido, entra num futuro "Reparar câmera") |

---

## 3. Resumo por grupo

### 🆕 Útil — candidatos a implementar (nenhum existe hoje no app)

| Id proposto | Origem (#) | Nome na interface | Categoria do app | Risco | Reinício | Reversível | Perfis sugeridos | Condição |
|---|---|---|---|---|---|---|---|---|
| `services.error-reporting` | 24, 26 | Desativar o Relatório de Erros do Windows | Processos e serviços | Baixo | Não | Sim | Equilibrado, Máximo, Gamer | — |
| `privacy.app-launch-tracking` | 23 | Não registrar os apps que você abre | Sistema e privacidade | Baixo | Não | Sim | Todos | — |
| `gpu.nvidia-telemetry` | 16 | Desativar a telemetria da NVIDIA | GPU | Baixo | Não | Sim | Máximo, Gamer | GPU NVIDIA **e** serviço `NvTelemetryContainer` existe |
| `system.visual-best-performance` | 19 | Efeitos visuais: melhor desempenho | Sistema e privacidade | Baixo | Sair e entrar | Sim (guarda a máscara binária original) | Máximo | — |
| `peripherals.usb-selective-suspend` | 65 | Não suspender dispositivos USB | CPU/energia (ou nova categoria) | Baixo | Não | Sim | Gamer | Notebook: só "na tomada" |
| `network.adapter-power-saving` | 66 | Não desligar a placa de rede para economizar | Rede | Médio | Não | Sim (guarda as opções de cada placa) | Nenhum | Aviso extra em notebook (bateria e Wake-on-LAN) |
| `gaming.process-priority` | 69 | Prioridade alta de CPU para um jogo | Jogos (página Modo Gamer) | Médio | Não (vale ao abrir o jogo) | Sim | Nenhum (por jogo) | Um toggle por jogo detectado/adicionado |
| `maintenance.repair-system-files` | 62, 76 | Verificar e reparar arquivos do Windows | Manutenção | Baixo | Não | — (reparo) | — | — |
| `maintenance.restart-explorer` | 57, 79 | Reiniciar o Explorer | Manutenção | Baixo | Não | — | — | — |
| `cleanup.recent-items` | 59 | Limpar a lista de itens recentes | Manutenção | Baixo | Não | Não (apaga) | — | — |
| `maintenance.repair-windows-update` | 81 | Reparar o Windows Update | Manutenção | Médio | Recomendado | Parcial (pastas `.old`) | — | — |
| `maintenance.repair-network` | 71, 77 | Reparar a rede (redefinir Winsock e TCP/IP) | Manutenção | Médio | **Sim** | **Não** (avisar que apaga IP/DNS manual) | — | — |
| `maintenance.check-disk` | 78 | Verificar erros no disco do Windows | Manutenção | Baixo | Não (`/scan` online) | — | — | — |
| `maintenance.restart-audio` | 72 | Reiniciar o áudio do Windows | Manutenção | Baixo | Não | — | — | — |
| `maintenance.restore-default-services` | 70, 74, 80 | Restaurar serviços essenciais ao padrão | Manutenção | Baixo | Não | — | — | Só mexe nos que estiverem **Desativados** |
| `maintenance.repair-store` | 73 | Reparar a Microsoft Store | Manutenção | Baixo | Não | — | — | — |
| `security.memory-integrity-off` ⚠️ | 47 | Desativar a Integridade de Memória (HVCI) e o VBS | CPU | **Alto** | **Sim** | Sim | **Nenhum** + confirmação extra | Só com sua aprovação explícita; avisar sobre WSL2/Hyper-V/Sandbox |

### ✅ Já existe no catálogo (não duplicar)

| Item do .bat | Id no app |
|---|---|
| Ponto de restauração, backup do registro | motor (`OptimizationEngine` + `WmiRestorePointService`) |
| MPO | `gpu.mpo-off` |
| Game DVR / captura | `gaming.gamedvr` |
| Painel e botão da Game Bar | `gaming.game-bar-shortcuts` |
| Modo de Jogo | `gaming.game-mode` |
| Otimizações de tela cheia | `gaming.fullscreen-optimizations` |
| Perfil "Games" do MMCSS | `gaming.scheduler-priority` |
| SystemResponsiveness | `network.throttling` (valor correto 10) |
| Plano de energia | `cpu.ultimate-power-plan` |
| Prioridade do primeiro plano | `cpu.foreground-priority` |
| Power Throttling / "EcoQoS" | `cpu.power-throttling` |
| PCI Express | `gpu.pcie-power` |
| Transparência | `system.transparency` |
| Apps em segundo plano | `system.background-apps` |
| Telemetria (`AllowTelemetry`, DiagTrack) | `privacy.telemetry` |
| Bing na busca | `privacy.bing-search` |
| Spooler | `services.print-spooler` |
| Indexação | `services.search-indexer` |
| SysMain | `services.sysmain` (falta a condição "só SSD" → etapa A) |
| Hibernação | `storage.hibernation-off` |
| Compressão de memória | `memory.compression-off` |
| Aceleração do mouse | `gaming.mouse-acceleration` |
| Temporários, cache do Windows Update, Lixeira, DNS | `cleanup.temp`, `cleanup.windows-update`, `cleanup.recycle-bin`, `maintenance.dns-flush` |
| EmptyStandbyList (standby) | `memory.standby-list` |
| Autoruns | módulo Inicialização (`StartupManager`) |
| DNS Jumper | `network.dns-cloudflare` |

### 🟡 Placebo ou mito (ignorar)

`ipconfig /release`+`/renew` para ping (4) · esvaziar working sets e modified list (7) · "Otimizar Energia" do .bat (18) · `VisualFXSetting` em `Control Panel\Desktop` (20) · serviços Xbox desativados (31) · 3 chaves de telemetria com nome errado (33) · `CortanaConsent` e Cortana no Win 11 (39, 41) · desligar Prefetch "para HDD" (42) · `EnableSuperfetch` (43) · `SystemResponsiveness=0` (14) · `GPU Priority=8` (já é padrão) · menus por fabricante AMD/Intel/NVIDIA (são idênticos) · timer resolution/ISLC (55) · EcoQoS duplicado (64) · Xbox em Automático (80) · re-registrar Xbox app, `pnputil`, `UsoClient`, `winget` (82, 86).

### ⛔ Perigoso (não implementar)

| Item | Motivo |
|---|---|
| Desativar Biometria `WbioSrvc` (27) | Quebra o Windows Hello, sem ganho |
| Remover tudo de Xbox, inclusive provisionado (30) | Quebra Game Pass e login Xbox em jogos; não tem volta |
| `DoNotConnectToWindowsUpdateInternetLocations` (34) | Política de WSUS com nome de "telemetria"; pode quebrar a Store |
| Credential Guard `LsaCfgFlags=0`, `RequirePlatformSecurityFeatures=0`, `hypervisorlaunchtype off` como itens soltos (48–50) | Só reduzem segurança / quebram WSL2, Hyper-V, Sandbox |
| Debloater com curingas + "reverter" que re-registra tudo (51) | Remove Media Player, Spotify, Email/Calendário; não tem volta |
| `AppCompat DisableEngine` / PCA (56) | Programas e jogos antigos podem parar de abrir |
| "PC não desliga" com `shutdown /f /t 0` (75) | Fecha tudo sem salvar e não corrige nada |
| `taskkill /F` em Teams/Zoom/Discord/OBS (83) | Perde trabalho/chamada sem perguntar |
| Forçar `Allow` de câmera/microfone em HKLM e apagar políticas (84) | Passa por cima da privacidade do usuário e de políticas da empresa |
| `SystemRestorePointCreationFrequency=0` permanente (1) | O app já faz isso só durante a criação |

### ➖ Ignorado por ser interface ou ferramenta externa

Logo em ASCII, gradientes, menus, `pause`/`cls`, `chcp`, `title` · `DnsJumper.exe`, `EmptyStandbyList.exe`, `ISLC.exe`, `Autoruns.exe`, `debloater.bat` (não vieram junto, não verificáveis, e o app já cobre o que eles fazem) · botão "Reiniciar PC".

---

## 4. Plano de implementação proposto (aguardando aprovação)

Tudo em C# usando as abstrações que o app já tem (`IRegistryAccess`, `IServiceManager`, `ICommandRunner`). Nada chama o .bat, nada é copiado dele. Cada etapa = uma categoria, com testes (aplicar → reverter → estado idêntico) e um commit.

| Etapa | O que entra | Mudança de arquitetura |
|---|---|---|
| **A — Condições** | Campo `Applicability` no `Tweak` (SSD, notebook, fabricante da GPU, versão do Windows, serviço existe). A interface mostra o item desabilitado com o motivo ("Só para SSD"). Aplicar em `services.sysmain` (só SSD) | `SystemContext` ganha dados de hardware (tipo do disco do Windows, se tem bateria/chassi portátil, fabricante da GPU) |
| **B — Processos, serviços e privacidade** | `services.error-reporting`, `privacy.app-launch-tracking`, `gpu.nvidia-telemetry` | Nenhuma (ações existentes) |
| **C — Visual** | `system.visual-best-performance` | Garantir suporte a valor binário (`REG_BINARY`) no snapshot |
| **D — Energia e periféricos** | `peripherals.usb-selective-suspend`, `network.adapter-power-saving` | Nova ação `NetAdapterPowerAction` (lê e guarda as opções por placa) |
| **E — Jogos** | `gaming.process-priority` na página Modo Gamer | Nova ação `ProcessPriorityAction` (IFEO `PerfOptions`; apaga só o que criou ao reverter) |
| **F — Manutenção e reparos** | 9 tarefas de manutenção (`repair-system-files`, `restart-explorer`, `recent-items`, `repair-windows-update`, `repair-network`, `check-disk`, `restart-audio`, `restore-default-services`, `repair-store`) | Novas `MaintenanceTask` |
| **G — Risco alto (opcional)** | `security.memory-integrity-off` | Fora dos perfis, confirmação extra |

Ao final: atualizar `README.md` e `docs/03-roteiro-de-etapas.md` do NinjaOptimizer.
