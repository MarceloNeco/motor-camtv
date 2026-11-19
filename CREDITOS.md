# Créditos e licenças

Tudo o que não nasceu na SolverONE, com origem e licença. Conferido em 19/Nov/2026.

## Já usado como base

| O quê | Origem | Licença | Uso |
|---|---|---|---|
| Receita de instalar app na TV LG pelo Modo Desenvolvedor (empacotar `.ipk` e instalar pelo Wi-Fi) | free-my-tv, de simbasjungles: https://github.com/simbasjungles/free-my-tv , que cita o FreeMyTvWebOS de danielthejoker18 | MIT | Base do `instalar-lg.py` (hoje no repositório `kids`, pasta `tv-lg/`); testado na LG em 17/Nov/2026 |

## Candidatos, quando o motor for escrito (nada copiado ainda)

| O quê | Origem | Licença | Observação |
|---|---|---|---|
| MediaPipe Tasks Vision (Pose Landmarker, Hand Landmarker, Gesture Recognizer) e modelos `.task` | Google AI Edge: https://github.com/google-ai-edge/mediapipe | Apache-2.0 | Motor de visão padrão. Copiar a biblioteca e os modelos para `1.0/libs/`, com o arquivo de licença |
| Filtro One Euro | Casiez, Roussel e Vogel, CHI 2012: https://gery.casiez.net/1euro/ | Algoritmo publicado; escrever a nossa versão | Citar o artigo |
| Comparação de poses por cosseno | Move Mirror, Google, 2018: https://blog.tensorflow.org/2018/07/move-mirror-ai-experiment-with-pose-estimation-tensorflow-js.html | Ideia publicada; escrever a nossa versão | Citar o artigo |
| Classificação de poses por vizinhos (k-NN) e contagem | Guia do Google ML Kit: https://developers.google.com/ml-kit/vision/pose-detection/classifying-poses | Ideia publicada; escrever a nossa versão | |

## Inspiração, sem reaproveitar código

| O quê | Origem | Licença | Observação |
|---|---|---|---|
| Ciclo "coletar → rotular → treinar → testar → usar ao vivo" | SignDETR, de Nicholas Renotte: https://github.com/nicknochnack/SignDETR e https://www.youtube.com/watch?v=o_MGqeFMAGE | O README diz MIT, mas **não há arquivo de licença** na raiz | Reaproveitamos só a ideia do "modo ensinar". Antes de copiar qualquer código, confirmar a licença com o autor |

## Descartados por licença

| O quê | Motivo |
|---|---|
| Ultralytics YOLO (detecção e pose) | AGPL-3.0 ou licença paga, inclusive para modelos treinados por nós |
| YOLO-NAS | Pesos proíbem uso comercial |
| Sapiens (Meta) | Não comercial |
| ml5.js | Licença própria, condicionada a código de conduta; bom para aprender, não para produto |
