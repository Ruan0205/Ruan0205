# Diagnóstico e estabilização de falhas intermitentes do shell do Windows em uma estação de trabalho pessoal

**Autor:** Ruan  
**Data do estudo de caso:** abril e maio de 2026  
**Tipo de relato:** estudo de caso técnico, observacional e interventivo em primeira pessoa

## Resumo

Neste artigo descrevo, a investigação de um conjunto de falhas intermitentes no meu Windows 10 Pro que afetavam componentes centrais da experiência gráfica do sistema. Embora a máquina tivesse bom desempenho de hardware, o shell do Windows degradava depois de algum tempo de uso: o `Alt+Tab` atrasava ou deixava de funcionar, a Calculadora e elementos da barra de tarefas demoravam a abrir, o calendário e o controle de volume não respondiam, o `Win+Tab` perdia animações, o overlay de volume ficava preso na tela e, em alguns momentos, janelas do Explorador eram abertas sem solicitação.

O comportamento tinha uma característica importante: reiniciar o computador resolvia temporariamente. Isso indicava que o problema provavelmente não estava em falta de capacidade de hardware, mas em algum acúmulo de estado, travamento do shell, extensão carregada no `explorer.exe`, overlay persistente, tarefa de inicialização ou componente de integração do sistema. A investigação foi conduzida com uma abordagem conservadora: antes das mudanças, criei pontos de restauração, exportei chaves de Registro, salvei tarefas agendadas e preservei arquivos removidos da inicialização. Em seguida, coletei eventos do Windows, relatórios do Windows Error Reporting, processos, módulos carregados no Explorer, serviços, tarefas agendadas, itens de startup, integridade do sistema e logs de um watchdog criado para monitoramento.

Os dados apontaram para travamentos recorrentes do `explorer.exe`, registrados como `AppHangB1` pelo Windows Error Reporting, com dezenas de ocorrências em poucos dias. Também identifiquei componentes de terceiros e integrações que voltavam a se acoplar ao shell: tarefas e serviços auxiliares da AMD Radeon, extensões de shell como WinRAR/PDF24/AMD Catalyst, integração do OneDrive com o Explorer, entradas de inicialização desnecessárias e overlays que interferiam na sessão gráfica. A solução aplicada combinou bloqueio seletivo de extensões de shell, desativação de tarefas e serviços auxiliares não essenciais, limpeza de caches regeneráveis, re-registro de pacotes do shell, correção de inicializações persistentes e implantação de um watchdog executado em segundo plano para recuperar o shell quando novos travamentos fossem detectados.

O resultado foi uma estabilização operacional: os sintomas visíveis foram reduzidos e a recuperação passou a ocorrer de forma automática e invisível. Ao mesmo tempo, a investigação mostrou que o sistema ainda registrava novos eventos de `AppHang` do Explorer em alguns períodos, o que torna este caso um exemplo útil de diferença entre "corrigir a experiência do usuário" e "eliminar totalmente a causa raiz profunda". A mitigação foi bem-sucedida para impedir que o problema paralisasse a interface, mas a eliminação definitiva da causa raiz provavelmente exigiria atualização/reparo in-place do Windows ou reconstrução mais profunda do perfil/shell caso os eventos continuassem.

**Palavras-chave:** Windows 10, explorer.exe, AppHangB1, shell extensions, OneDrive, AMD Radeon, PowerShell, watchdog, diagnóstico, estabilização.

## 1. Introdução

O problema que motivou este estudo não era uma lentidão geral da máquina. Pelo contrário: meu computador tinha recursos suficientes para executar as tarefas do dia a dia sem gargalos aparentes de CPU, memória ou disco. A anomalia estava concentrada em partes específicas da interface gráfica do Windows. Em alguns momentos, comandos simples da experiência do sistema pareciam entrar em colapso: alternar janelas com `Alt+Tab`, abrir a Calculadora, usar o calendário da barra de tarefas, abrir o controle de volume, acionar o `Win+Tab` ou interagir com elementos da barra de tarefas.

O sintoma mais confuso era sua natureza intermitente. Depois de reiniciar, tudo voltava a funcionar. Após algumas horas ou dias, os problemas retornavam. Isso me levou a tratar o caso como uma falha acumulativa do shell, e não como um defeito isolado de um aplicativo. A pergunta principal da investigação foi:

**Por que componentes centrais do shell do Windows deixavam de responder em uma máquina potente, e como eu poderia estabilizar isso sem arriscar quebrar o sistema operacional?**

Eu defini alguns critérios de segurança antes de agir. Primeiro, nenhuma correção deveria envolver reinstalação do Windows, exclusão agressiva de drivers ou comandos destrutivos. Segundo, toda alteração persistente precisava ter backup ou caminho de reversão. Terceiro, eu deveria separar componentes essenciais de componentes auxiliares. Por exemplo, manter o driver gráfico e o `AMD Crash Defender`, mas desativar overlays, telemetria e tarefas auxiliares que se acoplavam à sessão gráfica.

## 2. Sintomas observados

Os sintomas ocorreram ao longo de várias sessões e se repetiram mesmo após correções parciais. Os principais sinais observados foram:

- Abertura muito lenta da Calculadora do Windows.
- `Alt+Tab` demorando a responder, falhando ou ficando inconsistente.
- `Win+Tab` sem animações ou com comportamento incompleto.
- Botão de volume, calendário e outros elementos da barra de tarefas sem resposta.
- Overlay de volume preso na tela.
- Menu e componentes modernos do shell aparentando congelar.
- Janelas do Explorador abrindo ocasionalmente em "Este Computador".
- Janelas do PowerShell piscando na tela após a criação de uma tarefa de manutenção.
- Alívio temporário após reinicialização completa do computador.

O padrão "reiniciou, voltou ao normal" foi decisivo. Ele sugeria que o problema não estava simplesmente em um arquivo permanentemente corrompido, mas em processos, extensões ou integrações que entravam em estado ruim com o tempo.

## 3. Hipóteses iniciais

Antes de executar correções, eu considerei as seguintes hipóteses:

- Corrupção de arquivos do Windows ou do repositório de componentes.
- Travamentos recorrentes do `explorer.exe`.
- Extensões de shell de terceiros carregadas dentro do Explorer.
- Overlays de GPU, Game Bar, captura de vídeo ou métricas de hardware interferindo na interface.
- Integração quebrada do OneDrive com pastas do usuário, especialmente porque Área de Trabalho, Documentos e Imagens estavam redirecionados para o OneDrive.
- Itens antigos de inicialização reativando processos problemáticos no login.
- Tarefas agendadas recriando componentes que eu já havia parado manualmente.
- Cache de ícones, miniaturas ou histórico do Explorer em estado inconsistente.
- Problema residual de versão/build do Windows 10.

Essas hipóteses não foram tratadas como mutuamente exclusivas. O comportamento do Windows Shell normalmente depende de um conjunto grande de componentes em processo, fora de processo, extensões COM, pacotes AppX, serviços e tarefas agendadas. Portanto, a investigação foi incremental.

## 4. Metodologia

Adotei uma metodologia em cinco etapas: preservar, observar, isolar, intervir e validar.

### 4.1 Preservação e rollback

Antes da intervenção final, criei um ponto de restauração chamado:

```text
Codex Final Shell Fix 20260430-220833
```

Também exportei chaves relevantes do Registro, incluindo:

```text
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Shell Extensions\Blocked
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Shell Extensions\Approved
HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management
HKCU\Software\Microsoft\GameBar
HKCU\System\GameConfigStore
HKCU\Software\Microsoft\Windows\CurrentVersion\GameDVR
HKCU\Software\AMD\CN
HKCU\Software\AMD\DVR
HKLM\SYSTEM\CurrentControlSet\Services\AMD External Events Utility
HKLM\SYSTEM\CurrentControlSet\Services\AUEPLauncher
HKLM\SYSTEM\CurrentControlSet\Services\AMD Crash Defender Service
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer
HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced
```

Na primeira tentativa de exportação, o procedimento de backup registrou falhas por causa da forma como os caminhos foram passados ao `reg.exe`. Eu refiz a etapa usando uma chamada mais direta e confirmei 19 exportações de Registro concluídas com sucesso. Também exportei tarefas agendadas como XML, incluindo `StartCN`, `StartCNBM`, `StartDVR` e `CodexShellWatchdog`.

Essa etapa foi importante porque a intervenção mexeria em componentes de inicialização, tarefas e shell extensions. Eu queria poder desfazer qualquer alteração sem depender de memória ou tentativa e erro.

### 4.2 Coleta de eventos e estado do sistema

Coletei informações do sistema, processos relevantes, eventos do Windows e tarefas/serviços suspeitos. No momento da coleta final, a máquina estava com aproximadamente 1 dia e 17 horas de uptime. A versão registrada era:

```text
Windows 10 Pro
WindowsVersion: 2009
OsBuildNumber: 19045
OsHardwareAbstractionLayer: 10.0.19041.3636
Imagem DISM: 10.0.19045.3803
```

Também executei verificações de integridade:

```text
sfc /verifyonly
DISM /Online /Cleanup-Image /CheckHealth
```

O `SFC` final informou que a Proteção de Recursos do Windows não encontrou violações de integridade. O `DISM` informou que nenhuma corrupção do repositório de componentes foi detectada. Isso reduziu a probabilidade de corrupção ativa de arquivos do Windows naquele momento, embora em etapas anteriores já tivesse havido correção de integridade com `SFC` e `DISM`.

### 4.3 Análise do Windows Error Reporting

O dado mais forte veio do Windows Error Reporting. Em uma janela de 14 dias, havia 125 eventos do tipo:

```text
Windows Error Reporting, evento 1001
Nome do Evento: AppHangB1
Processo: explorer.exe
```

Isso confirmou que a falha central não era apenas "lentidão percebida": o `explorer.exe` realmente estava entrando em estado de não resposta. O Explorador do Windows não é apenas um gerenciador de arquivos; ele também sustenta barra de tarefas, shell, área de trabalho, menus e várias integrações visuais. Portanto, travamentos do `explorer.exe` explicavam bem os sintomas no `Alt+Tab`, volume, calendário, animações e janelas do shell.

### 4.4 Inspeção de módulos carregados no Explorer

Analisei os módulos não nativos carregados pelo `explorer.exe` e os relatórios `.wer` mais recentes. Em um relatório de travamento, encontrei:

```text
C:\Users\<usuario>\AppData\Local\Microsoft\OneDrive\26.040.0301.0001\FileSyncShell64.dll
```

Esse módulo era relevante porque a minha Área de Trabalho, Documentos e Imagens estavam dentro do OneDrive. Ao verificar a pasta correspondente, a versão citada no relatório não estava mais presente do modo esperado. Isso sugeriu uma integração de shell do OneDrive em estado inconsistente, possivelmente com referência residual ou componente carregado a partir de um estado anterior.

Em outra enumeração, o Explorer também carregava a extensão do WinRAR:

```text
C:\Program Files\WinRAR\rarext.dll
```

Depois das correções, a enumeração pós-fixação mostrou apenas um módulo não localizado diretamente em `C:\Windows`:

```text
C:\Program Files\Common Files\microsoft shared\ink\tiptsf.dll
```

Esse módulo é da Microsoft. A redução de extensões de terceiros dentro do Explorer foi um indicador positivo, porque extensões carregadas em processo aumentam o risco de travamentos do shell.

### 4.5 Inspeção de tarefas, serviços e inicialização

Eu encontrei tarefas agendadas da AMD reativadas:

```text
\AMDRyzenMasterSDKTask
\StartCN
\StartCNBM
\StartDVR
```

Essas tarefas executavam componentes como:

```text
cpumetricsserver.exe
cncmd.exe startwithdelay
cncmd.exe benchmark
RSServCmd.exe
```

Também encontrei serviços auxiliares da AMD ativos:

```text
AMD External Events Utility
AUEPLauncher
```

Mantive o serviço `AMD Crash Defender Service`, por ser mais diretamente relacionado à robustez do driver. A distinção foi proposital: eu queria remover acoplamentos desnecessários da sessão gráfica sem comprometer o driver de vídeo.

Na inicialização do usuário, identifiquei:

```text
AMDNoiseSuppression
degraça.bat
degraça - Atalho.lnk
```

O arquivo `.bat` continha:

```bat
BCDEDIT -set TESTSIGNING OFF
```

Esse comando não explicava sozinho todos os sintomas, mas era desnecessário rodá-lo a cada login. Como medida conservadora, movi o `.bat` e seu atalho para uma pasta de backup, em vez de apagá-los.

## 5. Intervenções aplicadas

As correções foram aplicadas em camadas. O objetivo era reduzir o número de componentes que se prendiam ao shell e impedir que processos auxiliares voltassem sozinhos.

### 5.1 Bloqueio seletivo de extensões de shell

Usei a chave:

```text
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Shell Extensions\Blocked
```

Bloqueei extensões associadas a PDF24, AMD Catalyst e WinRAR. Entre os CLSIDs bloqueados estavam:

```text
{F78FD16B-3DA7-4935-82E9-B82D9C1ED0AE} - PDF24 Shell Extension
{09E6D117-5330-4A29-8C20-0C3AF9F90A1C} - PDF24 Preview Handler
{3AF5A38C-78A5-4CE1-BCE5-6421BF94DCAD} - PDF24 Thumbnail Handler
{5E2121EE-0300-11D4-8D3B-444553540000} - AMD Catalyst Context Menu extension
{B41DB860-64E4-11D2-9906-E49FADC173CA} - WinRAR shell extension
{B41DB860-8EE4-11D2-9906-E49FADC173CA} - WinRAR shell extension legacy/aprovada
```

Essa decisão não removeu os aplicativos. Ela apenas impediu que essas extensões fossem carregadas dentro do Explorer. Para um diagnóstico de estabilidade, esse é um corte relativamente seguro: se a instabilidade reduz, uma extensão de shell era parte do problema; se houver efeito colateral, o backup permite reverter.

### 5.2 Desativação de componentes auxiliares da AMD

Desativei as tarefas:

```text
schtasks /Change /TN \AMDRyzenMasterSDKTask /DISABLE
schtasks /Change /TN \StartCN /DISABLE
schtasks /Change /TN \StartCNBM /DISABLE
schtasks /Change /TN \StartDVR /DISABLE
```

Também parei e desativei:

```text
AMD External Events Utility
AUEPLauncher
```

E encerrei processos de overlay/telemetria/captura quando estavam ativos:

```text
RadeonSoftware
AMDRSServ
AMDRSSrcExt
amdow
cncmd
cpumetricsserver
RSServCmd
AMDNoiseSuppression
GameBar
GameBarFTServer
```

O driver de vídeo em si não foi removido. Essa foi uma escolha de segurança: eu não queria trocar uma falha intermitente de shell por um problema gráfico mais grave.

### 5.3 Ajustes no OneDrive

Como o relatório de travamento do Explorer carregava `FileSyncShell64.dll`, apliquei uma mitigação na integração do OneDrive com o shell:

```text
HKCU\Software\Microsoft\OneDrive
FileSyncShellKillSwitchFlag = 1
AutoStartEnabled = 0
```

Essa ação não apagou os arquivos do OneDrive nem desinstalou o aplicativo. O objetivo foi reduzir a integração direta com o Explorer e impedir que uma extensão quebrada fosse parte ativa do ciclo de travamento.

### 5.4 Limpeza de inicialização

Removi da inicialização:

```text
AMDNoiseSuppression
degraça.bat
degraça - Atalho.lnk
```

Os itens da pasta Startup foram movidos para backup. Eu já havia lidado anteriormente com atalhos quebrados como `Stardock ObjectDock.lnk` e `system.lnk`. A ideia geral foi manter a sessão de login limpa e previsível.

### 5.5 Limpeza de caches regeneráveis do Explorer

Com os componentes do shell parados, removi caches regeneráveis:

```text
iconcache_*.db
thumbcache_*.db
```

Esses arquivos são recriados pelo Windows. A limpeza não resolve causa raiz sozinha, mas ajuda quando o Explorer está com estado visual corrompido, miniaturas inconsistentes ou histórico pesado.

### 5.6 Re-registro de pacotes do shell

Re-registrei pacotes AppX relacionados ao shell e aplicativos modernos:

```text
Microsoft.Windows.ShellExperienceHost
Microsoft.Windows.Search
Microsoft.WindowsCalculator
Microsoft.WindowsStore
```

O `StartMenuExperienceHost` recusou o re-registro em uma tentativa porque o pacote voltou a ficar em uso rapidamente, gerando `0x80073D02`. Como o status do pacote estava íntegro e outros pacotes re-registraram corretamente, tratei isso como uma limitação operacional do momento, não como evidência de corrupção ativa.

### 5.7 Watchdog de shell

Como o problema era intermitente, criei um script de manutenção:

```text
maintenance\windows-shell-watchdog.ps1
```

O script monitora sinais de falha do shell e age quando necessário. Os parâmetros principais ficaram:

```powershell
$maxHandles = 7000
$maxWorkingSetBytes = 600MB
$maxUptimeHours = 24
$recentHangWindowMinutes = 45
```

Ele consulta eventos recentes do Windows Error Reporting:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Application'
    ProviderName = 'Windows Error Reporting'
    Id = 1001
    StartTime = (Get-Date).AddMinutes(-$recentHangWindowMinutes)
}
```

Depois filtra por:

```powershell
AppHangB1
explorer.exe
```

Se detectar um travamento recente ainda não tratado, ele encerra overlays conhecidos, reinicia componentes do shell e registra a ação em:

```text
maintenance\logs\windows-shell-watchdog.log
```

O script foi agendado para rodar no login e a cada 30 minutos. Inicialmente, ele chamava `powershell.exe` diretamente e isso fazia uma janela piscar na tela. Corrigi esse efeito colateral criando um wrapper VBScript:

```vbscript
command = "powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File " & Chr(34) & psScript & Chr(34)
shell.Run command, 0, False
```

A tarefa passou a chamar:

```text
wscript.exe run-windows-shell-watchdog-hidden.vbs
```

Também corrigi outro efeito colateral: quando o script reiniciava o Explorer enquanto outro `explorer.exe` ainda estava vivo, o Windows podia abrir uma janela em "Este Computador". Ajustei a lógica para aguardar os processos antigos e fechar automaticamente janelas raiz como "Este Computador", "This PC" e "Quick access" se fossem criadas por acidente.

## 6. Resultados

Após a intervenção, validei os seguintes pontos:

- `SFC /verifyonly` não encontrou violações de integridade.
- `DISM /CheckHealth` não detectou corrupção do repositório de componentes.
- As tarefas AMD (`StartCN`, `StartCNBM`, `StartDVR`, `AMDRyzenMasterSDKTask`) ficaram desabilitadas.
- Os serviços auxiliares `AMD External Events Utility` e `AUEPLauncher` ficaram parados e desabilitados.
- O `AMD Crash Defender Service` permaneceu ativo.
- O startup do usuário ficou mais limpo.
- A pasta Startup passou a conter apenas `desktop.ini`.
- O Explorer pós-correção deixou de carregar extensões de terceiros identificadas anteriormente, ficando apenas com componente Microsoft relacionado a entrada/ink.
- O watchdog passou a executar sem janela de PowerShell visível.
- O watchdog deixou de abrir janelas "Este Computador" na tela.

Uma validação imediata registrou "nenhuma ação necessária" no log do watchdog. Isso indicou que a automação não estava em loop e que os limiares estavam aceitáveis.

## 7. Observação pós-intervenção

O monitoramento posterior mostrou um resultado importante: os eventos `AppHangB1` do `explorer.exe` continuaram aparecendo em alguns horários. Entre 30 de abril e 3 de maio de 2026, foram observados 33 registros desse tipo no intervalo analisado, com eventos distribuídos ao longo dos dias. Porém, o watchdog passou a capturar essas ocorrências e impedir que a falha ficasse visivelmente presa na interface.

Alguns registros do log exemplificam o comportamento:

```text
[2026-05-03 13:40:01] Reciclando shell: AppHang recente do explorer.exe
[2026-05-03 13:40:06] explorer.exe ainda estava ativo; nao abri nova janela do Explorer
[2026-05-03 14:10:01] Nenhuma acao necessaria
```

Esse resultado mudou minha interpretação. A intervenção não deve ser descrita como eliminação completa da causa raiz, mas como estabilização operacional com recuperação automática e redução de efeitos visuais. Em outras palavras: eu passei a ter um mecanismo de defesa contra o travamento, e removi várias causas prováveis ou agravantes, mas o Windows ainda emitia sinais de instabilidade profunda do Explorer.

## 8. Discussão

Este caso mostra como problemas de shell no Windows podem parecer "aleatórios" para o usuário, mas deixar rastros mensuráveis. A princípio, os sintomas pareciam desconectados: Calculadora lenta, volume preso, `Alt+Tab` falhando, animações ausentes. A análise dos eventos mostrou que todos podiam se conectar ao mesmo eixo: o `explorer.exe` e seus componentes adjacentes.

Também ficou claro que reiniciar a máquina era uma solução temporária porque reiniciar limpava o estado ruim do shell. Contudo, se as mesmas extensões, tarefas e integrações voltavam no login, a degradação reaparecia depois.

Três grupos de componentes mereceram atenção especial:

1. **Extensões de shell em processo.** Extensões como WinRAR, PDF24 e AMD Catalyst podem ser úteis, mas quando carregadas dentro do Explorer aumentam a superfície de falha. Bloqueá-las é uma técnica útil de isolamento.

2. **Overlays e serviços auxiliares de GPU.** Componentes de captura, métricas, DVR, noise suppression e overlays podem interferir com a experiência gráfica. Neste caso, tarefas AMD voltavam a iniciar processos mesmo depois de correções anteriores.

3. **Integração do OneDrive com pastas do usuário.** Como minha Área de Trabalho estava dentro do OneDrive, o Explorer dependia de integração de sincronização para itens muito usados. O relatório de travamento citando `FileSyncShell64.dll` tornou esse ponto relevante.

Outro aprendizado foi que uma automação de reparo precisa ser invisível. Na primeira versão, o watchdog abriu janelas de PowerShell. Depois, ao reiniciar o Explorer, podia abrir "Este Computador". Essas duas coisas foram efeitos colaterais aceitáveis durante desenvolvimento, mas inadequadas para uso diário. A solução final executa em segundo plano e evita abrir janelas.

## 9. Limitações

Este estudo tem limitações claras.

Primeiro, trata-se de um único computador, com um conjunto específico de drivers, softwares, tarefas e histórico de uso. Não posso generalizar que toda falha de `Alt+Tab` ou volume preso no Windows tenha a mesma causa.

Segundo, a intervenção removeu ou isolou vários fatores simultaneamente. Isso foi adequado para estabilizar uma máquina de uso real, mas dificulta atribuir causalidade absoluta a um único componente.

Terceiro, os eventos `AppHangB1` continuaram ocorrendo após a intervenção. Isso sugere que a causa raiz profunda ainda poderia estar em outro componente, no perfil do usuário, na própria build do Windows, em integração residual do OneDrive ou em um conjunto de interações entre shell e softwares auxiliares.

Quarto, o Windows 10 observado estava em uma build antiga em relação ao período do estudo. Uma etapa mais forte, caso eu buscasse eliminar a causa raiz e não apenas estabilizar o uso, seria um reparo in-place/atualização do Windows preservando arquivos e aplicativos, ou uma migração planejada para uma instalação mais nova e limpa.

## 10. Conclusão

Eu comecei com sintomas que pareciam subjetivos: "às vezes o Windows para de obedecer". Ao investigar os logs, encontrei um padrão objetivo: travamentos recorrentes do `explorer.exe` registrados pelo Windows Error Reporting. A partir disso, tratei o caso como instabilidade do shell, não como lentidão genérica.

A solução que implementei foi conservadora e reversível. Criei backups, exportei Registro, salvei tarefas, preservei itens removidos e mantive o driver essencial. Em seguida, reduzi a superfície de falha do Explorer, desativei overlays e tarefas auxiliares que voltavam sozinhos, corrigi inicializações desnecessárias, limpei caches, re-registrei componentes do shell e implantei um watchdog invisível.

O resultado prático foi que a máquina passou a ter recuperação automática e menos interferência visual. O caso também me ensinou que "resolver" pode ter duas camadas: eliminar a causa raiz profunda ou estabilizar o sistema para que o problema não interrompa o uso. Neste caso, alcancei uma estabilização operacional robusta, com caminho de rollback e monitoramento contínuo. Se os eventos de `AppHang` continuarem por muito tempo, a próxima etapa tecnicamente mais limpa será um reparo in-place do Windows ou uma atualização planejada, porque os dados indicam que o shell ainda entra em estado de falha mesmo depois de reduzir extensões e overlays.

## Apêndice A: parâmetros finais do watchdog

```powershell
$maxHandles = 7000
$maxWorkingSetBytes = 600MB
$maxUptimeHours = 24
$recentHangWindowMinutes = 45
```

Processos de overlay monitorados:

```powershell
RadeonSoftware
AMDRSServ
AMDRSSrcExt
amdow
cncmd
cpumetricsserver
RSServCmd
AMDNoiseSuppression
GameBar
GameBarFTServer
```

Gatilhos da tarefa:

```text
Ao fazer login
A cada 30 minutos
```

Modo de execução:

```text
wscript.exe -> VBScript -> powershell.exe -WindowStyle Hidden
```

## Apêndice B: componentes preservados por segurança

Eu não removi o driver gráfico AMD. Também mantive:

```text
AMD Crash Defender Service
Áudio do Windows
Construtor de Pontos de Extremidade de Áudio do Windows
Realtek Audio Service
```

Essa separação foi importante para não trocar um problema de shell por perda de vídeo ou áudio.

## Apêndice C: estratégia de reversão

Mantive um arquivo de rollback com instruções para:

- Restaurar extensões bloqueadas importando o backup de `Shell Extensions\Blocked`.
- Recriar o valor `AMDNoiseSuppression` na inicialização, se necessário.
- Voltar `FileSyncShellKillSwitchFlag` para `0` e `AutoStartEnabled` para `1` no OneDrive.
- Reativar tarefas AMD com `schtasks /Change /ENABLE`.
- Reativar serviços AMD auxiliares com `sc config`.
- Restaurar itens movidos da pasta Startup.
- Restaurar a definição anterior do `CodexShellWatchdog`.

Essa documentação de reversão foi parte essencial da solução, porque uma intervenção no shell do Windows deve sempre ser feita com a possibilidade de voltar ao estado anterior.
