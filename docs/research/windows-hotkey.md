# Atalho global Ctrl+Win, inserção de texto e widget no Windows

Pesquisa do ticket [issue 3](https://github.com/Marcio-RF/my-whisper/issues/3). Levantada em 2026-11-08. Reúne fatos e restrições; as decisões ficam para tickets posteriores.

Nada foi executado: as conclusões vêm de documentação e de leitura de código. O que depende de teste real está na seção [Não verificado](#não-verificado).

Código de referência lido nos commits [TypeLate `656af15`](https://github.com/bobo52310/TypeLate/tree/656af158c199ab785c8a107b87eb9b2bcab4c2c5) (Tauri 2) e [OpenWhispr `cb08c41`](https://github.com/OpenWhispr/openwhispr/tree/cb08c411dd008c57e4f0a8193756d6b5812479fd) (Electron 43).

## Resumo

- Ctrl+Win não pode ser registrado pelas APIs de atalho global do sistema, do Electron ou do Tauri, porque todas exigem uma tecla que não seja modificadora. Precisa de um hook `WH_KEYBOARD_LL`, que é código nativo nos dois frameworks.
- Ctrl+Win é viável como padrão: é o padrão do Wispr Flow e do OpenWhispr no Windows. Um hook que só observa não deve abrir o Iniciar, mas a Microsoft não documenta essa regra e há um caso de borda sem confirmação.
- A inserção comum nos dois apps de referência é área de transferência mais Ctrl+V via `SendInput`. Com janelas elevadas, tanto o atalho quanto a inserção falham, e a falha da inserção é silenciosa.
- O FAB sem foco é `WS_EX_NOACTIVATE` mais `WS_EX_TOPMOST`, e os dois frameworks expõem isso. Falta testar o clique com WebView2.
- Em tamanho, a diferença é grande (cerca de 10 MiB contra mais de 150 MiB). Em memória no Windows, não há medição confiável.

## 1. Ouvir Ctrl+Win

### Por que as APIs de atalho global não servem

| API | O que exige | Consequência para Ctrl+Win |
| --- | --- | --- |
| [`RegisterHotKey`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-registerhotkey) | Modificadores mais um código de tecla `vk`. Atalhos com a tecla Windows são declarados reservados ao sistema. | Sem como registrar só modificadores. |
| Electron [`globalShortcut`](https://www.electronjs.org/docs/latest/api/global-shortcut) | [Acelerador](https://www.electronjs.org/docs/latest/tutorial/keyboard-shortcuts): vários modificadores e um código de tecla. A documentação só descreve o evento de pressionar. | O OpenWhispr [desvia do `globalShortcut`](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/hotkeyManager.js#L51-L52) nesse caso. |
| Tauri [`global-shortcut`](https://tauri.app/plugin/global-shortcut/) | Usa o crate `global-hotkey`, que [chama `RegisterHotKey` com `vk`](https://github.com/tauri-apps/global-hotkey/blob/c0ee632d8dd4725f2c299cb010453590baa6ecec/src/platform_impl/windows/mod.rs#L77-L97). | Mesmo limite. Tem estado `Released`, obtido [consultando `GetAsyncKeyState` a cada 50 ms](https://github.com/tauri-apps/global-hotkey/blob/c0ee632d8dd4725f2c299cb010453590baa6ecec/src/platform_impl/windows/mod.rs#L153-L170). |

### O que um hook `WH_KEYBOARD_LL` exige

Segundo a [documentação de `LowLevelKeyboardProc`](https://learn.microsoft.com/en-us/windows/win32/winmsg/lowlevelkeyboardproc):

- A thread que instala o hook precisa de laço de mensagens, e o callback roda nela.
- O callback tem de retornar dentro de `LowLevelHooksTimeout`. Desde o Windows 10 1709 o teto é 1000 ms.
- Se estourar o tempo, a partir do Windows 7 o hook é removido em silêncio, sem que o app possa saber.
- A recomendação é uma thread dedicada que repassa o trabalho a outra e retorna imediatamente.
- `GetAsyncKeyState` não reflete a tecla do evento atual dentro do callback.
- Retornar valor diferente de zero bloqueia a tecla para o resto do sistema.
- A mesma página sugere Raw Input como alternativa na maioria dos casos em que só se quer monitorar. Nenhum dos apps de referência o usa.

[`KBDLLHOOKSTRUCT`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-kbdllhookstruct) traz `time` e o flag `LLKHF_INJECTED`, mas nenhum indicador de repetição. Quem usa o hook precisa guardar o próprio estado para ignorar a repetição automática.

### Ctrl+Win e o menu Iniciar

A Microsoft não documenta quando soltar a tecla Windows abre o Iniciar. As melhores fontes são indiretas:

- A [documentação do AutoHotkey sobre `A_MenuMaskKey`](https://www.autohotkey.com/docs/v2/lib/_MenuMaskKey.htm) descreve o comportamento observado: o Iniciar normalmente não abre quando o último pressionamento de Win veio modificado por Ctrl, Shift ou Alt. O mascaramento só é necessário quando nenhum outro evento de tecla não bloqueado ocorreu desde o Win.
- A [documentação do Wispr Flow](https://docs.wisprflow.ai/articles/2612050838-supported-unsupported-keyboard-hotkey-shortcuts) diz que segurar Ctrl+Win evita a ação de Iniciar/Pesquisa, e o [guia inicial](https://docs.wisprflow.ai/get-started/first-dictation) dá Ctrl+Win como padrão no Windows.
- O OpenWhispr usa [`Control+Super` como padrão](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/hotkeyManager.js#L18). Seu hook [sempre repassa os eventos](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/resources/windows-key-listener.c#L239-L254), sem mascarar nada. A busca por "Start menu" nas issues dele não trouxe nenhum relato do problema, o que é evidência fraca.

Caso de borda: o AutoHotkey avisa que a repetição automática também vale para Win quando ela foi a última tecla física pressionada. Se o usuário soltar Ctrl antes de Win, uma repetição pode chegar sem Ctrl, e o Iniciar poderia abrir. É inferência, sem teste.

Há dois caminhos conhecidos para impedir o Iniciar quando o hook bloqueia teclas:

- **Tecla máscara:** injetar uma tecla sem efeito antes de soltar Win. O AutoHotkey recomenda `vkE8`, mas lembra que a Microsoft pode dar função a códigos não atribuídos.
- **Bloquear o Win:** o Wispr Flow informa que atalhos de tecla única não chegam aos outros apps. Bloquear Ctrl+Win sem repor os eventos quebraria os atalhos abaixo.

### Conflitos com o sistema

[Atalhos do Windows](https://support.microsoft.com/en-us/windows/keyboard-shortcuts-in-windows-dcc61a57-8ff0-cffe-9796-cb9706c75eec) não incluem Ctrl+Win sozinho, mas ele é o prefixo de vários:

- `Win+Ctrl+D`, `Esquerda`, `Direita`, `F4` (áreas de trabalho virtuais);
- `Win+Ctrl+Q` (Assistência Rápida), `Win+Ctrl+Enter` (Narrador), `Win+Ctrl+F`, `Win+Ctrl+C`, `Win+Ctrl+Espaço`;
- `Win+Ctrl+V` (som no Windows 11, shoulder taps no 10);
- `Win+Ctrl+número` e `Win+Ctrl+Shift+B`.

O listener do OpenWhispr [olha só as modificadoras](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/resources/windows-key-listener.c#L239-L254), então `Win+Ctrl+D` também dispara o início do Ditado nele. Quem quiser evitar isso precisa cancelar ao detectar outra tecla, ou esperar um limiar antes de começar.

### Outros detalhes que o OpenWhispr já tratou

- **Estado preso:** `Win+L` perde o key-up. O app [ressincroniza as outras modificadoras com `GetAsyncKeyState`](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/resources/windows-key-listener.c#L91-L109).
- **AltGr:** em layouts com AltGr o Windows envia um Ctrl esquerdo sintético, marcado no scan code. O app [o ignora](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/resources/windows-key-listener.c#L35-L40). O ABNT2 tem AltGr.
- **Troca de atalho:** a [issue 364](https://github.com/OpenWhispr/openwhispr/issues/364) relata estado inconsistente ao alternar entre listeners.

## 2. Distinguir segurar de tocar

Não há API para isso. É um temporizador sobre os eventos de descer e subir da combinação.

- **TypeLate:** o modo é [escolhido na configuração](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L34-L38) (`Hold`, `Toggle`, `DoubleTap`). A janela de toque duplo é [400 ms](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L53).
- **OpenWhispr:** também por configuração, [`push` ou `tap`](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/hotkeyManager.js#L473-L474).

Nenhum dos dois distingue os gestos automaticamente na mesma combinação. Quem fizer isso enfrenta a seguinte restrição: no key-down ainda não se sabe qual será o gesto. Ou a captura começa de imediato e é descartada se for toque, ou o início do áudio espera o limiar.

## 3. Inserir o texto

| | Área de transferência + Ctrl+V | `SendInput` com `KEYEVENTF_UNICODE` |
| --- | --- | --- |
| Mecanismo | Grava o texto e injeta Ctrl+V | [Gera `VK_PACKET`, que vira `WM_CHAR` via `TranslateMessage`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-keybdinput) |
| Efeito colateral | Altera a área de transferência | Nenhum nela |
| Depende do app alvo | Do atalho de colar: terminais pedem Ctrl+Shift+V | De o app tratar `WM_CHAR` |
| Uso nos apps de referência | TypeLate e OpenWhispr | Nenhum dos dois |

Pontos conhecidos do caminho pela área de transferência:

- **Modificadoras seguradas:** [`SendInput` não reinicia o estado do teclado](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput). Com Ctrl+Win ainda pressionado, o Ctrl+V vira `Win+Ctrl+V`, que abre outra coisa. O OpenWhispr [solta e repõe as modificadoras](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/resources/windows-fast-paste.c#L331-L349). O TypeLate [não faz isso](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/clipboard_paste.rs#L72-L103).
- **Terminais:** no Windows Terminal, o Ctrl+V só inseriu `v` ([issue 438 do OpenWhispr](https://github.com/OpenWhispr/openwhispr/issues/438)). O app passou a [detectar terminais por classe de janela e executável e enviar Ctrl+Shift+V](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/resources/windows-fast-paste.c#L31-L53).
- **Restauração:** o OpenWhispr [guarda texto, HTML, RTF e imagem](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/clipboard.js#L859-L916), restaura [depois de 500 ms](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/clipboard.js#L44-L51) e [desiste se o conteúdo mudou](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/clipboard.js#L921-L956). O TypeLate [guarda só texto e limpa a área de transferência se o conteúdo original não era texto](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/clipboard_paste.rs#L186-L216), o que perde uma imagem copiada.
- **Histórico:** o formato [`ExcludeClipboardContentFromMonitorProcessing`](https://learn.microsoft.com/en-us/windows/win32/dataxchg/clipboard-formats) tira o item do histórico e da sincronização. `CanIncludeInClipboardHistory` e `CanUploadToCloudClipboard` controlam cada um separadamente. Sem isso, o texto ditado fica no histórico do Windows.
- **Disputa:** [`OpenClipboard` falha se outra janela estiver com a área de transferência aberta](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-openclipboard), portanto precisa de nova tentativa.
- **Janela alvo:** algo pode tomar o primeiro plano durante a transcrição. O OpenWhispr [guarda a janela no início e a reativa antes de colar](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/resources/windows-fast-paste.c#L176-L213), usando `AttachThreadInput` porque [`SetForegroundWindow` é restrito](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setforegroundwindow).

### Limites com apps elevados

- `SendInput` só injeta em processos de integridade igual ou menor. [Quando o UIPI bloqueia, nem o retorno nem `GetLastError` indicam a causa](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput).
- O atalho também para: segundo o [FAQ do AutoHotkey](https://www.autohotkey.com/docs/v2/FAQ.htm), hotkeys de processos não elevados não funcionam com janela elevada ativa. O [documento da Microsoft sobre integridade](https://learn.microsoft.com/en-us/previous-versions/dotnet/articles/bb625963(v=msdn.10)) lista a leitura de entrada em todos os níveis por hooks como capacidade de processos com UIAccess.
- [UIAccess exige assinatura Authenticode, instalação em local seguro como Program Files e manifesto](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-securityoverview). A Microsoft diz que não deve ser usado por apps que não são tecnologia assistiva. Mesmo com UIAccess, usuário sem privilégios de administrador não alcança janelas elevadas de nível alto.
- Consequência prática: o FAB continuaria clicável sobre uma janela elevada, mas a inserção falharia sem aviso (inferência). O texto ficaria apenas na área de transferência.

## 4. FAB sempre visível e sem foco

Os [estilos estendidos](https://learn.microsoft.com/en-us/windows/win32/winmsg/extended-window-styles) relevantes:

- `WS_EX_NOACTIVATE`: clicar não torna a janela a de primeiro plano;
- `WS_EX_TOPMOST`: fica acima das janelas que não são topmost;
- `WS_EX_TOOLWINDOW`: fora da barra de tarefas e do Alt+Tab.

| | Electron | Tauri 2 |
| --- | --- | --- |
| Sem foco | [`focusable: false`](https://github.com/electron/electron/blob/ef2710fa6c10fe56365a9949fa783d26ca1499e5/docs/api/structures/base-window-options.md#L26-L30), que [aplica `WS_EX_NOACTIVATE`](https://github.com/electron/electron/blob/ef2710fa6c10fe56365a9949fa783d26ca1499e5/shell/browser/native_window_views.cc#L1451-L1461) e implica `skipTaskbar` | [`focusable`](https://github.com/tauri-apps/tauri/blob/tauri-v2.12.5/crates/tauri-utils/src/config.rs#L2060), que o tao [traduz em `WS_EX_NOACTIVATE`](https://github.com/tauri-apps/tao/blob/d12848aabf0263b1ed5436fd8dcead054c2ba5d9/src/platform_impl/windows/window_state.rs#L293-L294) |
| Mostrar sem ativar | [`showInactive()`](https://www.electronjs.org/docs/latest/api/browser-window) | `focus: false` na configuração |
| Sempre no topo | [`setAlwaysOnTop` com níveis](https://www.electronjs.org/docs/latest/api/browser-window); de `pop-up-menu` para cima fica sobre a barra de tarefas | `alwaysOnTop` |
| Uso nos apps de referência | [OpenWhispr](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/windowConfig.js#L193-L203) usa `focusable: false` e [`pop-up-menu`](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/windowConfig.js#L404) | [TypeLate](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/tauri.conf.json) usa `alwaysOnTop` e `focus: false`, sem `focusable: false` |

Com o Tauri a dúvida é o clique: o wry [encaminha o foco para o WebView2 filho](https://github.com/tauri-apps/wry/blob/e776bbb697fe444c6308d4d7f3edd8863ce0489b/src/webview2/mod.rs#L206-L212) em `WM_SETFOCUS`, e não achei confirmação pública de que clicar num WebView2 dentro de janela `WS_EX_NOACTIVATE` preserva o foco do app alvo.

## 5. Microfone

- **Privacidade:** o Windows tem [uma chave geral para apps de desktop](https://support.microsoft.com/en-us/windows/windows-camera-microphone-and-privacy-a83257bc-e990-d54a-d212-b5e41beba857), sem controle por app. Desativada, o Ditado não captura.
- **Pelo Chromium (`getUserMedia`):** é o caminho do [OpenWhispr](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/hooks/usePermissions.ts). Fica em TypeScript. No Electron, a permissão passa por [`setPermissionRequestHandler`](https://www.electronjs.org/docs/latest/api/session), com o tipo `media`. No Tauri, o wry [só libera automaticamente a leitura da área de transferência](https://github.com/tauri-apps/wry/blob/e776bbb697fe444c6308d4d7f3edd8863ce0489b/src/webview2/mod.rs#L524-L536). O microfone cai no fluxo do WebView2, salvo uso de [`on_permission_request`](https://github.com/tauri-apps/tauri/blob/tauri-v2.12.5/crates/tauri/src/webview/mod.rs#L749).
- **Nativo:** o TypeLate usa [`cpal` em Rust](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/Cargo.toml), fora do WebView2.

## 6. Tauri v2 e Electron

Nenhum dos dois oferece, em TypeScript puro, o hook de modificadoras nem a injeção de teclas. Código nativo é inevitável aqui.

| Necessidade | Electron | Tauri 2 |
| --- | --- | --- |
| Hook Ctrl+Win | Executável auxiliar em C, como o [OpenWhispr](https://github.com/OpenWhispr/openwhispr/blob/cb08c411dd008c57e4f0a8193756d6b5812479fd/src/helpers/windowsKeyManager.js) (cerca de 390 linhas, isolado do processo Node), ou [`uiohook-napi`](https://github.com/SnosMe/uiohook-napi) | Rust no próprio app, como o [TypeLate](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L529-L648) (cerca de 120 linhas) |
| Colar | Auxiliar nativo ou `uiohook-napi` (`keyTap`) | Rust com o crate `windows` |
| Área de transferência | [`clipboard`](https://www.electronjs.org/docs/latest/api/clipboard) em TypeScript, com múltiplos formatos | `arboard` em Rust (o TypeLate só usa texto) |
| Microfone | `getUserMedia` | `getUserMedia` ou `cpal` |

Sobre as opções sem helper no Electron:

- `uiohook-napi` usa [`WH_KEYBOARD_LL` via libuiohook](https://github.com/kwhat/libuiohook/blob/1.2/src/windows/input_hook.c) e expõe [`keydown` e `keyup`](https://github.com/SnosMe/uiohook-napi) sem API para consumir eventos. Serve para observar, não para bloquear.
- Com [Koffi, callbacks JavaScript rodam sempre na thread principal](https://koffi.dev/callbacks). Um hook escrito assim dependeria do laço de eventos do Node responder dentro do teto de 1000 ms, com remoção silenciosa em caso de atraso (inferência a partir das duas documentações).

### Custo em tamanho

Dados dos assets das releases no GitHub:

| Artefato | Bytes | MiB |
| --- | ---: | ---: |
| [Electron 45.2.3, `win32-x64.zip`](https://github.com/electron/electron/releases/tag/v45.2.3), só o runtime | 158.182.975 | 150,9 |
| [OpenWhispr 1.11.0, instalador](https://github.com/OpenWhispr/openwhispr/releases/tag/v1.11.0) (traz binários próprios, não é piso do Electron) | 268.882.032 | 256,4 |
| [TypeLate 1.6.3, instalador x64](https://github.com/bobo52310/TypeLate/releases/tag/v1.6.3) (Tauri 2) | 11.016.258 | 10,5 |

O Tauri depende do WebView2, [incluído no Windows 11 e já presente na grande maioria dos Windows 10](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/distribution). Para os demais há o bootstrapper de cerca de 2 MB. O modo Fixed Version acrescenta mais de 250 MB.

### Custo em memória

Não encontrei medição confiável no Windows. O único número com método declarado é um [teste no macOS, N=1, com seis janelas](https://dev.to/gethopp/tauri-vs-electron-performance-bundle-size-and-the-real-trade-offs-1el4): cerca de 172 MB no Tauri e 409 MB no Electron. Ele não se transfere para o Windows, pois lá o Tauri não usa WKWebView.

O [WebView2 tem o mesmo modelo de processos do Edge](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/process-model): browser, renderizadores, GPU e serviço de áudio. Portanto, a vantagem do Tauri em memória tende a ser bem menor que em disco. Para medir, somar o working set do app e de todos os `msedgewebview2.exe` filhos, ociosos e durante a captura.

### Outro fato relevante para a escolha

O projeto também mira o Android. O [Tauri 2 declara suporte a plataformas móveis](https://v2.tauri.app/start/); o Electron é só para desktop.

## 7. Comparação com o `hotkey_listener.rs` do TypeLate

| Aspecto | TypeLate | Necessário para Ctrl+Win |
| --- | --- | --- |
| Instalação | [Thread própria com laço de mensagens](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L572-L585), correta | Reaproveitável |
| Gatilho | [Um único `vkCode`](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L629-L637); padrão no Windows é [Alt direito](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L659) | Rastrear duas modificadoras e seus lados |
| Bloqueio | [Sempre `CallNextHookEx`](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L646) | Igual, se só observar |
| Repetição | [`is_pressed.swap`](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L116) | Reaproveitável |
| Lock no callback | [`try_lock`](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L625), não bloqueia | Reaproveitável |
| Trabalho no callback | [Chama `capture_surrounding_text`](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L117), que lê o campo em foco de forma síncrona | Contraria a orientação da Microsoft e arrisca a remoção silenciosa do hook |
| Estado preso | Só com [comando explícito](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/hotkey_listener.rs#L489-L492) | Ressincronizar, como o OpenWhispr |
| Outra tecla na combinação | Não se aplica | Tratar os prefixos `Win+Ctrl+...` |

O [`keyboard_monitor.rs`](https://github.com/bobo52310/TypeLate/blob/656af158c199ab785c8a107b87eb9b2bcab4c2c5/src-tauri/src/plugins/keyboard_monitor.rs#L243-L335) instala um segundo hook só para notar edições depois da colagem (Backspace, Delete, Enter). Não participa do atalho.

## Não verificado

1. **Iniciar em todas as ordens de teclas**, no Windows 10 e 11, sobretudo soltar Ctrl antes de Win depois de segurar. A regra não é documentada pela Microsoft.
2. **Clique no FAB do Tauri** sem tirar o foco do app alvo. Vale testar também no Electron, embora o OpenWhispr já distribua essa configuração.
3. **Memória real** no Windows, nos dois frameworks.
4. **`KEYEVENTF_UNICODE`:** velocidade em textos longos, emojis (pares substitutos) e compatibilidade por app.
5. **Convivência com outros hooks** (PowerToys, AutoHotkey), jogos, anti-cheat e área de trabalho remota.
6. **Como detectar a remoção do hook.** Não há API, e nenhum app de referência tem vigia.
7. **`getUserMedia` com janela sem foco** no WebView2. O OpenWhispr só comprova o caso do Electron.
8. **`uiohook-napi`:** compatibilidade e binários pré-compilados para a versão atual do Electron.
9. **Wispr Flow:** a página do guia inicial só foi vista em trecho de busca (a leitura direta deu 404) e o comportamento citado vem da documentação do próprio fornecedor.
10. **Tamanho instalado** do Electron, porque o valor acima é do zip. Também falta um "hello world" dos dois frameworks como piso comparável.
