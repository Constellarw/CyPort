# Handoff: CyPort no iPhone 16e

## Objetivo atual
Investigar e melhorar a fluidez de Cyberpunk 2077 no iPhone 16e, especialmente em áreas abertas, onde há queda de FPS e o app pode fechar. A próxima LLM deve continuar a partir das evidências abaixo, sem presumir que reduzir resolução sozinho resolve o problema.

## Dispositivo e contexto
- Aparelho informado: iPhone 16e.
- Arquitetura relevante: A18 com GPU de 4 núcleos, tela de 60 Hz e cerca de 8 GB de RAM física.
- CyPort já executa o jogo com Game Mode, gerenciamento de QoS dos workers, streaming limitado, frame pacing e diagnóstico térmico.
- O jogador já tem o jogo baixado no aparelho; a IPA de teste reutiliza o app base e não inclui os dados do jogo.

## Alterações já feitas
Branch pessoal no fork público `Constellarw/CyPort`:
`personal/physical-controller-input-latency`

Commits recentes:
- `b1ab126` adiciona workflow de teste pessoal, inicialmente no fork CyPort.
- `2acab98` corrige a declaração de `g_realPad`, encontrada pela primeira compilação do workflow.
- `360d053` escolhe resolução fallback de 1,25x para telas de 60 Hz e mantém 1,5x em ProMotion; atualiza o rótulo `recommended`.
- `0309454` tentativa de Ultra Performance (DRS 35-50%) e HUD 125%/legendas 62pt (revertido em `df46be6` a pedido do usuário após testes reais no aparelho mostrarem perda excessiva de qualidade visual e invasão de tela).

A correção de atraso do controle físico drena a fila principal por frame quando `g_realPad` está conectado. A declaração de `g_realPad` foi movida para o bloco de globais para compilar no runner iOS.

## Workflow privado e IPA
A IPA base está no Release privado `Constellarw/cyipa`, tag `ipa-base-v1`, asset `CyPort-unsigned.ipa`. O repositório privado foi inicializado com um README e contém o workflow manual:
`.github/workflows/build-personal-test-ipa.yml`

O workflow no `cyipa`:
1. baixa a IPA do Release privado usando o `GITHUB_TOKEN` automático do próprio repo;
2. faz checkout da branch de código indicada;
3. reaplica `tools/ios_retarget.py` ao executável da IPA base;
4. compila `libMacShim.dylib` no runner macOS;
5. substitui a shim, reassina ad hoc e publica `CyPort-personal-test-ipa` como artifact.

A Action está na `main` do `cyipa`. Inputs padrão: branch `personal/physical-controller-input-latency`, tag `ipa-base-v1`, asset `CyPort-unsigned.ipa`. Não é necessário PAT cross-repo.

Última execução bem-sucedida, com retarget de pools, Ultra Performance e melhorias de HUD aplicadas:
https://github.com/Constellarw/cyipa/actions/runs/39866437846

Artifact: `CyPort-personal-test-ipa` (ID `12463299207`), aproximadamente 62.6 MB.
Build anterior (apenas retarget de pools):
https://github.com/Constellarw/cyipa/actions/runs/39852796186

## Evidência dos logs enviados pelo usuário
Os arquivos locais `app/macshim.log` e `app/game_output.log` foram adicionados ao workspace e estão untracked. **Não os adicione/commite**: contêm caminhos privados e identificadores de sessão/conta.

Na última sessão registrada antes do retarget corrigido:
- O jogo estava em saída `1055x488` (1,25x); MetalFX recebia `844x390` e o render efetivo ficou em torno de 703 px de largura (67%).
- O aparelho reportou 7719 MB de RAM física.
- A partir do início já aparecia `thermal HOT (throttling)`.
- Em áreas abertas, GPU busy variou aproximadamente 83–100%; foram registrados 32–56 ms por frame e FPS caiu a cerca de 14–20, apesar do cap de 30.
- Footprint chegou a aproximadamente 5.8–5.9 GB. O jogo leu ~17.9 GB cumulativos de dados durante a sessão.
- No encerramento, a reserva de um mapping anônimo de 4 GB falhou com `errno=12`; o maior bloco contíguo disponível era 1 GB. Em seguida ocorreu `SIGTRAP` no `SerializeAllocatorMetrics<FrameAllocatorWithFallback>`; a pilha também passou por `red::memory::Free<PoolGameServices>`.
- Não há indicação de que tenha sido apenas um jetsam silencioso: há um `***FAULT`/`***CRASH signal 5` capturado pelo shim.
- `game_output.log` não mostra assert claro correlato; há erros esperados de serviços GOG/rich presence e uma mensagem sobre run-loop mode.

## Diagnóstico atual
Há dois problemas diferentes:
1. **Performance térmica/GPU:** o 16e fica HOT e a GPU satura em cenas abertas. A resolução mais baixa ajuda, mas os dados mostram que ela não basta para manter 30 FPS nesse trecho.
2. **Crash de endereço virtual/allocator:** a IPA base não tinha o patch `pool-end` que faz o executável usar o fim real das reservas reduzidas pelo shim. O workflow original trocava apenas a biblioteca. O workflow atualizado reaplica o retargeter ao executável, e o build com esse patch passou.

A hipótese principal é que o crash da sessão enviada está ligado à IPA antiga sem patch do executável. O novo artifact ainda precisa ser testado no aparelho; não afirmar que resolveu até analisar os próximos logs.

## Próximo teste no aparelho
1. Baixar o artifact da execução `39852796186` e extrair `CyPort-personal-test.ipa`.
2. Atualizar o app existente por cima, com o mesmo bundle ID/perfil; não apagar o app, para preservar o jogo e os saves.
3. No launcher, selecionar `844x390` (1x), pois a resolução escolhida anteriormente fica salva e não muda automaticamente após atualizar.
4. Testar a mesma rota em área aberta com cap de 30 FPS, frame generation desligado e Low-latency input desligado. Comparar com o teste anterior após 5–10 minutos.
5. Se fechar, coletar novos `macshim.log` e `game_output.log` de **Arquivos > No Meu iPhone > CyPort** e conferir se aparecem `address space:`, `pool reservation`, `mmap ... FAILED`, `***FAULT` ou `***CRASH`.

## Arquivos de código relevantes
- `shim/MacShim.m`: streaming em `MSWriteStreamingIni`, pools e `ms_mmap`, estado térmico/HotBoost, pacing e diagnósticos.
- `tools/ios_retarget.py`: patch de pool-end e redução dos tamanhos dos pools no executável.
- `README.md`: resolução/frame pacing e performance tips.
- `.github/workflows/build-personal-test-ipa.yml` no repositório privado `Constellarw/cyipa`: build manual da IPA.

## Cuidados
- Não trocar a IPA base nem alterar bundle ID sem motivo: o app existente guarda o jogo e os saves.
- O artifact é ad hoc; reinstalar via Sideloadly/AltStore com o mesmo App ID/perfil. GetMoreRam depende do perfil/entitlement da conta e não é ativado pelo build sozinho.
- Não commitar `app/*.ipa`, `app/macshim.log` ou `app/game_output.log`.
- Nenhuma métrica de FPS pós-patch foi coletada ainda. Priorizar teste comparativo real no 16e antes de novos ajustes de streaming, pools ou QoS.
