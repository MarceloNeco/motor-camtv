# Motor CamTV

O motor de **câmera e TV** da SolverONE: o celular enxerga o corpo e as mãos pela câmera e conversa com a TV, que vira
uma segunda tela. Não pertence a nenhum app. Cada app (RiseONE, Bolinho de Chuva, Dança na Sala, Purpurina) é um
**cliente** do motor, como já acontece com o `motor-pixel` (desenho) e o `leitor-ocr` (leitura de documentos).

## Em que pé está (19/Nov/2026)

**Só pesquisa e proposta. Ainda não existe código do motor.**

| O que existe | Onde |
|---|---|
| A pesquisa completa (viabilidade, motores de pose, limites da TV LG, pareamento, lógica de jogo e exercício, plano) | [`docs/RELATORIO-motor-de-pose-TV-LG-2026-11-19.md`](docs/RELATORIO-motor-de-pose-TV-LG-2026-11-19.md) |
| A proposta de arquitetura, de pastas e de interface | [`ARQUITETURA.md`](ARQUITETURA.md) |
| O que falta decidir e fazer | [`PENDENCIAS.md`](PENDENCIAS.md) |
| De quem são as ideias e bibliotecas | [`CREDITOS.md`](CREDITOS.md) |

## A ideia em uma figura

```
 CELULAR                                        TV
 câmera → lê o corpo → tira o tremor → mede     mostra o QR para parear
        → manda só NÚMEROS (os pontos)  ─────▶  recebe os pontos e desenha o jogo
                                        ◀─────  devolve o estado do jogo
 A imagem nunca sai do celular.
```

## O que o motor vai resolver

- **Exercício** (RiseONE): contar repetições e dar dicas simples de postura.
- **Dança** (Dança na Sala): comparar o movimento da pessoa com uma coreografia.
- **Controle tipo Wii**: apontar e escolher na TV com a mão.
- **Celular como controle** de jogos na TV, com ou sem câmera.
- **Ensinar um gesto** com poucos exemplos, treinando no próprio celular.

## Próximo passo

Uma página de diagnóstico de 1 dia, aberta na TV e no celular, que responde as 12 perguntas da seção 15 do relatório.
Só depois dela vale escrever o motor.

## Regras

Sem build, JavaScript puro, sem CDN, nenhuma chave no repositório, nenhum nome de app dentro do motor. A imagem da
câmera nunca é gravada nem enviada. Detalhes em `CLAUDE.md`.
