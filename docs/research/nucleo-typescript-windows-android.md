# Como compartilhar um núcleo TypeScript entre Windows e Android

Pesquisa do ticket [#5](https://github.com/Marcio-RF/my-whisper/issues/5), feita em 2026-11-08. Este arquivo reúne fatos e trade-offs. A decisão fica para o ticket [#8, Arquitetura e stack do monorepo](https://github.com/Marcio-RF/my-whisper/issues/8).

Nada foi instalado, compilado ou executado. Código de terceiros foi lido pela API do GitHub. Os tamanhos vêm de metadados do Maven (`.module` e cabeçalho HTTP), sem baixar os binários.

## Resumo

- O problema central se confirma: o Ditado no Android roda em serviço e sobreposição, sem Activity. O que decide cada opção é se o TypeScript consegue rodar nesse contexto.
- **React Native** roda sem Activity, pelo Headless JS. **Capacitor** não: a ponte dele exige uma Activity. Um **motor embutido no Kotlin** roda em qualquer serviço. Na opção de **escrever duas vezes** a pergunta não se aplica.
- O ticket não listava, mas existe uma terceira variante de motor embutido: a biblioteca Jetpack **JavaScriptEngine** do Google, estável, documentada para uso em `Service` e quase sem peso no APK, porque usa o V8 do WebView do aparelho.
- Em **todas** as opções, o serviço em primeiro plano, a sobreposição, o serviço de acessibilidade e a gravação de áudio continuam em Kotlin. O que se compartilha é a lógica do pipeline, não a parte que toca o sistema.
- Motores puros (QuickJS, JavaScriptEngine, Hermes sem React Native) não trazem `fetch` pronto. Um núcleo que precise rodar neles tem de receber HTTP, armazenamento e relógio de fora. O `enhancer.ts` do TypeLate importa `fetch` de `@tauri-apps/plugin-http` e por isso não é portátil como está.

## Comparação

| Opção | Roda sem Activity? | O que fica compartilhado | Turborepo e Gradle | Peso no APK | Risco para uma pessoa |
|---|---|---|---|---|---|
| React Native | Sim, via `HeadlessJsTaskService` | Código TS do núcleo. Telas não, salvo adotar RN também no Windows | Gradle chama o Metro; Metro precisa enxergar o pacote do núcleo | O maior: dezenas de MB em AAR; app vazio na casa de 15 a 19 MB (estimativa) | Alto: atualizações a cada ~2 meses, duas toolchains |
| Capacitor | **Não** na ponte principal. Só via Background Runner (QuickJS, limitado) | Núcleo e telas com app aberto; o Ditado em serviço fica de fora | `cap sync` é um passo separado do Gradle | Baixo: ~185 KB mais assets web e AppCompat | Médio: a opção vira dois runtimes no Android |
| Kotlin + QuickJS (`quickjs-kt`) | Sim, é biblioteca nativa no processo | Lógica pura do núcleo em TS. Telas não | Simples: bundle único copiado para `assets` | ~1,8 MB em AAR | Médio: fronteira Kotlin-JS e binding de um mantenedor |
| Kotlin + Jetpack JavaScriptEngine | Sim, documentado para `Service` | Igual ao QuickJS | Igual ao QuickJS | ~105 KB | Médio: recursos dependem do WebView do aparelho |
| Kotlin + Hermes puro | Provavelmente, mas sem caminho oficial | Igual ao QuickJS | Exige `hermesc` e camada JNI em C++ | Grande: ~85 MB em AAR | Alto: C++ próprio, sem guia de embutimento |
| Só especificação, prompts e casos de teste | Não se aplica | Prompts, contratos e casos de teste. Nenhum código | Trivial: pacote só de dados | Zero | Médio-alto: toda mudança é feita duas vezes e diverge |

Os tamanhos de AAR não equivalem ao acréscimo no APK. Veja [o que não foi verificado](#o-que-não-foi-verificado).

## O que vale para todas as opções

**Parte nativa inevitável no Android.** O TypeLate Android, que resolve o mesmo problema, usa `startForeground` e `TYPE_APPLICATION_OVERLAY` em Kotlin no [`FloatingOverlayService.kt`](https://github.com/bobo52310/TypeLate-android/blob/main/app/src/main/kotlin/com/typelate/android/overlay/FloatingOverlayService.kt), mais um [`TypeLateAccessibilityService.kt`](https://github.com/bobo52310/TypeLate-android/blob/main/app/src/main/kotlin/com/typelate/android/accessibility/TypeLateAccessibilityService.kt) e um [`VoiceRecorder.kt`](https://github.com/bobo52310/TypeLate-android/blob/main/app/src/main/kotlin/com/typelate/android/ime/VoiceRecorder.kt). Nenhuma das opções abaixo substitui isso.

**O que cabe no núcleo.** Limpeza (prompt e pós-processamento), aprendizado do dicionário, contrato de configurações, montagem das requisições à Groq e interpretação das respostas são lógica pura. Chamada HTTP, armazenamento e áudio dependem do hospedeiro. Para rodar em motor puro, o núcleo precisa receber essas capacidades de fora. Isso também evita passar o áudio pela fronteira Kotlin-JS: o Kotlin envia o arquivo e devolve o texto ao núcleo.

**Lado Windows.** Em Tauri, o núcleo seria um pacote do workspace importado pelo frontend React, sem nenhum motor adicional. O código do TypeLate mostra o ponto de atenção: [`enhancer.ts`](https://github.com/bobo52310/TypeLate/blob/main/src/lib/enhancer.ts) importa o `fetch` do plugin HTTP do Tauri.

**Turborepo e projetos não JavaScript.** A [documentação de múltiplas linguagens](https://turborepo.dev/docs/guides/multi-language) usa .NET como exemplo: cada projeto ganha um `package.json` cujos scripts chamam a ferramenta nativa, e as dependências são declaradas nos dois lados, pois o Turborepo "does not read `.csproj` references". As integrações nativas experimentais citadas são Rust, Python e Go; Gradle não aparece. Aplicar o mesmo modelo ao Gradle (um `package.json` em `apps/android` chamando `gradlew`, com `outputs` e `dependsOn: ["^build"]`) é uma inferência minha.

**Colocar o bundle no APK.** Para as opções sem Metro, a API do Android Gradle Plugin oferece [`addGeneratedSourceDirectory`](https://developer.android.com/reference/tools/gradle-api/8.3/null/com/android/build/api/variant/SourceDirectories) nas fontes da variante, e o guia [Write Gradle plugins](https://developer.android.com/build/extend-agp) mostra o padrão. O exemplo específico de assets gerados não foi lido.

## Opção 1: React Native

**O que fica compartilhado.** O código TS do núcleo roda no Hermes no Android e no WebView2 no Windows. Telas não são compartilhadas se o Windows ficar em Tauri com React para web: as primitivas de interface são outras. O [react-native-windows](https://github.com/microsoft/react-native-windows) segue ativo, mas adotá-lo é outra decisão, do ticket #8.

**Execução sem tela.**
- A [documentação do Headless JS](https://reactnative.dev/docs/headless-js-android) (0.88) descreve iniciar um serviço, com exemplos a partir de `BroadcastReceiver`. A tarefa não pode tocar na interface.
- O código de [`HeadlessJsTaskService`](https://github.com/facebook/react-native/blob/main/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/HeadlessJsTaskService.kt) confirma o ponto que importa: é uma subclasse de `Service` e, na Nova Arquitetura, chama `reactHost.start()` sem nenhuma Activity. A documentação em si não menciona a Nova Arquitetura, então o código é a fonte aqui.
- A Nova Arquitetura é a única desde a [0.82](https://reactnative.dev/blog/2025/10/08/react-native-0.82), e o Hermes V1 é o padrão desde a [0.84](https://reactnative.dev/blog/2026/02/11/react-native-0.84).
- Os temporizadores JS seguem ativos enquanto há tarefa rodando ([`JavaTimerManager`](https://github.com/facebook/react-native/blob/main/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/modules/core/JavaTimerManager.kt)).

**Armadilhas lidas no código.**
- [`HeadlessJsTaskContext`](https://github.com/facebook/react-native/blob/main/packages/react-native/ReactAndroid/src/main/java/com/facebook/react/jstasks/HeadlessJsTaskContext.kt) lança exceção se a tarefa começa com o app em `RESUMED` e sem `isAllowedInForeground`. O Ditado em cima da própria tela de configurações cairia nesse caso se a flag não fosse ligada.
- O serviço chama `stopSelf()` quando as tarefas acabam, então o serviço em primeiro plano do Ditado teria de ser controlado à parte.
- `startTask` exige a thread principal e adquire um wake lock sem tempo limite, liberado em `onDestroy`.
- O timeout só é aplicado se for maior que zero.

**Disparar pelo serviço.** Um app com serviço em primeiro plano [conta como app em primeiro plano](https://developer.android.com/about/versions/oreo/background) para os limites de início de serviço, o que permite iniciar o serviço de tarefa a partir do serviço do Ditado.

**Turborepo e Gradle.**
- O [plugin Gradle do RN](https://reactnative.dev/docs/react-native-gradle-plugin) expõe `root`, `reactNativeDir`, `codegenDir` e `cliFile`, e a própria página diz para ajustar em monorepos.
- Nas variantes release, o plugin gera o bundle em `createBundle<Variant>JsAndAssets` e roda o `hermesc`. O encadeamento fica Turborepo → Gradle → Metro.
- O guia de [integração com app existente](https://reactnative.dev/docs/integration-with-existing-apps) assume `android/` logo abaixo da raiz do projeto RN.
- O [guia de monorepos do Expo](https://docs.expo.dev/guides/monorepos/) configura o Metro automaticamente desde o SDK 52 e aceita dependências isoladas do pnpm desde o SDK 54. Avisa que caminhos fixos para `react-native` no Gradle podem quebrar com hoisting e que versões duplicadas de `react-native` não são suportadas.

**Peso no APK.** O [anúncio do `libreactnative.so`](https://github.com/react-native-community/discussions-and-proposals/discussions/816) diz que um app vazio caiu "~3.8MB (20%)" na 0.76, o que sugere cerca de 19 MB antes e 15 MB depois (conta minha; a publicação não informa ABI nem formato). No Maven, `react-android` 0.88.0 release tem 171,6 MB e `hermes-android` 85,5 MB em AAR.

**Risco de manutenção.** É a opção com mais superfície: lançamentos a cada dois meses aproximadamente ([0.88.0](https://github.com/facebook/react-native/releases/tag/v0.88.0) em 15/10/2026, [0.89.0-rc.0](https://github.com/facebook/react-native/releases/tag/v0.89.0-rc.0) em 04/11/2026), Metro, codegen, autolinking, e o [Hermes precisa casar com a versão do RN](https://github.com/facebook/hermes/blob/main/README.md). Como a parte nativa continua em Kotlin, o RN soma às duas toolchains em vez de substituir uma.

## Opção 2: Capacitor

**O que fica compartilhado.** Com o app aberto, núcleo e telas React podem ser os mesmos do Windows, o maior reaproveitamento de interface entre as opções. O Ditado em serviço fica fora disso.

**Execução sem tela.**
- [`Bridge.java`](https://github.com/ionic-team/capacitor/blob/main/android/capacitor/src/main/java/com/getcapacitor/Bridge.java) guarda um `AppCompatActivity` e o `Bridge.Builder` só aceita `AppCompatActivity` ou `Fragment`. A inicialização acontece em [`BridgeActivity.onCreate`](https://github.com/ionic-team/capacitor/blob/main/android/capacitor/src/main/java/com/getcapacitor/BridgeActivity.java).
- O [construtor de `WebView`](https://github.com/aosp-mirror/platform_frameworks_base/blob/main/core/java/android/webkit/WebView.java) pede contexto de Activity e avisa que, com contexto de aplicação, recursos como diálogos JS e autofill deixam de funcionar.
- A saída oficial é o [`@capacitor/background-runner`](https://capacitorjs.com/docs/apis/background-runner), que no Android [embute QuickJS](https://github.com/ionic-team/capacitor-background-runner/tree/main/packages/android-js-engine) e agenda por [WorkManager](https://github.com/ionic-team/capacitor-background-runner/blob/main/packages/capacitor-plugin/android/src/main/java/io/ionic/backgroundrunner/plugin/BackgroundRunner.kt). É um contexto JS separado do WebView.

**Limites do Background Runner, pela documentação.** Cada `dispatchEvent` cria um contexto novo, sem estado entre chamadas. O `fetch` não tem objeto `Request` e aceita só `method`, `headers` e `body`. Tarefas repetidas têm intervalo mínimo de 15 minutos. O teto no Android é de 10 minutos, com recomendação de no máximo 30 segundos. Na prática, isso equivale a um motor embutido (opção 3) com menos controle. Chamá-lo a partir de um serviço Kotlin próprio usaria `BackgroundRunner.execute`, método público mas não documentado como API.

**Turborepo e Gradle.** O [fluxo oficial](https://capacitorjs.com/docs/basics/workflow) é build web, depois `npx cap sync`, um passo de CLI fora do Gradle. Código nativo próprio entra como [plugin registrado na `MainActivity`](https://capacitorjs.com/docs/android/custom-code).

**Peso no APK.** O AAR `com.capacitorjs:core` 8.6.0 tem cerca de 185 KB. Somam-se dependências AndroidX e os assets web. O WebView é do sistema.

**Risco de manutenção.** O framework em si é pequeno, mas o Android ficaria com dois ambientes JS (WebView para telas, outro para o Ditado), além da parte nativa em Kotlin, e o caminho principal do produto seria justamente o que o Capacitor não cobre.

## Opção 3: app Kotlin com motor JavaScript

**O que fica compartilhado nas três variantes.** A lógica pura do núcleo em TS, gerada como um único bundle. HTTP e áudio ficam em Kotlin, atrás de uma interface. As telas do Android seriam Kotlin, sem reaproveitar as do Windows.

**Turborepo e Gradle.** É o encaixe mais simples: `packages/core` gera um arquivo (com esbuild, por exemplo) e o Gradle o coloca em `assets`. Não há Metro nem plugin do RN.

**Custo comum.** Uma fronteira serializada entre Kotlin e TS, cujos tipos não atravessam sozinhos, e depuração menos confortável que em um app JS.

### 3a. QuickJS via `quickjs-kt`

- O [QuickJS](https://bellard.org/quickjs/) cobre ES2025, tem licença MIT e última versão de 04/06/2026. A página cita 367 KiB de código x86 em um "hello world".
- O [`quickjs-kt`](https://github.com/dokar3/quickjs-kt) (Apache-2.0, 158 estrelas, v1.0.18 em 07/11/2026) expõe funções Kotlin ao JS, inclusive `suspend` via `asyncFunction`, com cancelamento, timeout e compilação para bytecode. O bytecode é específico da versão do motor.
- Não traz `fetch`: os exemplos definem o próprio.
- Roda no processo do app, sem componente Android envolvido, portanto em qualquer serviço (leitura da API, não testado).
- Há artefato JVM para testes unitários sem aparelho.
- O AAR `quickjs-release` tem 1,8 MB.
- Risco: binding mantido por uma pessoa. O motor e o binding são de terceiros, ainda que o binding possa ser trocado, pois o núcleo não depende dele.

### 3b. Jetpack JavaScriptEngine (não listado no ticket)

- A [documentação oficial](https://developer.android.com/develop/ui/views/layout/webapps/jsengine) lista como vantagem rodar em `Service` e sugere um único `Service` encapsulando a avaliação de JS. A [referência](https://developer.android.com/reference/androidx/javascriptengine/JavaScriptSandbox) recomenda contexto de aplicação quando a conexão precisa durar mais que um componente.
- Está [estável, na 1.1.1](https://developer.android.com/jetpack/androidx/releases/javascriptengine) (23/09/2026). O AAR tem cerca de 105 KB, porque o motor vem do WebView instalado.
- O JS roda em processo isolado. Um crash do sandbox não derruba o app, mas exige tratamento.
- Requer API 26 ou superior **e** um WebView que suporte o recurso, verificado com `isSupported()`. Recursos como retorno de Promise, `ArrayBuffer` e portas de mensagem dependem da versão do WebView e são checados um a um. A API de portas de mensagem, que seria a forma de o JS chamar o Kotlin, é recente (`JS_FEATURE_MESSAGE_PORTS`, na 1.1.1).
- Só uma instância de sandbox por app, com alocação descrita como "fairly expensive".
- A documentação não descreve `fetch` nem acesso à rede; o núcleo precisaria de HTTP vindo do Kotlin.
- Risco: comportamento varia conforme o WebView do aparelho, e seria preciso um plano para aparelhos sem suporte.

### 3c. Hermes sem React Native

- O [README](https://github.com/facebook/hermes/blob/main/README.md) define o Hermes como motor otimizado para React Native e não traz guia de embutimento avulso.
- A árvore do repositório só tem Java ou Kotlin em `android/` para build e testes (`intltest`). A integração seria em C++ via JSI, com JNI próprio.
- O único projeto encontrado com esse objetivo é o [android-hermes](https://awesome.ecosyste.ms/projects/github.com%2Fandrew-chen-wang%2Fandroid-hermes), sem estrelas.
- O AAR tem 85,5 MB e o build exigiria `hermesc`.
- A lista de [recursos suportados](https://github.com/facebook/hermes/blob/main/doc/Features.md) ainda fala em ES6 com exceções, o que pode estar desatualizado frente ao Hermes V1. Na prática, seria preciso transpilar o bundle.

## Opção 4: só especificação, prompts e casos de teste

**O que fica compartilhado.** Nenhum código. Prompts e contratos ficariam em arquivos de dados do monorepo, consumidos no build pelos dois lados. Casos de teste de entrada e saída rodariam em Vitest e em JUnit.

**Evidência do TypeLate.** O projeto fez exatamente isso sem compartilhar nada, e dá para ver a divergência. O [`TextEnhancer.kt`](https://github.com/bobo52310/TypeLate-android/blob/main/app/src/main/kotlin/com/typelate/android/ime/TextEnhancer.kt) se declara espelho do `enhancer.ts`, "with the same prompt and parameters", mas:

| Aspecto | Android | Desktop |
|---|---|---|
| Timeout | 10 s | 5 s |
| Prompt | Fixo em zh-TW | Por idioma |
| Vocabulário e texto ao redor no prompt | Não | Sim (`<vocabulary>`, `<surrounding_text>`) |
| Extração de vocabulário, por nome de arquivo | Sem equivalente na árvore | [`textVocabularyExtractor.ts`](https://github.com/bobo52310/TypeLate/blob/main/src/lib/textVocabularyExtractor.ts), [`vocabularyAnalyzer.ts`](https://github.com/bobo52310/TypeLate/blob/main/src/lib/vocabularyAnalyzer.ts) |

O dicionário que aprende, justamente o item previsto para o núcleo, não aparece no Android do TypeLate.

**Turborepo e Gradle.** Pacote só de dados. **Peso no APK:** nenhum. **Risco:** nenhum risco de runtime, mas cada mudança na Limpeza ou no dicionário é feita duas vezes, em duas linguagens, e só os casos de teste seguram a divergência. Contraria em parte a preferência por TypeScript, já que o núcleo do Android seria Kotlin.

**Combinação possível.** Casos de teste e prompts como dados também servem à opção 3, então as duas não são excludentes.

## O que não foi verificado

- **Acréscimo real no APK.** Os tamanhos de AAR provavelmente incluem várias ABIs e, no caso do RN, símbolos; os arquivos não foram abertos. Só um build em arm64-v8a daria o número real, em cada opção.
- **Tempo de partida a frio.** Quanto leva `reactHost.start()`, ou criar o sandbox do JavaScriptEngine, entre o toque no FAB e a primeira chamada. A aquisição do sandbox é descrita como cara, sem números.
- **Headless JS como caminho principal.** Nada foi executado: a conclusão vem da leitura do código, inclusive sobre uso a partir de um serviço de acessibilidade.
- **Ausência de `fetch` no JavaScriptEngine.** Inferida do silêncio da documentação e do processo isolado.
- **Disponibilidade do JavaScriptEngine e de suas portas de mensagem** na base real de aparelhos. A fonte não dá percentuais.
- **WebView sem Activity em um serviço**, fora do Capacitor. Só se confirmou a advertência do construtor.
- **Telas React Native dentro da sobreposição.** Não pesquisado; todas as opções assumem sobreposição em Kotlin.
- **Build do RN em monorepo pnpm sem Expo.** O guia lido é do Expo; a configuração do Metro com a CLI pura não foi conferida.
- **Compatibilidade ES do Hermes V1**, por causa do documento possivelmente desatualizado.

## Fontes

Primárias, além das já ligadas no texto:

- [Headless JS, React Native 0.88](https://reactnative.dev/docs/headless-js-android)
- [Hermes em React Native](https://reactnative.dev/docs/hermes)
- [Sobre a Nova Arquitetura](https://reactnative.dev/docs/the-new-architecture/landing-page)
- [Lançamento do React Native 0.76](https://reactnative.dev/blog/2024/10/23/release-0.76-new-architecture)
- [Metadados Maven de `react-android` 0.88.0](https://repo1.maven.org/maven2/com/facebook/react/react-android/0.88.0/react-android-0.88.0.module)
- [Metadados Maven de `hermes-android`](https://repo1.maven.org/maven2/com/facebook/hermes/hermes-android/260318099.0.7/hermes-android-260318099.0.7.module)
- [Metadados Maven de `quickjs-kt-android` 1.0.18](https://repo1.maven.org/maven2/io/github/dokar3/quickjs-kt-android/1.0.18/quickjs-kt-android-1.0.18.module)
- [AAR do JavaScriptEngine no Maven do Google](https://dl.google.com/android/maven2/androidx/javascriptengine/javascriptengine/1.1.1/javascriptengine-1.1.1.aar)
- [AAR de `com.capacitorjs:core` 8.6.0](https://repo1.maven.org/maven2/com/capacitorjs/core/8.6.0/core-8.6.0.aar)

Código de referência: [TypeLate](https://github.com/bobo52310/TypeLate) e [TypeLate Android](https://github.com/bobo52310/TypeLate-android).
