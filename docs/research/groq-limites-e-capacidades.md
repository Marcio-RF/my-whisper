# Groq: limites e capacidades para Ditado e Limpeza

Pesquisa do ticket #4. Fontes lidas em **2026-11-08**. Nada aqui é decisão de produto: são fatos, restrições e o que ficou sem verificar.

Nenhuma chamada foi feita à API da Groq com chave. O único contato direto com `api.groq.com` foram duas requisições **sem chave** para observar CORS e a resposta 401 (seção 6).

Legenda de confiança:

- **[doc]**: dito pela documentação da Groq.
- **[obs]**: observado diretamente nesta pesquisa.
- **[terceiro]**: relato de terceiro, não confirmado pela Groq.
- **[cálculo]**: conta ou inferência minha, com as premissas explícitas.

## 1. Modelos de transcrição, formatos e tamanho

| | `whisper-large-v3` | `whisper-large-v3-turbo` |
|---|---|---|
| Idiomas | Multilíngue ("99+") | Multilíngue |
| Transcrição / tradução | Sim / Sim | Sim / Não |
| Fator de velocidade | 189x | 216x |
| Taxa de erro de palavra publicada | 10,3% | 12% |
| Preço (plano pago) | US$ 0,111/hora | US$ 0,04/hora |

Fonte: [Speech to Text](https://console.groq.com/docs/speech-to-text) **[doc]**. A Groq não diz em que idioma ou conjunto de dados mediu a taxa de erro.

Regras de arquivo **[doc]**:

- **Formatos:** `flac`, `mp3`, `mp4`, `mpeg`, `mpga`, `m4a`, `ogg`, `wav`, `webm`, por upload ou URL.
- **Tamanho máximo:** 25 MB no plano gratuito, 100 MB no plano Developer.
- **Duração mínima:** 0,01 s.
- **Duração mínima cobrada:** 10 s (áudio mais curto é cobrado como 10 s).
- **Pré-processamento:** o áudio é reduzido para 16 kHz mono no servidor. Fazer isso no cliente reduz o arquivo sem perda de qualidade. A Groq recomenda FLAC para compressão sem perda e `wav` para menor latência.
- **Saída:** `json`, `verbose_json` (com `avg_logprob`, `compression_ratio`, `no_speech_prob` por segmento) e `text`.

Em termos práticos **[cálculo]**: 25 MB equivalem a cerca de 13 min de WAV 16 kHz mono PCM de 16 bits (32 kB/s). Formatos comprimidos rendem bem mais. O teto de 25 MB só importa em ditados muito longos.

## 2. Parâmetros `prompt`, `language` e `temperature`

### `prompt`

- Limite de **224 tokens** **[doc]** ([Speech to Text](https://console.groq.com/docs/speech-to-text)).
- Efeito: orienta estilo e contexto, e pode indicar grafias. A Groq diz que esses prompts "only guide style and context, not specific actions". Instruções não são seguidas **[doc]**.
- Boas práticas da Groq: usar o mesmo idioma do áudio, dar contexto, indicar grafias corretas, ser conciso **[doc]**.
- O guia da OpenAI confirma o uso como glossário de nomes próprios e diz que só os **últimos** 224 tokens são considerados e o resto é ignorado em silêncio ([Whisper prompting guide](https://developers.openai.com/cookbook/examples/whisper_prompting_guide)). Isso descreve a API da OpenAI. A Groq não diz se corta ou rejeita acima do limite.
- No código de referência do Whisper, o `initial_prompt` só alimenta a primeira janela de até 30 s, salvo com `carry_initial_prompt` ([transcribe.py](https://raw.githubusercontent.com/openai/whisper/main/whisper/transcribe.py)). A implementação da Groq não é pública, então não dá para afirmar o mesmo dela.
- Tamanho prático **[cálculo]**: 224 tokens são poucas dezenas a cerca de 100 termos, porque nomes incomuns consomem vários tokens cada.

### `language`

- Opcional, **um único** código ISO-639-1 (por exemplo `pt`). Informar melhora precisão e latência **[doc]**.
- A documentação da Groq **não descreve a detecção automática** nem mostra o idioma detectado na resposta. O parâmetro apenas é opcional.
- No código de referência do Whisper, sem `language` o idioma é detectado uma vez, nos primeiros 30 s, e vale para o arquivo inteiro ([transcribe.py](https://raw.githubusercontent.com/openai/whisper/main/whisper/transcribe.py)).

### `temperature`

- De 0 a 1, padrão 0, e a Groq recomenda manter 0 **[doc]**.
- Em 0, o modelo usa probabilidade logarítmica para aumentar a temperatura automaticamente até certos limiares **[doc]** ([API Reference](https://console.groq.com/docs/api-reference)).

## 3. Limites do plano gratuito

### O que a Groq publica

**A Groq não publica uma tabela rotulada como plano gratuito.** A situação nas fontes dela é esta:

| Fonte da Groq | O que diz | Whisper | `gpt-oss-20b` |
|---|---|---|---|
| [Rate Limits](https://console.groq.com/docs/rate-limits) **[doc]** | "base limits for the Developer plan" | 20 RPM, 2K RPD, 7,2K ASH, 28,8K ASD | 30 RPM, 1K RPD, 8K TPM, 200K TPD |
| [Models](https://console.groq.com/docs/models) **[doc]** | coluna "Rate limits (Developer plan)" | v3: 300 RPM, 200K ASH. Turbo: 400 RPM, 400K ASH | 1K RPM, 250K TPM |
| [Billing Plans](https://console.groq.com/settings/billing/plans) **[doc]** | "For the current free and developer plan rate limits, see the Rate Limits page" | remete à primeira linha | remete à primeira linha |
| Console, [Limits](https://console.groq.com/settings/limits) | valores exatos da organização | exige login | exige login |

As duas primeiras linhas se contradizem: ambas dizem ser do plano Developer e os números diferem até 55 vezes. O FAQ de cobrança confirma que existem "Free tier rate limits" diferentes, sem dar números.

### Melhor evidência dos números gratuitos

Um usuário do plano gratuito publicou, em 2026-09-21, um print da página Limits do próprio console ([TypeLate #2](https://github.com/bobo52310/TypeLate/issues/2), segunda imagem) **[terceiro]**. A página diz "On Developer plan, you get higher limits", o que indica uma conta fora do plano Developer. Os valores:

| Modelo | RPM | RPD | TPM | TPD | ASH | ASD |
|---|---|---|---|---|---|---|
| `whisper-large-v3` | 20 | 2K | | | 7,2K | 28,8K |
| `whisper-large-v3-turbo` | 20 | 2K | | | 7,2K | 28,8K |
| `openai/gpt-oss-20b` | 30 | 1K | 8K | 200K | | |
| `openai/gpt-oss-120b` | 30 | 1K | 8K | 200K | | |
| `qwen/qwen3.8-27b` | 30 | 1K | 8K | 200K | | |

Esses valores coincidem com a tabela da página Rate Limits para Whisper, GPT-OSS 20B/120B e Qwen. A leitura mais provável **[cálculo]** é que a página Rate Limits traz, na prática, os números do plano gratuito com o rótulo errado, e que a coluna da página Models é a do Developer. Isso explicaria as divergências entre fontes de terceiros.

Limites dessa conclusão: é uma única conta, o print tem cerca de 7 semanas, e a própria Groq avisa que "there may be exceptions". O print também difere da documentação em `openai/gpt-oss-safeguard-20b` (30 RPM e 8K TPM contra 3 RPM e 2K TPM). **Cada usuário pode conferir os próprios números em Limits.**

### Como os limites funcionam **[doc]**

- Valem **por organização**, não por usuário. Qualquer limite pode ser atingido primeiro. Com chave própria, cada usuário tem a própria cota, desde que sua chave seja de organização própria **[cálculo]**.
- A tabela é por modelo. As cotas parecem independentes entre modelos, mas a Groq não diz isso explicitamente **[cálculo]**.
- Ao exceder: HTTP `429`, com `retry-after` em segundos.
- Cabeçalhos `x-ratelimit-*` sempre presentes, mas só cobrem **RPD e TPM**. Nenhum cabeçalho documentado informa o saldo de **segundos de áudio**.
- Tokens em cache não contam para os limites, mas a Groq avisa que são subtraídos depois do processamento e que requisições paralelas ainda podem atingir o limite. O cache é automático nos modelos GPT-OSS, depende de prefixo idêntico e não é garantido ([Prompt Caching](https://console.groq.com/docs/prompt-caching)).

### Em termos práticos **[cálculo]**

Usando os números acima, supondo que sejam mesmo os do plano gratuito.

**Só Ditado (Whisper):**

- 28.800 s/dia são **8 horas de áudio por dia**, e 7.200 s/h são **2 horas de áudio por hora**.
- 2.000 requisições/dia e 20/min (um ditado a cada 3 s). Para ditados curtos, o que limita são as requisições: 2.000 ditados por dia.
- Mesmo que cada ditado curto consumisse 10 s da cota (não verificado, seção 7), 2.000 × 10 s = 20.000 s, abaixo do teto de 28.800 s.

**Ditado + Limpeza com um modelo GPT-OSS:** quem limita é o **TPD de 200K**, não o áudio.

Premissas: 150 palavras por minuto, 1,5 token por palavra, 400 tokens fixos (instruções e vocabulário), 100 tokens de raciocínio por chamada, e uma chamada de Limpeza por ditado. O total por chamada é 500 + 2 × tokens do texto.

| Duração do ditado | Tokens por chamada | Ditados/dia | Fala/dia (sem cache) | Fala/dia (cache perfeito) |
|---|---|---|---|---|
| 15 s | ~610 | ~325 | ~80 min | ~235 min |
| 30 s | ~725 | ~275 | ~140 min | ~310 min |
| 60 s | ~950 | ~210 | ~210 min | ~360 min |

Resumo: **de cerca de 1h20 a 6h de fala por dia**, conforme o tamanho dos ditados e o cache. Um uso de 30 min de fala por dia, em ditados de 30 s, consome cerca de 22% do TPD diário sem cache. O limite de 8K TPM cabe de 8 a 13 ditados por minuto, folgado para uma pessoa. Chamadas de aprendizado de vocabulário saem da mesma cota se usarem o mesmo modelo.

## 4. Modelos de texto disponíveis agora

Fonte: [Models](https://console.groq.com/docs/models) **[doc]**.

| Modelo | Categoria | Velocidade | Contexto / saída máxima | Preço por 1M (entrada / saída) |
|---|---|---|---|---|
| `openai/gpt-oss-20b` | Production | ~1000 tok/s | 131.072 / 65.536 | US$ 0,075 / 0,30 |
| `openai/gpt-oss-120b` | Production | ~500 tok/s | 131.072 / 65.536 | US$ 0,15 / 0,60 |
| `qwen/qwen3.8-27b` | **Preview** | ~450 tok/s | 131.072 / 16.384 | US$ 0,80 / 4,00 |
| `llama-3.1-8b-instant`, `llama-3.3-70b-versatile` | Production, marcados "Enterprise" | 560 / 280 | ver página | Contact Sales |
| `meta-models/muse-spark-open`, `minimaxai/minimax-m2.7` | Preview, marcados "Enterprise" | 350 / 260 | ver página | Contact Sales |

Atenção: os dois Llama aparecem como Production, mas foram **desligados para os planos gratuito e Developer em 2026-08-16**. Só clientes enterprise com contrato continuam ([Deprecations](https://console.groq.com/docs/deprecations)).

**Latência.** A Groq publica só tokens por segundo e fator de velocidade. Não publica tempo até o primeiro token, percentis nem garantia. Nominalmente **[cálculo]**, 30 s de áudio são processados em cerca de 0,15 s, e 300 tokens de saída levam cerca de 0,3 s no 20B e 0,6 s no 120B, sem contar rede, upload e fila.

**Raciocínio.** Os GPT-OSS sempre raciocinam: `reasoning_effort` aceita `low`, `medium` e `high`, sem opção de desligar. O conteúdo vem no campo `reasoning`, controlável por `include_reasoning`. O `qwen/qwen3.8-27b` aceita `reasoning_effort: none` ([Reasoning](https://console.groq.com/docs/reasoning)) **[doc]**.

**Qualidade em português.** A documentação da Groq **não menciona português** e não traz métrica alguma por idioma, nem para Whisper nem para modelos de texto.

**Saída estruturada** ([Structured Outputs](https://console.groq.com/docs/structured-outputs)) **[doc]**:

- `strict: true` (garante o esquema): `gpt-oss-20b`, `gpt-oss-120b`, `qwen3.8-27b`.
- `strict: false` (melhor esforço): os mesmos, mais `gpt-oss-safeguard-20b`.
- Streaming e uso de ferramentas **não** funcionam junto com Structured Outputs.

## 5. Aposentadoria de modelos e descoberta em tempo de execução

### Política **[doc]** ([Deprecations](https://console.groq.com/docs/deprecations))

- Anúncio por e-mail "to all affected users" e atualização da página de depreciações.
- Depois do desligamento: "Requests to deprecated model IDs will return errors".
- Modelos Preview podem sair "at short notice with limited advance warning".
- A Groq recomenda projetar sistemas independentes de modelo específico.
- **Não há prazo mínimo de aviso publicado.**

O e-mail vai para quem usa o modelo. Com chave própria, isso é o dono da chave, não necessariamente quem mantém o app **[cálculo]**.

### Frequência observada

Entre 2025-11-09 e 2026-11-08, **12 IDs foram desligados em 8 datas**. Avisos no histórico:

| Anúncio | Desligamento | Aviso | Modelos |
|---|---|---|---|
| 2025-12-23 | 2025-12-31 | 8 dias | `playai-tts`, `playai-tts-arabic` |
| 2026-02-10 | 2026-03-05 | 23 dias | `llama-guard-4-12b` |
| 2026-02-20 | 2026-03-09 | 17 dias | `llama-4-maverick` |
| 2026-03-23 | 2026-04-15 | 23 dias | `kimi-k2-instruct-0905` |
| 2026-06-17 | 2026-07-17 | 30 dias | `qwen3-32b`, `llama-4-scout` |
| 2026-06-17 | 2026-08-16 | 60 dias | `llama-3.1-8b-instant`, `llama-3.3-70b-versatile` |
| não informado | 2026-09-14 | ? | `qwen3.6-27b` |
| 2026-08-24 | 2026-09-21 | 28 dias | `groq/compound`, `compound-mini` |

Mesmo um substituto recomendado saiu depressa: em 2026-06-17 a Groq indicou `qwen3.6-27b`, que foi desligado em 2026-09-14.

**Whisper:** nenhum Whisper Large foi aposentado. A única saída foi `distil-whisper-large-v3-en`, em 2025-08-23.

### O caso TypeLate

Confirmado. Os quatro modelos da lista do TypeLate (Qwen3 32B, Llama 3.3 70B, Llama 4 Scout, Kimi K2) constam como desligados na página de depreciações. O erro relatado foi `404 model_not_found`, "does not exist or you do not have access to it" **[terceiro]**. Segundo o relato, o texto bruto era colado sem nenhum erro visível ([TypeLate #2](https://github.com/bobo52310/TypeLate/issues/2)).

### Descoberta em tempo de execução

- `GET https://api.groq.com/openai/v1/models` retorna a lista de modelos ativos, e `GET /openai/v1/models/{model}` retorna um modelo **[doc]** ([API Reference](https://console.groq.com/docs/api-reference)).
- O exemplo documentado traz `id`, `object`, `created`, `owned_by`, `active`, `context_window` e `public_apps`. **Não há campo de capacidade**: a lista não diz o que é modelo de texto, de áudio, se aceita `strict` etc. Esse exemplo também parece antigo, pois mostra modelos já desligados.
- A API não documenta um campo com data de aposentadoria.

## 6. Validar a chave e chamadas diretas do app

**Validação.** Não existe rota dedicada. `GET /openai/v1/models` com `Authorization: Bearer <chave>` é a chamada autenticada mais barata documentada. Sem chave, a resposta observada foi **[obs]**:

```
401 {"error":{"message":"Invalid API Key","type":"invalid_request_error","code":"invalid_api_key"}}
```

A Groq documenta 401 para credencial inválida, 403 para permissão e 429 para limite ([Errors](https://console.groq.com/docs/errors)) **[doc]**.

**Chamadas diretas.**

- Apps nativos não estão sujeitos a CORS.
- Para WebView/navegador, o preflight sem chave retornou `access-control-allow-origin: *`, permitiu o cabeçalho `authorization` e os métodos `GET`, `POST` e outros **[obs]**. Tecnicamente, a API aceita a chamada de qualquer origem.
- Em contrapartida, o guia de segurança da Groq diz "Never embed keys in frontend code or expose them in browser bundles" e recomenda proxy para uso no cliente ([llms-full.txt](https://console.groq.com/llms-full.txt), seção Security Onboarding) **[doc]**. O texto trata de chaves embutidas pelo desenvolvedor. O guia não aborda o caso de o próprio usuário informar a chave.

**Retenção de dados (fora da pergunta, mas relevante).** Os endpoints de transcrição e chat retêm dados por até 30 dias para confiabilidade e monitoramento de abuso, e qualquer cliente pode ativar Zero Data Retention nas configurações ([llms-full.txt](https://console.groq.com/llms-full.txt), seção Your Data in GroqCloud) **[doc]**.

## 7. O que ficou sem verificar

1. **Números oficiais do plano gratuito.** A Groq não os publica com esse rótulo. A melhor evidência é um print de terceiro.
2. **Se a duração mínima de 10 s conta para ASH/ASD.** A documentação fala só em cobrança.
3. **O que acontece acima de 224 tokens no `prompt`.** A Groq pode cortar ou rejeitar. Uma busca sobre um suposto limite de 896 caracteres não achou fonte confiável.
4. **Detecção automática de idioma na Groq**, inclusive em fala com mistura de idiomas.
5. **Qualidade real em português**, tanto do Whisper quanto da Limpeza (preservar sentido, tom e escolha de palavras). Só um teste com áudio e texto reais responde.
6. **Latência real** de ponta a ponta e variação entre chamadas.
7. **Quantos tokens de raciocínio** um GPT-OSS em `low` gasta numa Limpeza. Os 100 tokens da seção 3 são suposição.
8. **Se `/models` é filtrado por conta**, ou seja, se um modelo só Enterprise aparece para conta gratuita e depois dá 404. O que `active: false` significa também não é documentado.
9. **Se chamar `/models` consome cota**, e se a resposta traz cabeçalhos `x-ratelimit-*`.
10. **Quais codecs são aceitos** dentro de `ogg`, `webm` e `m4a`. A lista só cita contêineres e extensões.
11. **Se uma chave válida tem Whisper e modelos de texto liberados.** O 200 em `/models` só prova que a chave é aceita.

Os itens 2, 3, 4, 8 e 9 precisam de um teste com uma chave real.

## Fontes

Todas lidas em 2026-11-08.

- Groq: [Speech to Text](https://console.groq.com/docs/speech-to-text), [Rate Limits](https://console.groq.com/docs/rate-limits), [Models](https://console.groq.com/docs/models), [Deprecations](https://console.groq.com/docs/deprecations), [Structured Outputs](https://console.groq.com/docs/structured-outputs), [Reasoning](https://console.groq.com/docs/reasoning), [Prompt Caching](https://console.groq.com/docs/prompt-caching), [Errors](https://console.groq.com/docs/errors), [API Reference](https://console.groq.com/docs/api-reference), [Billing Plans](https://console.groq.com/settings/billing/plans), [Whisper Large v3](https://console.groq.com/docs/model/whisper-large-v3), [llms-full.txt](https://console.groq.com/llms-full.txt).
- OpenAI: [Whisper prompting guide](https://developers.openai.com/cookbook/examples/whisper_prompting_guide), [whisper/transcribe.py](https://raw.githubusercontent.com/openai/whisper/main/whisper/transcribe.py).
- Terceiro: [TypeLate #2](https://github.com/bobo52310/TypeLate/issues/2) (aberta em 2026-09-21).
