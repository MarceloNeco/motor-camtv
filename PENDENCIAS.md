# Motor CamTV — pendências (lista oficial e viva)

Item novo entra no fim, com o próximo número (números não mudam nem são reaproveitados). Criada em 19/Nov/2026.

**Situações:** 🟡 aberta · 🔵 em andamento · ⏸ aguardando decisão do dono · ✅ feita (com a versão)

As seções citadas são do relatório `docs/RELATORIO-motor-de-pose-TV-LG-2026-11-19.md`.

| # | Pendência | Origem | Situação |
|---|---|---|---|
| 1 | **Decisão do dono:** aprovar (ou ajustar) a estrutura de pastas e a interface propostas no `ARQUITETURA.md` (seções 4 e 5), e os ids de interruptor sugeridos (`camera-corpo`, `camera-maos`, `modo-tv`, `ensinar-gesto`) | Pesquisa de 19/Nov/2026 | ⏸ aguardando o dono |
| 2 | **Fase 0, diagnóstico (1 dia):** página que roda na TV LG e no Galaxy S24 e responde as 12 perguntas da seção 15: navegador exato da TV e o que ele suporta; WebRTC conecta?; `wss://` na casca; cache na casca; tempo de carga casca x empacotado; quadros por segundo do canvas na TV; Three.js do `motor-pixel` no Chromium 79; modo jogo vale para apps?; leituras por segundo no S24 (leve e completo); queda por calor em 20 minutos; câmera grande-angular acessível? | Pesquisa, seção 15 | 🟡 aberta (depende do #1 e do dono com a TV) |
| 3 | **Fase 1, protótipo mínimo (1 a 2 dias):** `exemplos/esqueleto-no-celular.html` e `exemplos/esqueleto-na-tv.html`. O celular lê a pose e manda pacotes compactos; a TV mostra QR, desenha o esqueleto com interpolação e mostra o cano em uso e o atraso. Sucesso: atraso abaixo de 200 ms por 5 minutos, nos dois canos | Pesquisa, seção 14 | 🟡 aberta (depende do #2) |
| 4 | **Fase 2, núcleo 1.0:** `nucleo`, `camera`, `corpo`, `suave`, `medidas`, `gravador`, `motor.js`, `manifesto.json`, exemplos e testes com gravações. Copiar o MediaPipe (versão atual) para `1.0/libs/` | Pesquisa, seção 13 | 🟡 aberta |
| 5 | **Gravações de teste:** gravar clipes de pontos (sem imagem) de agachamento, polichinelo, prancha, flexão e afundo, anotados com o número de repetições, para regular filtros e limites | Pesquisa, seção 7.7 | 🟡 aberta |
| 6 | **Fase 3, RiseONE:** trocar o bloco "43. Espelho com câmera" pelo motor; contagem de repetições com calibração por pessoa; pedir a vista certa (de lado) por exercício; dicas relativas. Corrige: esqueleto piscando, ângulos distorcidos, MediaPipe vindo de CDN | Pesquisa, seções 3.2 e 7 | 🟡 aberta (conversa no repositório `rise-one`) |
| 7 | **Fase 4, `sala` + `tv` + `desenho`:** pareamento por QR e código com segredo longo, dois canos com troca automática, relógio combinado, navegação por setas, saída pelo Voltar. Resolve as pendências 59 e 60 do `kids` | Pesquisa, seções 6 e 8.3 | 🟡 aberta |
| 8 | **Cota do cano reserva:** o Supabase grátis dá 2 milhões de mensagens/mês (~18 h a 30 por segundo) e 100 por segundo no projeto inteiro. Mandar 15 por segundo no reserva; se o WebRTC não funcionar na LG, pesquisar um retransmissor próprio barato | Pesquisa, seção 6.2 | 🟡 aberta (depende do #2) |
| 9 | **Fase 5, Purpurina:** desfile como página da TV, camarim no celular. Enquanto isso, cabo USB-C → HDMI resolve sem código | Pesquisa, seção 9 | 🟡 aberta (conversa no repositório `kids`) |
| 10 | **Fase 6, Dança na Sala:** comparação de poses por direção dos membros, nota por passo com janela de ±250 ms e termo de movimento; uma coreografia própria (gravada por nós, não baixada do YouTube); parar de mandar vídeo para a TV | Pesquisa, seção 7.5 | 🟡 aberta (conversa no repositório `kids`) |
| 11 | **Fase 7, `ensinar` + `maos` + `ponteiro`:** modo ensinar gesto (k-NN sobre pontos, treinado no celular), os 8 gestos prontos, ponteiro tipo Wii com clique por permanência | Pesquisa, seções 7.6 e 11 | 🟡 aberta |
| 12 | **Fase 8, outras marcas e loja:** cascas Samsung (Tizen) e Android TV; conferir no LG Seller Lounge se o Brasil aceita pessoa física, documentos, regras para app infantil, prazo, e se app com WebRTC passa | Pesquisa, seções 8.4 e 8.6 | ⏸ aguardando o dono (decidir se quer loja) |
| 13 | **Mudar as cascas `tv-lg*` e o `instalar-lg.py`** do repositório `kids` para `cascas/lg/` deste motor, e rodar o instalador da SolverONE numa TV real (até hoje só o original do free-my-tv foi testado) | Planilha, itens 62 e 63 | 🟡 aberta |
| 14 | **Privacidade:** texto do aviso antes da câmera (PT, EN, ES), indicador de câmera ligada, e validação com advogado antes de qualquer coleta, em especial nos apps infantis | Pesquisa, seção 10 | ⏸ aguardando o dono |
| 15 | **SignDETR:** se um dia formos copiar código de lá, confirmar a licença com o autor (o README diz MIT, mas não há arquivo de licença). Hoje só a ideia do "modo ensinar" é aproveitada | Pesquisa, seção 11 | 🟡 aberta (baixa prioridade) |
| 16 | **Publicação:** decidir quando este repositório passa a ser servido em `solverone.com.br/motor-camtv/` e cadastrar o motor no RootifyONE, no ArchONE e no EaglePlanONE | Sincronia de 19/Nov/2026 | ⏸ aguardando o dono |

## Histórico das entregas

- **19/Nov/2026:** pesquisa concluída e documentação inicial (README, ARQUITETURA, PENDENCIAS, CREDITOS, CLAUDE e o
  relatório em `docs/`), na branch `claude/great-ritchie-m2yprb`. Nenhum código do motor.
