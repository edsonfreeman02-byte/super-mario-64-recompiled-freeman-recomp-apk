# SM64 Recompiled Android — v12 (BUILD-v12-notes.md)

**Ficheiro:** `~/workspace/sm64-recomp/sm64-recompiled-freeman-v12.apk`
**Tamanho:** 43 913 773 bytes (~41,9 MiB)
**SHA-256:** `b80db7121977fca817cbc77bf1d442d9b40aa94f420a2164528f5b9427c36f2e`
**versionCode:** 12 · **versionName:** 0.1.8-android-freeman · **package:** `com.freemanrecomp.sm64recomp`
**minSdk 24 / targetSdk 36** · mesmo certificado das v1–v11 (`bed31c3f…f05636`) → instala por cima da v11 sem perder saves nem a ROM importada.
**`.so` novo:** `fd285fd5…` (19 992 168 bytes, stripped; v11 era `e5061d3f…`).
**Primeiro arranque num telemóvel real NÃO TESTADO** — não há dispositivo ligado a esta VM.

## O que a v12 faz

Mantém TODOS os itens da v11 (Opções numa só tela sem scroll, sem D-pad nem botões C, ABXY formato Switch, zero mensagens de erro, gate de arranque de 500 ms, ícone CoopDX v1) e corrige as duas regressões da v11 + adiciona o stick de câmara pedido. Nada mais: sem funcionalidades novas, sem botões novos, sem opções novas.

## 1. O crash — simbolização (o registo era da v10, confirmado pelo utilizador)

O registo de crash em `crash-2026-11-13.txt` **é da v10** (confirmado pelo utilizador: "O registro é da V10"). Prova independente: a base de carregamento derivada dos frames #13 (`send_dl+0x64`) e #14 (`gfx_thread_func+0x368`) dá a mesma base `0x6e86abf000` com os símbolos da v10.so (`297bd735…`) e bases diferentes com a v11.so. A v11 NÃO corrigia isto (a v11 só mexeu em `main.cpp` e `ui_state.cpp` — nada no caminho render/send_dl).

Cadeia simbolizada (llvm-addr2line, `-f -C -i`, no `.so` unstripped da v11 com o desvio v10→v11 de +0x12c0):

```
#14 gfx_thread_func                    librecomp/src/recomp.cpp / ultramodern/src/events.cpp:679
#13 RT64Context::send_dl               lib/RecompFrontend/recompui/src/renderer/rt64_render_context.cpp:728
#12 RT64::Application::processDisplayLists   lib/rt64/src/hle/rt64_application.cpp:573
#11 RT64::Interpreter::processDisplayLists   lib/rt64/src/hle/rt64_interpreter.cpp:202
#10 RT64::GBI_RDP::fullSync            lib/rt64/src/gbi/rt64_gbi_rdp.cpp:275
#09 State::fullSync()::$_3             lib/rt64/src/hle/rt64_state.cpp:1511
#08 State::fullSync()::$_5             lib/rt64/src/hle/rt64_state.cpp:1744
#07 RT64::Framebuffer::copyRenderTargetToNative  lib/rt64/src/hle/rt64_framebuffer.cpp:135
#06 unique_ptr<plume::RenderBuffer>::get() (inline) in NativeTarget::copyToNative
     lib/rt64/src/render/rt64_native_target.cpp:422
#05–#03  noutra biblioteca (driver/loader Adreno, 0x6e8d…) — NÃO é libmain.so
#00  o handler de crash (android_diag::crash_handler, main.cpp:173) — é o frame
     do handler, não o PC da falha
```

**Local exato da falha — `copyToNative`, `rt64_native_target.cpp:422`:**
`RenderBufferBarrier(readBuffer->nativeBuffer.get(), RenderBufferAccess::READ)` — a barreira do estágio COPY. Tudo antes disto correu bem (a barreira COMPUTE, `setPipeline`, `setComputePipelineLayout`, `setComputePushConstants`, `setComputeDescriptorSet`, `dispatch`) — por isso o command buffer, o pipeline e os descriptor sets são válidos. O ÚNICO handle usado pela primeira vez aqui é o VkBuffer de **`writeBuffer.nativeReadbackBuffer`** (`readBuffer->nativeBuffer` já tinha sido provado válido na barreira COMPUTE). Conclusão: o VkBuffer do buffer de readback é `VK_NULL_HANDLE`.

**Porque é nulo (provado no código):** o construtor de `VulkanBuffer` (`lib/rt64/src/contrib/plume/plume_vulkan.cpp:845`) — se `vmaCreateBuffer` falhar (`res != VK_SUCCESS`) imprime "vmaCreateBuffer failed" e RETORNA, deixando `vk = VK_NULL_HANDLE` (valor por omissão do membro). O objeto continua a existir (unique_ptr não nulo) com um handle nulo, e o RT64 nunca verifica. `VulkanDevice::createBuffer` devolve `std::make_unique<VulkanBuffer>(...)` sem qualquer verificação. `copyToNative` cria o readback buffer sem verificar a validade e passa-o ao `vkCmdPipelineBarrier` (via `barriers()`, `plume_vulkan.cpp:3045`) / `vkCmdCopyBuffer` (via `copyBufferRegion`, linha 3375), que o driver Adreno desreferencia → SIGSEGV, fault addr 0xf0 (nulo + 0xf0). Porque é que o `vmaCreateBuffer` falhou para um buffer de ~150 KB (`getNativeSize(320, 240, 16b)` = 153 600 bytes) nessa corrida não está provado pelo trace (não foi enviado logcat) — a correção não depende de saber porquê.

### A correção — patch `rt64-0010` (novo; `runtime-patches/rt64-0010-android-null-vkbuffer-guards.patch`, entrada no `runtime-patches/README.md`)

Guardas de handle nulo no caminho exato da falha (nada muda quando os buffers estão saudáveis):

1. `plume_vulkan.cpp` — `VulkanCommandList::barriers`: salta barreiras de buffer onde o buffer é nulo ou `vk == VK_NULL_HANDLE`; o mesmo para barreiras de textura (`vk == VK_NULL_HANDLE`).
2. `plume_vulkan.cpp` — `VulkanCommandList::copyBufferRegion`: retorna sem fazer nada se um dos buffers tiver `vk` nulo (nunca entrega um VkBuffer nulo ao `vkCmdCopyBuffer`).
3. `plume_vulkan.cpp` — `VulkanBuffer::map`: devolve `nullptr` logo se `vk`/`allocation` for nulo (antes de `vmaMapMemory`).
4. `rt64_native_target.cpp` — `NativeTarget::copyToRAM`: depois do `map()`, se `readbackData == nullptr` salta o `memcpy` E o `unmap()` (só se faz unmap depois de um map com sucesso), mas incrementa na mesma o `writeBufferHistoryIndex` (mantém o histórico de write-buffers sincronizado) e retorna. Deixar a RDRAM com os valores antigos nessas linhas é muito melhor do que fechar.

**Provas no binário:**
- O patch aplica limpo ao submódulo rt64 no seu commit registado (`git apply --check` num `git archive HEAD` limpo — 5 hunks, 4 no plume + 1 no native_target).
- O `.so` unstripped da v12 desmontado (`llvm-objdump --line-numbers`, `copyToRAM` em 0xc1ec38): a linha 444 (`if (readbackData == nullptr)`) compila para `cbz x0` ANTES do `bl memcpy` da linha 448 — a guarda está no binário.
- Rebuild incremental: só os 2 ficheiros alterados recompilaram (`plume_vulkan.cpp.o`, `rt64_native_target.cpp.o`) + relink.

**Nota de âmbito:** o `map()` de `copyFromRAM` (contador de alterações) tem o mesmo padrão de risco, mas NÃO é o local provado pelo trace — não foi mexido. Se um dia houver um crash aí, é o próximo sítio a olhar.

## 2. O painel sem botões — causa e correção

**Causa (prova por comparação de dex):** o `onCreate` da v11 é igual ao da v10 exceto no fim — a v10 cria um `ScrollView`, mete o `root` lá dentro e faz `setContentView(scrollView)`; a v11 faz `setContentView(root)` diretamente. A captura de ecrã prova que o `top` (altura 0, peso 1f) mediu 0 de altura na v11: só a linha `bottom` (os pills) aparece, e aparece no TOPO do ecrã; todos os filhos do `top` (subtítulo, cartão da ROM, os 5 botões) desaparecem; a arte de capa continua a encher o ecrã porque é o fundo do `root`. Na v10 a mesma estrutura dentro do ScrollView mostrava os 5 botões (captura do utilizador às 00:29). O mecanismo exato de medição do AOSP não foi reproduzível nesta VM (sem dispositivo/emulador), por isso a correção é baseada na evidência, não em primeiros princípios.

**Correção (`LauncherActivity.java`, `onCreate`):** restaurada a estrutura da v10 EXATAMENTE (ScrollView à volta do `root`) + `setFillViewport(true)` (a forma documentada de fazer o filho do ScrollView encher a altura do viewport, tornando a distribuição dos pesos determinística). O conteúdo cabe numa só tela como na v10, por isso nada faz scroll — o requisito "uma só tela" continua cumprido (a queixa de scroll dele era sobre o diálogo de Opções, que continua como o diálogo de uma só tela da v11 — NÃO foi tocado). Adicionada uma linha de log só para o logcat com as alturas medidas do painel (`"Panel: root=…px top=…px screen=…px"`), no mesmo padrão do log do diálogo de Opções da v11. Fallback se isto falhar no telefone: reescrever o painel com RelativeLayout (sem pesos).

**Prova no dex da v12:** `ScrollView` ×1, `setFillViewport` ×1, `"Panel: root="` ×1 — todos no `LauncherActivity.onCreate` (dexdump: `new-instance ScrollView` + `setFillViewport(true)` + `addView(root)` dentro do onCreate). O diálogo de Opções continua sem ScrollView (só há uma criação de ScrollView no dex inteiro).

## 3. Stick de câmara (novo, pedido pelo utilizador)

**Posição** (`TouchControlsView.java`, frações da vista real):
- Centro: **fx=0.89, fy=0.44** → num ecrã 2400×1080: **x=2136, y=475**
- Raio: **rf=0.040f** (≈0.6× o raio do stick principal, 0.067f) → **~43 px**; knob rf=0.019f → ~20 px
- Folgas: borda inferior 518 px; topo do botão X 578 px → **folga ~60 px** (hit-test: ~26 px); muito abaixo de START/SELECT (fy 0.06) e do R (fy 0.06). Desenhado como o stick principal (base + anel + knob, estilo Storm, mais pequeno) com um pequeno rótulo "C" acima da base.

**Mapeamento (sem alteração nativa — `touch_input.cpp` intocado):**
- O stick principal e o de câmara têm ponteiros próprios (multi-toque: os dois sticks funcionam ao mesmo tempo).
- Deflexão → bits dos botões C do N64 (os mesmos bits que os botões BTN_C_* removidos usavam): direita → C-right, esquerda → C-left, cima → C-up, baixo → C-down; **diagonais combinam dois** (ex.: cima-direita = C_UP + C_RIGHT).
- Deadzone **0.3** (os botões C são digitais — é preciso uma deflexão deliberada); ao recentrar, os bits são libertados (a câmara para).
- Os bits vão no `mask` que o `pushState()` já envia por `nativeSetTouchInput(mask, stickX, stickY)`; o lado nativo já traduz `kBtnCUp=1<<9`, `kBtnCDown=1<<10`, `kBtnCLeft=1<<11`, `kBtnCRight=1<<12` para os bits C do N64 (C_UP 0x0008, C_DOWN 0x0004, C_LEFT 0x0002, C_RIGHT 0x0001).

**Prova no dex:** `camStick` ×1, `camMask` ×1, `updateCamStick` ×1; o bytecode do `updateCamStick` faz OR dos 4 bits — 4096 (1<<12, C_RIGHT), 2048 (1<<11, C_LEFT), 512 (1<<9, C_UP), 1024 (1<<10, C_DOWN) — para o campo `camMask`, e o caminho de release limpa o `camMask`.

## Itens da v11 mantidos (verificados)

- Opções numa só tela, centradas, sem scroll (`showOptionsDialog` intocado; 19 opções, mesmas chaves/valores).
- Sem D-pad nem botões C (só os bits C voltaram, como entrada do stick de câmara — nenhum botão C desenhado).
- ABXY formato Switch (A direita, B em baixo, X em cima, Y à esquerda).
- Zero mensagens de erro: `"Sem vídeo"` ×0, `"O jogo fechou"` ×0, `"registo"`/`"Registo"` ×0 no dex; `message_box` nativo continua só-log (RecompFrontend-0011).
- Gate de arranque 500 ms (10×50 ms) + tempos `t=+Nms` em cada linha de boot — strings `"gate_attempts"` e `"t=+"` presentes no `.so`.
- Ícone CoopDX v1: xxxhdpi square `0e8d183a…` e round `7e401387…` — idênticos aos da v11 em todas as variantes.
- Painel com os 5 botões (JOGAR / IMPORTAR ROM / OPÇÕES / GUARDAR / SAIR DO JOGO), fundo de capa, pill "Jogo pronto" (×1 no dex).

## Verificação (delivery gate)

| Verificação | Resultado |
|---|---|
| Ficheiro + tamanho | 43 913 773 bytes |
| SHA-256 | `b80db712…c36f2e` (diferente da v11 `c5017cd7…`) |
| Badging | package `com.freemanrecomp.sm64recomp`, versionCode **12**, versionName 0.1.8-android-freeman, minSdkVersion **24**, targetSdkVersion **36**, label "Super Mario 64: Recompiled – Freeman Recomp", launchable-activity `LauncherActivity` |
| apksigner verify | Verifies; v2 **true**, v3 **true**; **1 signer**; cert `bed31c3f…f05636` (igual às v1–v11) |
| Conteúdo | 58 ficheiros; **0 `.z64`**; **0 secções `.debug`** no `.so` |
| Dex | `"GUARDAR"` ×1, `"SAIR DO JOGO"` ×1, `"Jogo pronto"` ×1, `"Sem vídeo"` ×0, `"O jogo fechou"` ×0, `"registo"`/`"Registo"` ×0, `"Recompilação estática"` ×0 |
| Painel | ScrollView + setFillViewport no onCreate (dexdump) |
| Stick de câmara | bits C 512/1024/2048/4096 no `updateCamStick` (dexdump) |
| Crash fix | guarda `cbz` antes do `memcpy` no `.so` unstripped; patch aplica limpo ao HEAD do submódulo |

## Lista de testes (para o utilizador, quando a v12 for entregue)

1. Painel mostra os 5 botões (JOGAR / IMPORTAR ROM / OPÇÕES / GUARDAR / SAIR DO JOGO) numa só tela.
2. Ícone (se aparecer o antigo, reiniciar o telemóvel — é a cache do launcher).
3. Arranque mais rápido (gate de 500 ms).
4. Opções numa só tela, sem scroll.
5. Controlos: sem D-pad nem botões C; A à direita / B em baixo; **stick de câmara acima do losango ABXY** — empurrar para a direita/esquerda/cima/baixo move a câmara; ao soltar, a câmara para.
6. Vídeo nos DOIS arranques seguidos (o caso da v7).
7. Nenhum popup de erro em lado nenhum.
8. Sem crash ao arrancar (o caso da v10).

## Fallbacks intactos

- v11: `sm64-recompiled-freeman-v11.apk`, SHA-256 `c5017cd7a616373bd27429b0d902d00d888e5171c68ae6d394f3fa1fc52fddba`
- v10: `sm64-recompiled-freeman-v10.apk`, SHA-256 `b2bef7e3285f0f48aaebd6c183d8fbce2cdc3a8e04eb78bc389e8c01b5f678d1`
- v9: `sm64-recompiled-freeman-v9.apk`, SHA-256 `d525346d40dd1c3d4aa8995ff62a3a8b6009c76613824801917d2490dd633669`
- v7 (última confirmada funcional pelo utilizador): `sm64-recompiled-freeman-v7.apk`, SHA-256 `d98e9ac4275344a70acb816093712503dd0058e8df7789e3ad49c09d11bdd525`

## Ficheiros alterados nesta build

- `android/app/src/main/java/com/freemanrecomp/sm64recomp/LauncherActivity.java` — ScrollView + fillViewport no painel + log de alturas.
- `android/app/src/main/java/com/freemanrecomp/sm64recomp/TouchControlsView.java` — stick de câmara (bits C, desenho, toque, `pushState`).
- `lib/rt64/src/contrib/plume/plume_vulkan.cpp` — guardas de handle nulo (barriers, copyBufferRegion, map).
- `lib/rt64/src/render/rt64_native_target.cpp` — guarda de map nulo no copyToRAM.
- `runtime-patches/rt64-0010-android-null-vkbuffer-guards.patch` (novo) + entrada no `runtime-patches/README.md`.
- Script de embalagem: `~/work-tmp/apk-build-v12/build-v12.sh` (cópia do da v11, versionCode 12).
