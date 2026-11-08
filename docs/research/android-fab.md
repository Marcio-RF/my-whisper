# Como pôr um FAB sobre qualquer app e inserir texto no Android

Pesquisa do ticket #2 (mapa #1), feita em 2026-11-08. Reúne fatos e restrições; as decisões de produto ficam para os tickets seguintes.

Cada afirmação traz o grau de certeza:

- **[Doc]**: documentação ou política oficial do Google.
- **[AOSP]**: lido no código-fonte do Android. O espelho `aosp-mirror/platform_frameworks_base` foi arquivado em 2025-11-12, então pode não refletir o Android 17. O serviço de ECM foi lido em `android.googlesource.com` (`main`).
- **[Ref]**: lido no código de TypeLate ou Fluence. Mostra o que os projetos fazem, não que funcione em todo aparelho.
- **[Inferência]**: conclusão minha a partir das fontes. Precisa de teste em aparelho.

## Resumo

1. O FAB como pedido (sobre qualquer app, só com o teclado do próprio usuário aberto) depende de um **Serviço de Acessibilidade**. É a única API pública encontrada que enxerga a janela do teclado de outros apps.
2. Ele é tecnicamente viável, e o Fluence já faz quase exatamente isso. O problema está na **distribuição**: um APK baixado pelo navegador que declara Acessibilidade cai em até três barreiras, e a primeira é o bloqueio de instalação pelo Play Protect, anunciado para o Brasil.
3. A inserção de texto com melhor encaixe é `commitText` da API `InputMethod` de Acessibilidade (API 33+). Nenhum dos dois projetos de referência a usa; ambos usam `ACTION_PASTE` e/ou `ACTION_SET_TEXT`.
4. Há um caminho sem Acessibilidade: um **teclado próprio** (IME). Ele não sofre essas barreiras de Acessibilidade, mas substitui o teclado do usuário, ou seja, não é o FAB pedido.

## 1. Detectar a abertura do teclado

| Caminho | Como detecta | O que exige | Limites |
|---|---|---|---|
| Serviço de Acessibilidade | Evento `typeWindowsChanged` e busca, em `getWindows()`, de uma janela `TYPE_INPUT_METHOD` | Usuário ativa o serviço nas configurações; `flagRetrieveInteractiveWindows`; barreiras da seção 4 | Único que funciona com qualquer teclado |
| Teclado próprio (`InputMethodService`) | O próprio app é o teclado e recebe o ciclo de vida | Usuário ativa e seleciona o teclado | Só vale enquanto ele for o teclado ativo |
| Só sobreposição (`SYSTEM_ALERT_WINDOW`) | Nenhuma API documentada | Permissão especial | Não encontrei forma suportada de saber do teclado de outro app |

- `AccessibilityWindowInfo.TYPE_INPUT_METHOD` existe desde a API 21 e descreve uma janela de teclado; `getBoundsInScreen` dá a posição dela. [Doc: [AccessibilityWindowInfo](https://developer.android.com/reference/android/view/accessibility/AccessibilityWindowInfo)]
- A flag `FLAG_RETRIEVE_INTERACTIVE_WINDOWS` pede acesso a todas as janelas interativas. [Doc: [AccessibilityServiceInfo](https://developer.android.com/reference/android/accessibilityservice/AccessibilityServiceInfo)]
- O Fluence faz essa varredura em `isImeVisible()`. [Ref: [FluenceAccessibilityService.kt#L365](https://github.com/raviumeshkulkarni-web/Fluence/blob/7cab4ff4beb233cc622c41b835688781f3e4feee/app/src/main/java/com/groq/voicetyper/FluenceAccessibilityService.kt#L365)]
- `InputMethod.onStartInput` e `onFinishInput` (API 33) avisam quando a entrada começa e termina em um editor. Isso indica foco de edição, não que o teclado esteja visível. [Doc: [InputMethod](https://developer.android.com/reference/android/accessibilityservice/InputMethod); a distinção é Inferência]

### Ordem das janelas: o FAB pode ficar por cima do teclado?

- `TYPE_APPLICATION_OVERLAY` fica "below critical system windows like the status bar or IME" e exige `SYSTEM_ALERT_WINDOW`. [Doc: [WindowManager.LayoutParams](https://developer.android.com/reference/android/view/WindowManager.LayoutParams)]
- Camadas no código: sobreposição de app 11, teclado 13, sobreposição de acessibilidade 31. [AOSP: [WindowManagerPolicy.java](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/services/core/java/com/android/server/policy/WindowManagerPolicy.java)]
- Portanto, para o FAB ficar **sobre** a área do teclado, é preciso `TYPE_ACCESSIBILITY_OVERLAY`. Com `TYPE_APPLICATION_OVERLAY`, ele precisa ficar acima da borda do teclado. [Inferência]
- A documentação só exige `SYSTEM_ALERT_WINDOW` para `TYPE_APPLICATION_OVERLAY`. Um serviço de acessibilidade poderia desenhar seu FAB sem essa permissão. [Inferência; o Fluence pede as duas]

## 2. Sobreposição e microfone em serviço em primeiro plano

**Regras documentadas:**

- `RECORD_AUDIO` é permissão *while-in-use*. Um serviço `microphone` não pode ser criado com o app em segundo plano, salvo exceções. O início precisa ocorrer com uma activity visível. [Doc: [restrições de início](https://developer.android.com/develop/background-work/services/fgs/restrictions-bg-start), [tipos de serviço](https://developer.android.com/develop/background-work/services/fgs/service-types)]
- As exceções *while-in-use* listadas são: componente do sistema, widget, notificação, `PendingIntent` de app visível, DPC em device owner, `VoiceInteractionService` e `START_ACTIVITIES_FROM_BACKGROUND`. **Nem sobreposição nem Acessibilidade aparecem na lista.** [Doc: mesma página]
- `SYSTEM_ALERT_WINDOW` isenta apenas a restrição geral de início em segundo plano. Com `targetSdk` 35+, exige também uma janela de sobreposição visível. Essa isenção não cobre o microfone. [Doc: mesma página]
- Android 14+: é obrigatório declarar o tipo `microphone` e a permissão `FOREGROUND_SERVICE_MICROPHONE`; no Google Play, é preciso declarar o tipo no Console. [Doc: [tipos de serviço](https://developer.android.com/develop/background-work/services/fgs/service-types)]
- Android 17: o endurecimento de áudio em segundo plano cobre reprodução, foco de áudio e volume. Gravação não é citada. Com `targetSdk` 37, exige serviço com capacidade *while-in-use*. [Doc: [mudanças do Android 17](https://developer.android.com/about/versions/17/behavior-changes-17)] Um som de confirmação do FAB entraria nessa regra. [Inferência]

**O código é mais permissivo que a documentação:**

- `shouldAllowFgsWhileInUsePermissionLocked` aceita um UID com janela não-app visível: `isUidForeground` → `hasActiveVisibleWindow` → `hasNonAppVisibleWindow`. [AOSP: [ActiveServices.java#L8398](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/services/core/java/com/android/server/am/ActiveServices.java#L8398), [ActivityTaskManagerService.java#L5686](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/services/core/java/com/android/server/wm/ActivityTaskManagerService.java#L5686)]
- Esse rastreio só conta tipos de `isSystemAlertWindowType`, o que inclui `TYPE_APPLICATION_OVERLAY`. [AOSP: [Session.java#L768](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/services/core/java/com/android/server/wm/Session.java#L768)]
- O serviço de acessibilidade é ligado pelo sistema com `BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS` e `BIND_INCLUDE_CAPABILITIES`. O próprio `ActiveServices` comenta que essa flag é aceita na verificação *while-in-use*. [AOSP: [AccessibilityServiceConnection.java](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/services/accessibility/java/com/android/server/accessibility/AccessibilityServiceConnection.java), [OomAdjuster.java#L2779](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/services/core/java/com/android/server/am/OomAdjuster.java#L2779)]
- Conclusão: captar áudio a partir de um FAB com Serviço de Acessibilidade conectado parece possível, mas se apoia em comportamento **não documentado como garantia**. [Inferência; ver seção 7]

## 3. Inserção de texto

| Mecanismo | Comportamento | Onde falha ou tem custo |
|---|---|---|
| `InputMethod...commitText` (API 33+, `flagInputMethodEditor`) | Insere no cursor e substitui a seleção | Sem documentação de limites por tipo de campo; não testado |
| `ACTION_SET_TEXT` | Troca **todo** o texto do nó e põe o cursor no fim; sem argumento, apaga | Precisa ler e concatenar; arriscado em campos cujo texto não dá para ler |
| `ACTION_PASTE` + área de transferência | Cola o conteúdo da área de transferência | Sobrescreve a área de transferência; aparece a UI do sistema no Android 13+ |
| Teclado próprio (`InputConnection.commitText`) | Caminho padrão de um IME | Só com o teclado próprio ativo |

Detalhes:

- `commitText` "the new text is inserted at the cursor position, removing text inside the selection if any". [Doc: [AccessibilityInputConnection](https://developer.android.com/reference/android/accessibilityservice/InputMethod.AccessibilityInputConnection)]
- `ACTION_SET_TEXT` em `TextView` só exige campo habilitado e editável. [AOSP: [TextView.java#L14987](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/core/java/android/widget/TextView.java#L14987); Doc: [AccessibilityNodeInfo](https://developer.android.com/reference/android/view/accessibility/AccessibilityNodeInfo)]
- `ACTION_PASTE` em `TextView` exige `isFocused()` e `canPaste()`. [AOSP: [TextView.java#L14934](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/core/java/android/widget/TextView.java#L14934)]
- Área de transferência: **escrever** é permitido sem foco, **ler** não. Desde o Android 10, só o app em foco ou o IME padrão leem. Por isso, restaurar o conteúdo anterior a partir de um serviço em segundo plano tende a não funcionar. [AOSP: [ClipboardService.java#L1383](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/services/core/java/com/android/server/clipboard/ClipboardService.java#L1383); Doc: [privacidade Android 10](https://developer.android.com/about/versions/10/privacy/changes); a conclusão é Inferência]
- Android 13+ mostra uma UI padrão ao copiar. [Doc: [copiar e colar](https://developer.android.com/develop/ui/views/touch-and-input/copy-paste)]

**Onde falha:**

- **WebView/Chrome:** o Chromium implementa `ACTION_SET_TEXT` apenas se `isEditableText` for verdadeiro, e `ACTION_PASTE` chama `WebContents.paste()`. [AOSP-equivalente: [WebContentsAccessibilityImpl.java](https://github.com/chromium/chromium/blob/955451908894240880d55ca07b15dd7c5620e036/content/public/android/java/src/org/chromium/content/browser/accessibility/WebContentsAccessibilityImpl.java)] O Fluence registra que nós de WebView e `contenteditable` podem expor `SET_TEXT` sem `isEditable`, e evita `SET_TEXT` nesses casos. [Ref: [BubbleController.kt#L687](https://github.com/raviumeshkulkarni-web/Fluence/blob/7cab4ff4beb233cc622c41b835688781f3e4feee/app/src/main/java/com/groq/voicetyper/BubbleController.kt#L687)]
- **Senhas:** o `TextView` não bloqueia `SET_TEXT` nem `canPaste` para senhas ([TextView.java#L15670](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/core/java/android/widget/TextView.java#L15670)), e o texto exposto à acessibilidade é o transformado, isto é, mascarado ([L15111](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/core/java/android/widget/TextView.java#L15111)). Ler e concatenar não funciona nesses campos. [AOSP; Inferência] `isPassword` permite detectá-los: o Fluence recusa inserir neles ([Ref: FluenceAccessibilityService.kt#L120](https://github.com/raviumeshkulkarni-web/Fluence/blob/7cab4ff4beb233cc622c41b835688781f3e4feee/app/src/main/java/com/groq/voicetyper/FluenceAccessibilityService.kt#L120)); o TypeLate não verifica.
- **Campos sensíveis (bancos, pagamentos):** com `accessibilityDataSensitive` ou `setFilterTouchesWhenObscured(true)`, um serviço sem `isAccessibilityTool=true` tem o acesso negado àquela view. [Doc: [blog Android, dez/2025](https://developer.android.com/blog/posts/enhancing-android-security-stop-malware-from-snooping-on-your-app-data)]
- **Superfície própria (jogos, Canvas, telas sem semântica):** sem nó acessível não há alvo para `SET_TEXT` ou `PASTE`. [Inferência; não testado app a app]

## 4. Política do Google Play e distribuição fora da loja

### Google Play

- Apps que não são ferramenta de acessibilidade **podem** usar a API, com divulgação em destaque dentro do app, consentimento afirmativo, formulário de declaração no Console e vídeo do fluxo. [Doc: [uso da AccessibilityService API](https://support.google.com/googleplay/android-developer/answer/10964491?hl=en)]
- `isAccessibilityTool=true` é só para apps cujo propósito principal é apoiar pessoas com deficiência. "Voice-based input systems which support people with motor impairments" qualifica; um assistente geral por voz, não. [Doc: mesma página] Declarar sem ter direito leva a rejeição no Play e bloqueio pelo Play Protect. [Doc: [blog Android](https://developer.android.com/blog/posts/enhancing-android-security-stop-malware-from-snooping-on-your-app-data)]
- A política pede APIs de escopo mais estreito em vez de Acessibilidade "when possible". [Doc: [permissões e APIs sensíveis](https://support.google.com/googleplay/android-developer/answer/16558241?hl=en)] Como existe a alternativa do teclado próprio, a aprovação de um ditado com FAB é incerta. [Inferência]
- Com `isAccessibilityTool=false`, o sistema mostra, depois de um tempo, uma notificação sobre as implicações de privacidade. [Doc: [AccessibilityServiceInfo](https://developer.android.com/reference/android/accessibilityservice/AccessibilityServiceInfo)]

### Barreiras a um APK instalado fora da loja

**(a) Bloqueio de instalação pelo Play Protect (a mais grave para o Brasil)**

- Apps de "Internet-sideloading sources" (navegadores, apps de mensagem, gerenciadores de arquivos) que declaram `ACCESSIBILITY` são considerados de alto risco e têm a instalação bloqueada, "in select markets". [Doc: [orientação do Play Protect](https://developers.google.com/android/play-protect/warning-dev-guidance)]
- O Google anunciou esse bloqueio explicitamente para o Brasil em junho de 2024, como piloto, e citou Acessibilidade. [Doc: [blog Google Brasil](https://blog.google/intl/pt-br/produtos/android-chrome-play/android-protegendo-os-brasileiros-contra-roubo-de-celular-e-fraude/)]
- Em fevereiro de 2026, o Google disse que a proteção alcançava 185 mercados. O post não lista os países. [Doc: [blog Google](https://blog.google/security/keeping-google-play-android-app-ecosystem-safe-2025/)]
- O usuário pode desativar a verificação do Play Protect; não achei "instalar mesmo assim" para esse bloqueio. [Doc: [ajuda Play Protect](https://support.google.com/android/answer/2812853?hl=pt-br)] O desenvolvedor pode apresentar uma contestação. [Doc: orientação acima]

**(b) Configurações restritas (Android 13+)**

- O acesso à Acessibilidade fica bloqueado até o usuário abrir *Informações do app → menu → Permitir configurações restritas*. [Doc: [ajuda Android](https://support.google.com/android/answer/12623953?hl=en)]
- O bloqueio é aplicado a instalações de origem `PACKAGE_SOURCE_LOCAL_FILE` ou `DOWNLOADED_FILE`. [AOSP: [InstallPackageHelper.java](https://github.com/aosp-mirror/platform_frameworks_base/blob/1cdfff555f4a21f71ccc978290e2e212e2f8b168/services/core/java/com/android/server/pm/InstallPackageHelper.java)]
- No ECM atual, as configurações protegidas incluem **`BIND_ACCESSIBILITY_SERVICE` e `SYSTEM_ALERT_WINDOW`**. Ativar um teclado próprio não aparece na lista. [AOSP: [EnhancedConfirmationService.java](https://android.googlesource.com/platform/packages/modules/Permission/+/refs/heads/main/service/java/com/android/ecm/EnhancedConfirmationService.java)]

**(c) Verificação de desenvolvedor**

- Calendário publicado: a exigência começa em 30/09/2026 em Brasil, Indonésia, Singapura e Tailândia, **só para instalações vindas de sete lojas participantes**. Sideload e outras lojas seguem sem exigência nessa fase. Expansão global em 2027, em aparelhos certificados com Android 7+. [Doc: [guia](https://developer.android.com/developer-verification/guides), [FAQ](https://developer.android.com/developer-verification/guides/faq)]
- Três caminhos a partir da expansão global: [Doc: [guia](https://developer.android.com/developer-verification/guides), [FAQ](https://developer.android.com/developer-verification/guides/faq)]
  - **Distribuição completa:** verificação de identidade, conta de US$ 25, instalação normal.
  - **Distribuição limitada:** sem documento e sem taxa, até 20 aparelhos.
  - **Sem registro:** o usuário passa pelo *advanced flow* (modo desenvolvedor, reinício, espera de 24 h, biometria ou PIN) e ainda vê um aviso. Atualizações falham se o fluxo for desativado. Instalação via ADB segue livre.
- O registro não aparece no código de ECM como critério, então registrar não elimina (b). [AOSP; Inferência] Se o registro afeta (a), não está documentado.

**(d) Modo de Proteção Avançada**

- Com o modo ligado, o Android bloqueia instalação e atualização de fontes desconhecidas e restringe Acessibilidade a ferramentas verificadas. [Doc: [ajuda Android](https://support.google.com/android/answer/16339980?hl=en)] O post sobre o Android 17 reforça a remoção do acesso para apps não rotulados como ferramenta. [Doc: [blog Google](https://blog.google/security/whats-new-in-android-security-privacy-2026/)] Nesses aparelhos, o FAB não funciona. É um modo opcional, voltado a usuários de risco.

### F-Droid

- A política de inclusão exige software livre e não menciona Acessibilidade. [Doc: [Inclusion Policy](https://f-droid.org/docs/Inclusion_Policy/)]
- Um app que "depend entirely on a proprietary network service" recebe o antirrecurso **NonFreeNet**, o que indica o caso do Groq. O antirrecurso *Tethered Network Services* não se aplica se for possível apontar para outro servidor por configuração simples. [Doc: [Anti-Features](https://f-droid.org/docs/Anti-Features/); o enquadramento é Inferência]
- O cliente F-Droid instala por sessão e declara `PACKAGE_SOURCE_STORE`. [Ref: [SessionInstallManager.kt](https://github.com/f-droid/fdroidclient/blob/5809dc397da52b66725e87355bc4918d6f519d36/app/src/main/kotlin/org/fdroid/install/SessionInstallManager.kt)] Assim, escapa do critério direto de (b). Ainda assim, o ECM consulta uma lista de instaladores confiáveis, e se ela não estiver vazia o app volta a ser protegido. [AOSP: EnhancedConfirmationService; o resultado em aparelho não foi verificado]
- O F-Droid é contra a verificação e declarou que ela "will end the F-Droid project and other free/open-source app distribution sources as we know them today". [Doc: [F-Droid, set/2025](https://f-droid.org/2025/09/29/google-developer-registration-decree.html)] Mesmo assim, o Google publicou um guia para registrar apps em repositórios abertos, inclusive os reassinados com a chave do repositório, o que ainda exige conta verificada do desenvolvedor. [Doc: [guia open source](https://developer.android.com/developer-verification/guides/open-source-app-registration)]

## 5. Menor versão de Android viável

| Piso | O que determina | Observação |
|---|---|---|
| API 22 | `TYPE_ACCESSIBILITY_OVERLAY` | Sem `TYPE_APPLICATION_OVERLAY` |
| API 26 | `TYPE_APPLICATION_OVERLAY` | `minSdk` de TypeLate e Fluence |
| API 33 | `InputMethod.commitText` | Abaixo disso, só `ACTION_SET_TEXT`/`PASTE` |

Em outubro de 2026, os números do Android Studio repassados pela imprensa indicavam cerca de 56,4% em Android 14+ e 11,8% em Android 13, ou seja, perto de 68% em API 33+. [Fonte secundária: [Beebom](https://gadgets.beebom.com/news/google-android-17-adoption-figures-october-2026)]

## 6. Comparação com TypeLate e Fluence

| Tema | [TypeLate](https://github.com/bobo52310/TypeLate-android) (MIT) | [Fluence](https://github.com/raviumeshkulkarni-web/Fluence) (AGPL-3.0) |
|---|---|---|
| `minSdk` / `targetSdk` | 26 / 36 | 26 / 34 |
| Quando o botão aparece | Sobreposição iniciada pelo app, sem relação com o teclado | Ao focar campo editável; modo opcional "só com teclado visível" |
| Detecção | Nenhuma: `onAccessibilityEvent` vazio | `typeWindowsChanged` + `TYPE_INPUT_METHOD` |
| Tipo de janela | `TYPE_APPLICATION_OVERLAY` | `TYPE_ACCESSIBILITY_OVERLAY` (visual) + `TYPE_APPLICATION_OVERLAY` (toque) |
| Permissões | Acessibilidade + `SYSTEM_ALERT_WINDOW` + microfone | As mesmas, + `REQUEST_INSTALL_PACKAGES` |
| Microfone | Serviço `microphone` a partir da sobreposição | Serviço `microphone` iniciado com o FAB |
| Inserção | Só `ACTION_PASTE`; cai para copiar + aviso | `ACTION_SET_TEXT` em campo nativo; senão `ACTION_PASTE`; tenta restaurar a área de transferência |
| Senhas | Sem verificação | Recusa (`isPassword` + nome de classe) |
| `InputMethod.commitText` | Não usa | Não usa |
| Teclado próprio | Tem, como modo alternativo | Tem, como modo alternativo |

Arquivos lidos nos commits [`a875a2d`](https://github.com/bobo52310/TypeLate-android/tree/a875a2d5187714204b38a3caa31b93da319fd09a) e [`7cab4ff`](https://github.com/raviumeshkulkarni-web/Fluence/tree/7cab4ff4beb233cc622c41b835688781f3e4feee): manifestos, configuração de acessibilidade, serviços de sobreposição e de acessibilidade, `TextOutputHandler` e `BubbleController`. Nada foi baixado como projeto nem executado. A licença AGPL-3.0 do Fluence importa caso algum trecho seja reaproveitado.

## 7. O que ficou sem verificar

1. **Bloqueio do Play Protect hoje no Brasil.** O anúncio de 2024 foi de piloto; a página atual diz apenas "select markets". Precisa de teste real: instalar pelo navegador, em aparelho no Brasil, um APK que declare Acessibilidade.
2. **Se registrar o app (verificação de desenvolvedor) evita o bloqueio do Play Protect.** Não documentado.
3. **Se o cliente F-Droid (ou Obtainium) escapa do bloqueio e das configurações restritas.** Código lido, resultado em aparelho não testado.
4. **Se `SYSTEM_ALERT_WINDOW` já era restrita no Android 13 e 14.** A lista lida é do ECM atual.
5. **Microfone a partir do FAB sem activity visível.** O código permite, a documentação não lista. Precisa de teste em Android 14–17 e em fabricantes como Samsung e Xiaomi.
6. **`commitText` da Acessibilidade em WebView, Compose, Flutter e campos de senha.** Sem limite documentado, sem exemplo nos projetos de referência.
7. **Detecção por insets em sobreposição sem Acessibilidade.** Não documentada; não investigada a fundo.
8. **Se a fiscalização de 30/09/2026 de fato começou como anunciado.** Li apenas o calendário publicado.
