# RELATÓRIO — Motor de pose + TV LG (19/11/2026)

Pesquisa pedida pelo Marcelo (dono da SolverONE) e executada em 19/Nov/2026. Cobre a Parte B do brief de 17/Nov,
o adendo de 17/Nov (apps na TV, espelhar x transmitir, câmera sem TV), o material F de 18/Nov (SignDETR) e a decisão
de 18/Nov de que isto vira um **motor independente**, no repositório `motor-camtv`.

**Como ler:** cada afirmação vem marcada.
- **[Confirmado]** = li na fonte hoje (link na seção 16).
- **[Suposição]** = raciocínio meu ou conhecimento geral, não conferido em fonte hoje.
- **[Testar na TV]** = só dá para saber testando no aparelho de verdade.

Nada foi implementado. Este documento é pesquisa, proposta e plano.

---

## 1. Resumo em 10 linhas

1. **Dá para fazer, e a sua hipótese está certa:** o celular enxerga o corpo e manda só números para a TV. A TV desenha.
   Vídeo nunca precisa sair do celular.
2. **O motor de visão certo já é o que você usa:** MediaPipe Pose Landmarker (Google, licença livre, 33 pontos com 3D).
   O problema dos seus apps não é o motor de visão, é o que foi feito (ou não foi feito) depois dele.
3. **Achei defeitos concretos no seu código** (RiseONE e Bagunçolândia): esqueleto piscando, ângulos distorcidos,
   ponto "inventado" quando a junta some, nada de suavização, nada de contagem com memória. Seção 3.
4. **A TV LG de 2021 tem um navegador de 2019** (Chromium 79). O que roda nela precisa ser escrito "à moda antiga".
   Os motores de pose **não** devem rodar na TV, só no celular.
5. **Risco nº 1:** a LG diz que WebRTC (o "cano direto" entre celular e TV que seus jogos usam hoje) só é liberado para
   parceiros. Pode funcionar na sua TV, mas não é garantido. O motor precisa de dois canos: o direto e um reserva.
6. **Modo Desenvolvedor não serve para cliente.** Dura 1000 horas (uns 41 dias), os apps somem quando expira, e uma
   conta só fica logada em uma TV por vez. Para cliente: endereço/QR no navegador da TV ou a loja oficial da LG.
7. **Espelhar a tela sempre terá atraso.** Sua TV tem Miracast e AirPlay 2, mas **não tem Chromecast**. Para o
   Purpurina, o certo é o desfile rodar na própria TV e o celular mandar só os comandos.
8. **O vídeo do Nicholas Renotte (SignDETR) tem a ideia certa e a ferramenta errada para celular.** "Ensinar o app com
   meus exemplos" se faz muito mais leve: classificando os pontos da mão ou do corpo, treinando em segundos no próprio
   celular.
9. **Proposta:** o `motor-camtv` nasce com o mesmo molde do `motor-pixel` (módulos sob demanda, versão em pasta,
   bibliotecas guardadas no repositório), e substitui as 4 leituras de câmera e as 5 "salas" que hoje estão copiadas
   dentro dos jogos.
10. **Primeiro passo, antes de qualquer código do motor:** uma página de diagnóstico de 1 dia, aberta na TV e no S24,
    que responde as 12 perguntas da seção 15. Sem ela, qualquer arquitetura é aposta.

---

## 2. Veredito e arquitetura recomendada

**Veredito: viável.** Com três condições: (a) o trabalho pesado fica no celular; (b) a página da TV é leve e escrita
para um navegador antigo; (c) o transporte tem plano B.

```
        CELULAR (Chrome / Samsung Internet)                    TV LG webOS 6.5 (Chromium 79)
 ┌──────────────────────────────────────────┐          ┌──────────────────────────────────────┐
 │ câmera 640x480, 30 quadros/s             │          │ página "tela de TV" (leve, JS antigo) │
 │   ↓                                      │          │   mostra QR + código para parear      │
 │ MediaPipe Pose (lite), 15 a 20 leituras/s│          │   recebe pacotes, ordena pelo número  │
 │   ↓                                      │          │   amortece 50 a 100 ms e interpola    │
 │ filtro (tira o tremor) + confiança       │          │   desenha: esqueleto, avatar, jogo    │
 │   ↓                                      │          │   regras do jogo e placar             │
 │ medidas: ângulos, normalização           │   cano   │                                      │
 │   ↓                                      │ ───────▶ │                                      │
 │ pacote pequeno (~240 bytes, com relógio) │ ◀─────── │   estado do jogo, "vibre", "pronto"   │
 └──────────────────────────────────────────┘          └──────────────────────────────────────┘
                         │                                          │
                         └───────────── cano A: direto (WebRTC) ────┘   mais rápido, se a TV aceitar
                         └───── cano B: pela nuvem (Supabase Realtime) ─┘   reserva que sempre funciona
```

Decisões da arquitetura:

| Decisão | Escolha | Por quê |
|---|---|---|
| Onde roda a visão | Só no celular | A TV tem navegador de 2019 e processador fraco. [Confirmado] Chromium 79 |
| O que trafega | Só pontos (números), nunca vídeo | Privacidade e leveza. Mandar vídeo e rodar IA ao mesmo tempo esquenta e trava o celular [Suposição forte] |
| Cano principal | WebRTC, canal de dados "sem garantia de entrega" | Na mesma rede Wi-Fi o atraso de rede fica em poucos milissegundos [Suposição, com base em AirConsole] |
| Cano reserva | Supabase Realtime (Broadcast) | Já existe na plataforma. Mediana de 13 ms no servidor [Confirmado, benchmark do Supabase] |
| Troca de cano | Automática: tenta o direto por 3 a 5 s, se não abrir usa a nuvem | A LG não garante WebRTC [Confirmado] |
| Pareamento | QR na TV + código curto de reserva | É o padrão do Just Dance Now [Confirmado] |
| Página da TV | Hospedada (https), aberta por uma "casca" instalada | App empacotado puro bloqueia módulos e cache [Confirmado em emulador] |

---

## 3. O que já existe no seu código, e por que "aparecem os risquinhos mas não funciona"

Li o RiseONE e o repositório `kids` (para onde os jogos e a pasta `tv-lg` foram em 02/Nov). Isto é leitura de código,
não teste em aparelho.

### 3.1 O que encontrei

| Onde | O que existe |
|---|---|
| RiseONE, `index.html`, bloco "43. Espelho com câmera" | MediaPipe 0.10.14 **baixado da internet** (jsDelivr/unpkg) e modelo do Google Storage. Regras por exercício (agachamento, prancha, rosca...) em cima de ângulos 2D |
| kids, `baguncolandia/esqueleto.js` e `peluqueria/esqueleto.js` | MediaPipe guardado no repositório (`baguncolandia/libs/mediapipe/`, 28 MB). Devolve 13 pontos, até 2 pessoas, 10 a 15 leituras por segundo |
| kids, `vruuum/maos.js` e `rabiscolandia/mao.js` | Leitura das mãos (Hand Landmarker) |
| kids, `baguncolandia/sala.js`, `peluqueria/sala.js`, `purpurina/sala.js`, `rabiscolandia/sala.js`, `batalha-naval/rede.js` | Cinco versões da "sala": código de 4 letras, Supabase Realtime para combinar, WebRTC ponto a ponto para vídeo e dados |
| kids, `tv-lg`, `tv-lg-kids`, `tv-lg-baguncolandia`, `tv-lg-peluqueria` | Quatro "cascas" de app para a LG que só abrem o endereço do site |
| kids, `PENDENCIAS.md` #59 e #60 | Suas duas pendências: "celular vira só controle" e "jogos de câmera: refazer, hoje não funcionam ('nenhum funcionou, horrível')" |

Ou seja: a ideia de mandar só pontos para a TV **já está implementada** na Bagunçolândia. Ela falha por execução.

### 3.2 Defeitos concretos (lidos no código)

1. **Esqueleto piscando no RiseONE.** A tela é apagada a cada quadro (60 vezes por segundo), mas o esqueleto só é
   redesenhado quando há leitura nova (a cada 70 ms). Em 3 de cada 4 quadros a tela fica vazia. Isso sozinho explica
   "os risquinhos aparecem meio que reconhecendo".
2. **Ângulos distorcidos no RiseONE.** O ângulo do joelho é calculado com x e y "normalizados" (0 a 1), mas o vídeo é
   retangular (1280x720). Um mesmo movimento dá ângulos diferentes conforme a direção. É preciso multiplicar pela
   largura e altura reais antes, ou usar os pontos 3D em metros que o MediaPipe também entrega.
3. **Vista errada para o exercício.** [Confirmado, Google ML Kit] ângulo em 2D muda conforme a pessoa está de frente
   ou de lado. Agachamento e flexão vistos de frente não medem profundidade. Estudo publicado: erro de 15° parado e
   25° no agachamento com uma câmera só.
4. **Ponto inventado na Bagunçolândia.** Quando uma junta some, o código coloca ela no centro da tela
   (`{x: 0.5, y: 0.5}`) e a confiança (visibilidade) nunca é consultada. O boneco "pula" para o meio.
5. **Sem suavização.** Nenhum dos dois filtra o tremor natural das leituras. Qualquer regra com limite ("joelho
   abaixo de 100°") fica ligando e desligando.
6. **Sem memória.** As regras do RiseONE olham um quadro por vez. Não existe "desceu, subiu, conta 1". Por isso não há
   contagem de repetições nem nota do movimento.
7. **Duas pessoas, modelo leve, 10 a 15 leituras/s, sem interpolação.** Custa o dobro e o movimento na TV fica aos
   trancos.
8. **Enquadramento.** [Confirmado, cartão do modelo] o BlazePose precisa da **cabeça visível** e funciona até uns 4 m.
   Em sala pequena, com a câmera da frente, o corpo não cabe. É o seu pedido de "zoom out". Saída a testar: usar a
   câmera traseira **grande-angular** do S24 [Testar no celular].
9. **Biblioteca vinda da internet no RiseONE.** Contraria a diretriz (hospedar no repositório), quebra offline e expõe
   o IP da pessoa a terceiros.
10. **Nenhum clipe gravado para regular.** Sem gravações de teste, cada ajuste é "no olho". A Ubisoft usou 9.000
    movimentos gravados para calibrar o Just Dance com câmera [Confirmado].
11. **WebRTC na TV nunca foi testado de verdade.** A pendência "testar no navegador real da LG" continua aberta no
    `kids` (#4 de lá). E a LG diz que não garante (seção 5).

### 3.3 A lista geral de causas (vale para qualquer app de pose)

Tremor cru alimentando regras; comparar coordenadas de tela em vez de formas normalizadas; falta de máquina de estados
com folga (histerese); limites copiados em vez de calibrados por pessoa; ignorar a confiança de cada ponto;
espelhamento trocado (esquerda/direita); ângulo 2D na vista errada; visão rodando na mesma "pista" da tela e travando
tudo; mandar vídeo e rodar IA juntos; relógios do celular e da TV não combinados; câmera mal posicionada; e nenhuma
gravação para testar de novo.

---

## 4. Comparativo dos motores de pose

Nenhuma fonte publica quadros por segundo no **Galaxy S24** nem em celular médio no navegador. Procurei bastante.
Os números abaixo são de outros aparelhos e servem só de referência. **Medir no S24 é obrigatório.**

| Motor | Pontos | Pessoas | Tamanho | Licença | Velocidade publicada (referência) | Veredito |
|---|---|---|---|---|---|---|
| **MediaPipe Pose Landmarker** (lite / full / heavy) | 33, com 3D, confiança e versão em metros | 1 (até mais com `numPoses`) | 5,8 / 9,4 / 30,7 MB + 14,7 MB de base | Apache-2.0 (livre) | Pixel 5 no navegador: 32 / 22 quadros/s (2021). Pixel 3 nativo: 49 / 40 / 19 | **Padrão do motor** |
| MediaPipe "Pose" antigo | 33 | 1 | — | Apache-2.0 | — | Abandonado em 2023. Não usar |
| **MoveNet** Lightning / Thunder (TensorFlow.js) | 17, só 2D | 1 (MultiPose: até 6) | não confirmado | Apache-2.0 | Pixel 5: 34 / 12. iPhone 12: 51 / 43 | Biblioteca parada desde 2023/2024. Só como reserva futura |
| YOLO-pose (Ultralytics) | 17, 2D | várias | 2,9 M de parâmetros (n) | **AGPL ou licença paga** | só números de computador | **Descartado** pela licença |
| RTMPose / RTMO | 17 / 26 / 133 | várias | 3,3 a 13,6 M de parâmetros | Apache-2.0 (não conferi o arquivo) | Snapdragon 865 nativo: 9 a 26 ms | Bom, mas sem kit pronto para navegador. Pesquisa futura |
| ml5.js bodyPose | 17 ou 33 | várias | os mesmos modelos | licença própria, com código de conduta | — | Ótimo para aprender, não para produto |
| ML Kit / Apple Vision | 33 / 17 | 1 | — | fechados | 30 a 45 quadros/s nativo | Só se um dia houver app nativo |
| Sapiens (Meta) | — | — | — | não comercial | — | Descartado |

Fatos importantes sobre o MediaPipe [Confirmado]:
- Está vivo: versão 1.2.3 publicada em 11/Nov/2026. Seus apps usam a 0.10.x, de 2024.
- Os arquivos de modelo não mudam desde abril de 2023, então atualizar a biblioteca não muda a precisão.
- Dá para hospedar tudo no repositório (é o que o `kids` já faz).
- Fora do escopo declarado do modelo: mais de uma pessoa, além de 4 m, cabeça fora do quadro, profundidade exata.
- Em modo vídeo ele faz rastreamento entre quadros. Se aplica filtro de tremor por conta própria na web, não confirmei.
- Rodar numa "pista" separada (Web Worker) tem um defeito conhecido; há contorno, a testar.

Qual usar para quê:

| Uso | Padrão | Reserva |
|---|---|---|
| Exercício (forma e repetições) | Pose Landmarker **full**, usando os pontos em metros para os ângulos | lite |
| Dança | Pose Landmarker **lite** | lite a menos leituras por segundo |
| Controle tipo Wii (ponteiro) | Pose lite, pulso em relação ao ombro, bem filtrado | Mãos (Hand Landmarker) quando a pessoa está perto |
| Duas pessoas | Uma câmera por pessoa (um celular cada) | `numPoses: 2`, sabendo que piora |

Calor e bateria: não achei medição publicada de sessões de 10 a 30 minutos. Recomendação [Suposição]: câmera em
640x480, modelo lite, 15 a 20 leituras por segundo com interpolação na tela, um "vigia" que reduz o ritmo quando o
tempo de leitura sobe, pausa quando a pessoa sai da tela do app, e um teste de 20 minutos em cada celular alvo.

WebGPU (aceleração nova): no Android vale para chips Qualcomm e ARM desde o Chrome 121. No chip do S24 brasileiro
(Exynos, Xclipse) só há relato da comunidade de que foi ligado em agosto de 2026. **Não contar com isso.** Base: WebGL.

---

## 5. A TV: limites do webOS 6.5.3 e do Modo Desenvolvedor

### 5.1 O navegador da TV

- [Confirmado, LG] webOS TV 6.x = **Chromium 79** (de dezembro de 2019). A tabela da LG tem uma linha só para "6.x";
  não há entrada separada para 6.5. [Testar na TV] conferir pelo `navigator.userAgent`.
- Para comparação [Confirmado]: webOS 22 = 87, 23 = 94, 24 = 108, 25 = 120, 26 = 132. Samsung 2021 = 76 (mais antigo
  ainda).
- **O que falta no Chromium 79** [Confirmado para `?.`; Suposição para os demais, conferir]: `?.` e `??`,
  `replaceAll`, `:has()`, `gap` em flex, `aspect-ratio`, `:is()`/`:where()`, WebGPU.
- **Consequência para a plataforma:** o módulo comum da SolverONE exige Chrome 105 por causa do `:has()` da
  `janela.js`. A TV está abaixo disso. A "tela de TV" precisa ser uma página própria, leve, que **não** carrega o
  módulo comum inteiro. O Three.js r187 do `motor-pixel` também pode não rodar nela [Testar na TV].

### 5.2 Recursos, um a um

| Recurso | Situação | Fonte |
|---|---|---|
| WebSocket | Deve funcionar (padrão do Chromium 79) | [Suposição] sem declaração da LG |
| **WebRTC** | LG, 07/Jan/2026: "só disponível para apps de parceiros contratados". Em 2023: "especificação não aberta". Relatos: Google Meet funcionou no navegador da TV em webOS 6.3.2; câmera/microfone falham em apps de terceiros. Nada sobre canal de dados | [Confirmado] fórum oficial |
| WebGL 1 e 2, Web Audio | Devem funcionar | [Suposição] |
| WebGPU | Não existe | [Suposição forte] |
| Resolução do app | 1920x1080 (padrão) ou 1280x720. "UHD não é suportado para apps web" | [Confirmado] |
| Módulos (`type="module"`) em app empacotado | **Bloqueados** em emulador webOS 6. Resposta da LG: sirva por um servidor | [Confirmado] |
| Cookies em app empacotado | Não funcionam; só em app hospedado | [Confirmado] |
| Guardar para usar sem internet (service worker) | Em app empacotado, não. Em hospedado, talvez | [Testar na TV] |
| Limite de memória | A LG não publica. Samsung publica 300 a 380 MB na plataforma dela | [Confirmado só Samsung] |
| TV como servidor na rede | LG: "a TV não pode ser usada como servidor". Há projeto da comunidade que faz mesmo assim | [Confirmado] os dois |

**Mini servidor na TV ou só cliente? Só cliente.** A LG desaconselha, o Node da TV é a versão 8 (de 2018), e a página
do celular (https) seria impedida de falar com um endereço local sem cadeado. Não vale a pena.

### 5.3 Como se empacota um app

[Confirmado, LG] `appinfo.json` com: `id` (tipo `br.com.solverone.nome`), `title`, `main`, `icon` (80x80),
`type: "web"`, `version`. Opcionais úteis: `largeIcon` (130x130), `resolution`, `disableBackHistoryAPI`,
`splashBackground`. O pacote `.ipk` é montado pelo script Python que você já usou (o do free-my-tv), sem ferramenta da LG.

O aviso que você viu no site da LG (campo `requiredACG`): é um sistema de permissões novo, **não suportado no webOS 25
ou anterior**. Na sua TV não muda nada. Só importa para quem publicar na loja para TVs novas (webOS 27). Colocar
`"requiredACG": []` desde já é inofensivo [Suposição].

### 5.4 Modo Desenvolvedor

- [Confirmado, LG, dez/2025] a sessão dura **1000 horas** (41,7 dias). Os "50 horas" de guias antigos estão
  desatualizados.
- Renova no botão **EXTEND**, com internet. Se expirar, não dá para estender: no próximo reinício os apps são
  apagados e é preciso instalar de novo.
- [Confirmado] uma conta de desenvolvedor só fica logada em **uma TV por vez**.
- [Confirmado] a própria LG diz que o Modo Desenvolvedor é para teste, não para uso permanente.
- Existe renovação automática feita pela comunidade, por um endereço não documentado da LG. Fica só como informação:
  não é oficial e pode parar.

**Conclusão: o Modo Desenvolvedor serve para a sua casa e para testar. Não é jeito de entregar app a cliente.**

---

## 6. Emparelhamento celular ↔ TV

### 6.1 As opções

| Opção | Veredito |
|---|---|
| **WebRTC ponto a ponto, canal de dados** | Melhor atraso. É o que seus jogos já usam. Risco: a LG não garante na TV |
| **Supabase Realtime Broadcast como cano** | Reserva que funciona sempre. Tem cota (abaixo) |
| WebSocket por outro servidor | Igual ao anterior, exigiria servidor novo |
| Celular fala direto com a TV pela rede local | Não: página https não pode abrir conexão sem cadeado; o Chrome ainda pede uma permissão nova de "rede local" (desde o Chrome 142/147) |
| TV como servidor | Não (seção 5.2) |
| Bluetooth / NFC | Não é viável na TV |

### 6.2 A cota do Supabase [Confirmado]

| Limite | Grátis | Pro |
|---|---|---|
| Mensagens por segundo (projeto inteiro) | 100 | 500 |
| Mensagens por mês | 2 milhões | 5 milhões |
| Conexões ao mesmo tempo | 200 | 500 |

Conta: 30 mensagens por segundo = 108 mil por hora. **Os 2 milhões acabam em cerca de 18 horas de jogo por mês.** E o
limite de 100 por segundo é do projeto todo, ou seja, uns 3 pares jogando ao mesmo tempo. Passar do limite derruba as
conexões.

O que isso significa: como reserva, o Supabase serve. Como cano principal de um produto com vários clientes, não.
Medidas [Suposição]: no cano reserva mandar 15 por segundo e interpolar na TV; e, se o cano direto não funcionar na
LG, avaliar um retransmissor próprio barato (não pesquisei preços nesta rodada).

### 6.3 Pareamento e segurança

- QR na TV com o endereço e a sala; código curto digitado como reserva.
- Quatro letras são 456.976 combinações: dá para adivinhar. [Suposição] Melhorias: o código vale por 2 minutos ou até
  alguém entrar; a TV mostra "celular conectado, confirme"; o QR carrega um segredo longo que é o nome real do canal,
  e o código curto só serve para achar esse segredo.
- Privacidade: no modo "só pontos", nem criar faixa de vídeo. Dizer na tela que a imagem não sai do celular.

### 6.4 Wi-Fi com isolamento (rede de visitante, alguns roteadores de operadora)

O cano direto falha, porque os aparelhos não se enxergam. A saída padrão é um retransmissor (TURN). Para tráfego só de
pontos (uns 35 MB por hora) o custo é desprezível: Cloudflare dá 1.000 GB grátis por mês [Confirmado]. A credencial do
TURN não pode ficar no site estático; teria de vir de uma função no Supabase. Na prática, o cano reserva pela nuvem já
resolve esse caso sem TURN.

### 6.5 Atraso esperado

| Etapa | Tempo típico | Origem |
|---|---|---|
| Espera pelo quadro da câmera (30 por segundo) | 0 a 33 ms | conta |
| Leitura do corpo no celular | 10 a 40 ms | [Suposição] depende do aparelho |
| Rede, cano direto no Wi-Fi | 1 a 10 ms | alegação da AirConsole |
| Rede, pela nuvem em São Paulo | 13 ms no servidor + ida e volta da sua internet | benchmark Supabase |
| Desenho na TV | até 17 ms | conta |
| Atraso da tela da TV | ~10 ms em modo jogo; muito mais fora dele | sites secundários, série UP7000 |

Total grosseiro [Suposição]: **80 a 150 ms no cano direto; 120 a 220 ms pela nuvem em São Paulo.** Referências:
0,1 s é o limite do "instantâneo" (Nielsen) [Confirmado]; Kinect ficava em 60 a 100 ms. Dança e contagem toleram 150 a
250 ms **se a nota for dada pelo relógio do celular, não pela hora de chegada**. Ponteiro começa a incomodar acima de
100 a 150 ms.

Truques para esconder o atraso [Suposição, prática comum]: cada pacote leva número de ordem e hora da captura; os
relógios são combinados com um pingue-pongue; a TV desenha com 50 a 100 ms de amortecimento e interpola; pacotes
atrasados são jogados fora; os pontos viajam como números inteiros compactos (33 pontos cabem em ~240 bytes).

---

## 7. Lógica de jogo e de exercício

Esta é a parte que falta nos seus apps. Tudo aqui roda em JavaScript puro e é pequeno.

### 7.1 Tirar o tremor: filtro One Euro

[Confirmado, artigo de Casiez e colegas, 2012] Um filtro que suaviza muito quando a pessoa está parada e pouco quando
ela se move rápido. Dois botões: `min_cutoff` (menos tremor, mais atraso) e `beta` (menos atraso em movimento rápido).
Regulagem publicada: comece com `beta = 0` e `min_cutoff = 1`; parado, baixe o `min_cutoff` até o tremor sumir; depois
mova rápido e suba o `beta` até o atraso ficar aceitável.

```
alfa(fc, dt) = 1 / (1 + (1 / (2π·fc)) / dt)
velocidade   = (x − xAnterior) / dt
velSuave     = mistura(velAnterior, velocidade, alfa(1 Hz, dt))
fc           = min_cutoff + beta · |velSuave|
xSuave       = mistura(xAnterior, x, alfa(fc, dt))
```

Um filtro por coordenada, por junta. Melhor filtrar o **resultado** (o ângulo, a posição do ponteiro) do que empilhar
filtros pesados sobre os pontos.

### 7.2 Confiança de cada ponto

[Confirmado] cada ponto do MediaPipe traz `visibility`. Regras [Suposição]: ponto vale se a confiança passar de 0,5 a
0,65 (regular com gravações); se sumir, segura o último valor por até 0,2 a 0,3 s e depois vira "desconhecido"; ponto
desconhecido não conta nem a favor nem contra; se faltar junta essencial do exercício, mostrar "ajuste a câmera" em vez
de dar nota.

### 7.3 Ângulo de uma junta

```
u = A − B ;  v = C − B           (B é a junta: joelho, cotovelo...)
em 3D:  ângulo = arccos( (u·v) / (|u|·|v|) )
em 2D:  ângulo = | atan2( u.x·v.y − u.y·v.x ,  u.x·v.x + u.y·v.y ) |
```

Em 2D, usar os pontos em pixels de verdade (corrigindo a proporção do vídeo). Para exercício, preferir os pontos em
metros do MediaPipe, ou pedir a vista certa: de lado para agachamento, flexão e prancha.

Expectativa honesta [Confirmado em estudos]: contar repetições acerta de 82% a 88% em estudo com MediaPipe; medir
ângulo exato erra de 15° a 25° com uma câmera. Então: **contagem e dicas simples ("desça mais", "costas retas"), sim;
avaliação médica ou de fisioterapia, não.** Dicas sempre relativas à própria pessoa, nunca um número absoluto.

### 7.4 Contar repetições: máquina de estados com folga

```
config: junta = joelho ; baixo = 100° ; alto = 160° ; faseMínima = 250 ms ; repMínima = 800 ms

se estado == "em pé"  e ângulo < baixo por 250 ms  →  estado = "embaixo"
se estado == "embaixo" e ângulo > alto  por 250 ms  →  se passou 800 ms da última: conta +1 ; estado = "em pé"
```

A distância entre 100° e 160° é a folga que impede de contar duas vezes. Os números são ponto de partida: o certo é
calibrar por pessoa (uma repetição de aquecimento define o "em pé" e o "embaixo" dela). Pausar enquanto a junta estiver
desconhecida.

Alternativa pronta do Google [Confirmado, guia do ML Kit]: classificar a pose por vizinhos mais próximos (k-NN), com
cerca de 100 exemplos por pose, e contar quando a probabilidade sobe e desce.

### 7.5 Comparar poses (dança)

Normalizar antes [Suposição alinhada ao Google]: mover a origem para o meio do quadril, dividir pelo tamanho do tronco,
e comparar **direções dos membros** (braço, antebraço, coxa, perna) em vez de posições. Assim altura e distância da
câmera deixam de importar.

[Confirmado, artigo Move Mirror do Google, 2018] semelhança de cosseno entre os vetores normalizados, e uma versão que
dá menos peso às juntas de baixa confiança.

```
para cada membro:  semelhança = produto escalar( direçãoJogador , direçãoReferência )      (de −1 a 1)
nota do quadro   = média ponderada pela confiança (braços com peso maior)
nota do passo    = melhor nota dentro de ± 250 ms em torno da batida
faixas (regular com gravações): ≥ 0,90 Perfeito ; ≥ 0,75 Bom ; ≥ 0,55 OK
```

Somar um termo de **movimento** (direção da velocidade dos membros), senão ficar parado na pose final pontua. A Ubisoft
também usa "medidas de movimento" [Confirmado]. DTW (alinhar duas sequências no tempo) fica para passos de 1 a 2
segundos; a janela de ±250 ms é mais barata e resolve a maior parte.

Coreografia de referência: rodar o **mesmo modelo** sobre um vídeo de referência, fora do jogo, e guardar uma linha do
tempo compacta em JSON com as batidas. Cuidado legal [Suposição, não é parecer jurídico]: baixar vídeo do YouTube fere
os termos dele e coreografia e música têm dono. Gravar o próprio dançarino ou licenciar.

### 7.6 Ponteiro tipo Wii

Posição do pulso em relação ao ombro, medida em "larguras de ombro", mapeada para a tela; filtro One Euro; zona morta
para não tremer; ganho que aumenta com a velocidade. Clique por **permanência** (ficar 0,8 a 1,2 s em cima do botão,
com um anel enchendo) é o mais confiável a 2 ou 3 metros. Botões grandes (15% da largura) com "ímã". Deslize: velocidade
do pulso acima de um limite por 0,1 s, com pausa de 0,5 s. Mãos (pinça, palma aberta) só funcionam bem de perto.

### 7.7 Gravar e reproduzir (o que mais faz falta)

Gravar os pontos com a hora, em JSON, anotar "aqui tem 10 agachamentos", e passar a gravação pelo motor sem câmera.
Cada ajuste vira um teste que se repete. É o que transforma "regular no olho" em engenharia.

---

## 8. Apps direto na TV (adendo A)

### 8.1 Bolinho de Chuva como ícone no menu da TV

Você já tem isso pronto no `kids`: a pasta `tv-lg-kids` é uma casca que abre `solverone.com.br/kids/`. Falta instalar
(mesmo roteiro do free-my-tv, que funcionou em 17/Nov).

### 8.2 Casca que abre o endereço x app com os arquivos dentro

| | Casca que abre o endereço (hospedado) | Arquivos dentro do pacote (empacotado) |
|---|---|---|
| Atualização | Chega sozinha, sem reinstalar | Precisa montar e instalar pacote novo |
| Abrir sem internet | Só se o cache funcionar [Testar na TV] | Sim |
| Tempo de carga | "Depende da conexão" (LG não dá números) | Mais rápido na abertura |
| Módulos JS, cookies, cache | Funcionam | **Bloqueados** (seção 5.2) |
| Combina com o `motor-pixel` e o `motor-camtv` (módulos) | Sim | Não, sem reescrever |
| Loja da LG | Tipo oficial de app | Tipo oficial de app |

**Recomendação: casca hospedada**, com uma tela de espera bonita dentro do pacote e, se o teste confirmar, o cache do
site para abrir rápido na segunda vez. O ganho de carga do empacotado existe, mas a LG não o quantifica e ele custa as
atualizações automáticas e os módulos. [Testar na TV] medir os dois com cronômetro.

### 8.3 Navegar com o controle

[Confirmado, LG] Todo app **tem de** funcionar com as setas (códigos 37 a 40) e OK (13), além do ponteiro do Magic
Remote. Voltar é o código 461; na primeira tela, mostrar a saída do sistema. Todo item selecionável precisa mostrar que
está em foco. Recomendado: botões de pelo menos 75x75 px, letras de 20 px ou mais, sinal de carregamento.
Isso hoje não existe nos jogos do Bolinho, que foram feitos para dedo. É trabalho novo por jogo.

### 8.4 Loja oficial da LG (para não expirar)

- [Confirmado] Três etapas: envio, teste, aprovação. Exige o pacote, imagens, roteiro de uso e uma lista de conferência
  preenchida. **Cada atualização passa por aprovação de novo.**
- [Confirmado, terceiros] Vendedor pessoa física ou empresa; a LG não publica taxa nem prazo. Estimativas: 3 semanas a
  3 meses.
- [Não confirmado] Se o Brasil aceita pessoa física, quais documentos, regras para app infantil, se existe canal de
  teste fechado. Conferir no cadastro do LG Seller Lounge.
- **Risco:** se o app depender de WebRTC, a LG pode barrar (é recurso de parceiro).

### 8.5 Homebrew Channel (só informação)

É uma loja não oficial da comunidade. Instala pelo Modo Desenvolvedor e funciona sem destravar a TV para instalar apps,
mas continua preso à validade do Modo Desenvolvedor. Os recursos extras exigem destravar a TV por falhas que a LG
corrige, e o próprio projeto avisa de riscos de segurança. **Não recomendo para você nem para clientes.**

### 8.6 Outras marcas

| Marca | Como é um app web | Navegador (modelos 2021 → 2026) | Loja |
|---|---|---|---|
| LG webOS | `.ipk` | 79 → 132 | LG Seller Lounge |
| Samsung Tizen | `.wgt`, assinado com certificado preso à TV | 76 → 130 | Samsung Seller Office |
| Android TV / Google TV | APK com uma janela de navegador dentro | o WebView do sistema, que se atualiza | Google Play (exige navegação por setas e banner) |
| Fire TV | aceita app web | — | Amazon |
| Roku | não aceita HTML | — | — |

Estratégia de código único [Suposição]: uma "tela de TV" em HTML/CSS/JS simples, escrita para o navegador mais antigo
(o da Samsung 2021), com navegação por setas, e três embalagens finas (LG, Samsung, Android TV) que só mudam os códigos
das teclas e a saída. **E o caminho sem instalar nada continua valendo em todas: abrir o endereço no navegador da TV
ou num aparelho ligado na HDMI.**

Sobre a caixinha BTV B13: só achei páginas de vendedores, que se contradizem (Android 11 ou 13, com ou sem Google
Play). Nada confiável. Olhar em Configurações → Sobre na própria caixa.

---

## 9. Espelhar x transmitir (adendo B)

**Espelhar** = o celular filma a própria tela e manda o vídeo para a TV. **Transmitir** = o celular só diz à TV o que
mostrar, e a TV busca e desenha sozinha.

| Tecnologia | O que é | Sua TV tem? | Atraso (fontes fracas) |
|---|---|---|---|
| Screen Share / Miracast (no Samsung: Smart View) | Espelhar por Wi-Fi Direct | **Sim** [Confirmado, página do produto no Brasil] | 80 a 200 ms ou mais, com engasgos |
| AirPlay 2 | Espelhar ou transmitir vídeo, da Apple | **Sim** [Confirmado] | ~150 ms espelhando |
| Google Cast / Chromecast | Transmitir (a TV roda uma página receptora) | **Não.** A LG só pôs Cast nos modelos 2024 e em alguns 2022/2023 [Confirmado] | 100 a 700 ms espelhando aba |
| DLNA | Mandar arquivo de foto/vídeo/música | não citado na página | segundos |
| ThinQ | Controle remoto e compartilhamento | Sim | não cria cano rápido |
| Cabo USB-C → HDMI | Espelhar por fio | S24 suporta | só o atraso da TV (~10 ms em modo jogo) |

Não consegui números de laboratório confiáveis para o atraso sem fio; os estudos acadêmicos existem, mas não abri os
dados. O que é certo: **espelhar soma filmar + comprimir + Wi-Fi + descomprimir + processamento da TV**, e o rádio do
celular ainda se divide entre o roteador e a TV. Por isso você achou lento.

**Para o Purpurina:**

| Caminho | Atraso | Esforço | Comentário |
|---|---|---|---|
| Espelhar (Smart View) | pior | zero | o que você já viu que é lento |
| Cabo USB-C → HDMI | ótimo | comprar um adaptador | solução imediata, sem código; celular fica preso ao fio |
| Abrir o jogo no navegador da TV | desenho local | baixo | precisa rodar no Chromium 79 |
| **Desfile como página da TV + celular mandando o estado** | desenho local | médio | **recomendado**: o celular vira o camarim, a TV vira a passarela |
| Receptor Chromecast | bom | médio/alto | exige comprar um aparelho, taxa de US$ 5 e tem risco de fechar por inatividade |

---

## 10. Câmera do celular sem TV (adendo D)

É o caso do RiseONE. Tudo da seção 7 vale, sem a parte de rede.

- **Desempenho:** sem número publicado para o S24. Referência: celular médio de 2021 (Pixel 5) fazia 32 quadros/s no
  modelo leve. O S24 deve passar disso [Suposição]. Um relato isolado mostra que **sem aceleração gráfica** até um
  celular topo de linha cai para ~9 leituras/s: o motor precisa conferir se a aceleração ligou de verdade.
- **iPhone / Safari** [relatos da comunidade]: app instalado na tela inicial pede a câmera a cada abertura; trocar de
  tela pode congelar a câmera; o sistema mata a aba sob pressão de memória. Cuidados: manter um fluxo de câmera só
  durante a sessão, não trocar de rota com a câmera ligada, reabrir a câmera se congelar, evitar o modelo pesado.
- **Tela acesa:** suportado no Chrome e no Safari 16.4+; em app instalado no iPhone só ficou confiável no iOS 18.4.
- **Posição:** corpo inteiro no quadro, 2 a 3 m, celular na altura do quadril ao peito, quase em pé, luz de frente.
  Uma "checagem de pronto" que só libera quando tornozelos e pulsos aparecem por 1 segundo.

### Privacidade e LGPD (não é parecer jurídico)

- A lei lista "dado biométrico" como sensível, sem definir o termo.
- [Confirmado] A ANPD ainda **não** publicou guia final sobre biometria (está na agenda 2025–2026). Uma nota técnica de
  nov/2025 só reúne contribuições: a maioria entende que é biométrico quando serve para **identificar** alguém; uma
  minoria inclui qualquer traço físico, até o jeito de andar. Processar no aparelho é apontado como salvaguarda.
- Leitura para o projeto [Suposição]: pontos do corpo usados na hora, sem guardar, sem enviar e sem identificar ninguém
  provavelmente não são dado sensível. Mas a imagem da pessoa é dado pessoal mesmo processada localmente, e o tema não
  está fechado. **Projetar como se houvesse fiscalização.**
- Crianças: vale o melhor interesse da criança (art. 14). Há lei nova de 2025 (ECA Digital) com deveres de verificação
  de idade; só vi por fontes secundárias. Confirmar com advogado antes de qualquer coleta.
- Práticas: aviso antes de pedir a câmera ("a imagem é processada só neste aparelho; nada é gravado nem enviado");
  indicador de câmera ligada e parar com um toque; nunca guardar imagem; guardar só resultado (repetições, pontos);
  bibliotecas no próprio repositório; nos apps infantis, aviso voltado ao responsável.

---

## 11. Modelo treinado pela própria pessoa (material F, SignDETR)

### 11.1 O que é o SignDETR, conferido

- [Confirmado] O repositório existe (52 estrelas, último envio em 17/Ago/2025). O README diz licença MIT, mas **não há
  arquivo de licença** na raiz. O peso pré-treinado tem ~108 MB.
- [Confirmado] O título real do vídeo é "I trained a Sign Language Detection Transformer (here's how you can do it
  too!)". Os números do vídeo (30 imagens por classe, 24 h de treino) não pude conferir fora da transcrição.
- É um projeto **didático**: detector de objetos escrito do zero, em Python, que reconhece 3 formas paradas de mão.

### 11.2 Por que não serve no celular

Um modelo equivalente pesa de 43 a 167 MB [Confirmado]. Não há medição publicada de velocidade em celular; a
expectativa é de poucos quadros por segundo [Suposição]. E ele olha **pixels** para descobrir algo que os **pontos da
mão** já dizem. Também não entende movimento.

### 11.3 Onde cada abordagem entra

| Quero reconhecer | Abordagem |
|---|---|
| Movimento do corpo (exercício, dança) | Pose pronta (33 pontos) + ângulos e comparação. **Sem treino** |
| Gesto de mão como comando (joinha, palma, vitória) | MediaPipe Gesture Recognizer: 8 gestos prontos [Confirmado] |
| Um sinal ou pose **meu**, ensinado por mim | Pontos da mão ou do corpo → normalizar → **k-NN treinado no navegador**, em segundos |
| Gesto com movimento (acenar, deslizar, círculo) | Trajetória dos pontos + regra de velocidade, ou comparação de sequências com 3 a 10 exemplos |
| Objeto físico (brinquedo, carta) | Aí sim detector de objetos, com licença livre (D-FINE-N, YOLOX). Para carta impressa, QR é mais simples |
| Rosto ou expressão | Face Landmarker + regras. Atenção redobrada à LGPD |

**Quando um basta:** quase sempre a pose pronta resolve o corpo, e o k-NN sobre pontos resolve o "meu gesto". Detector
de pixels só para objetos.

### 11.4 Custo de cada caminho

| Caminho | Onde treina | Tempo | Dados |
|---|---|---|---|
| **k-NN sobre pontos** | no próprio celular | segundos | 20 a 100 exemplos por classe |
| Classificador de gestos do Google (Model Maker) | Colab | minutos | — **mas a instalação está quebrada e sem manutenção desde 2024** [Confirmado] |
| Detector pequeno (D-FINE-N, YOLOX) | Colab com placa | dezenas de minutos | 100+ caixas rotuladas por classe |
| DETR do zero (SignDETR) | notebook | ~24 h | ~30 imagens por classe |

Licenças [Confirmado]: **Ultralytics YOLO é AGPL ou pago** (vale até para modelos treinados por você); YOLO-NAS proíbe
uso comercial. Livres: D-FINE, YOLOX, NanoDet, RT-DETR original, RF-DETR nos tamanhos Nano a Large.

### 11.5 O que reaproveitar do vídeo

A **experiência**, não o código: o ciclo coletar → conferir → treinar → testar → usar ao vivo, como um "modo ensinar"
dentro do app; captura guiada de N exemplos por classe, variando posição e distância; uma classe "nenhum"; limite de
confiança; lista de classes gravada junto com o modelo; e um teste separado antes de valer.

### 11.6 Libras

Reconhecer Libras de modo geral é problema de pesquisa em aberto. Os conjuntos públicos têm dezenas de sinais ou poucos
sinalizantes. Um vocabulário pequeno e fixo é viável. Único conjunto claramente aberto que achei: MINDS-Libras (20
sinais, 12 pessoas, licença CC BY 4.0). O VLibras faz o caminho inverso (texto para avatar), não reconhece.

---

## 12. Quadro final (adendo E) e recomendação por produto

| Caminho | Atraso | Desempenho | Esforço | Manutenção | Outras marcas |
|---|---|---|---|---|---|
| (i) Endereço no navegador da TV | bom (desenho local) | limitado pelo Chromium 79 | baixo | automática | **todas** as que têm navegador |
| (ii) App webOS (casca hospedada) | igual ao (i) | igual, em tela cheia, com ícone | médio | automática; casca reinstalada a cada ~41 dias ou loja | só LG; Samsung e Android pedem embalagem própria |
| (iii) Espelhar | ruim | vídeo comprimido | zero | nenhuma | LG, Samsung, Apple |
| (iii-b) Cabo USB-C → HDMI | ótimo | nítido, na proporção do celular | um adaptador | nenhuma | qualquer TV com HDMI |
| (iv) Transmitir (Cast) | bom | bom | médio/alto | conta de desenvolvedor Google | só onde há Chromecast (sua TV não tem) |
| (v) **Segunda tela com dados leves** | **bom a ótimo** | o melhor para jogo | médio/alto na 1ª vez, baixo depois (é o motor) | a do motor | todas, pelo caminho (i) |

**Por produto:**

| Produto | Recomendação |
|---|---|
| **Bolinho de Chuva** | Hoje: instalar a casca `tv-lg-kids` que já existe. Depois: navegação por setas jogo a jogo. Para clientes: endereço no navegador da TV; loja da LG só quando houver jogos prontos para controle |
| **Purpurina** | Hoje: cabo USB-C → HDMI. Depois: caminho (v), desfile na TV e camarim no celular |
| **RiseONE** | Primeiro só no celular: consertar o espelho com o motor (seção 3.2), ganhar contagem de repetições. TV depois, como segunda tela opcional |
| **Dança na Sala** | Caminho (v) com a comparação de poses da seção 7.5. Tirar o envio de vídeo do celular para a TV |
| Vruuum!, Rabiscolândia, GLUB GLUB | Caminho (v) sem câmera: o celular como controle (suas pendências 59 e 40 do `kids`) usa a mesma "sala" do motor |

---

## 13. O motor `motor-camtv`: proposta (M1, M2, M3)

**Tudo nesta seção é proposta. Nada foi implementado.** Segue o molde do `motor-pixel` 1.0, que é o motor mais recente
da plataforma.

### 13.1 Princípios

- Um motor, muitos módulos, muitos apps. Cada app pede só o que usa.
- JavaScript puro, sem build. Funções com nomes em português.
- **Nenhum nome de app dentro do motor.** Regra de exercício, coreografia, texto e arte ficam no app.
- Bibliotecas e modelos copiados no repositório, sem CDN.
- O motor não guarda dado de pessoa, não tem chave, não chama servidor por conta própria. O endereço e a chave pública
  do cano reserva são **passados pelo app**.
- Versão em pasta (`/1.0/`); o app aponta para a versão, nunca para "a última".
- **Duas metades com regras diferentes:** o lado do celular pode usar JavaScript moderno; o lado da TV (`sala`, `tv`,
  `desenho`) é escrito para o Chromium 76/79.

### 13.2 Estrutura de pastas proposta (M1)

```
motor-camtv/                      publicado em solverone.com.br/motor-camtv/
  README.md  ARQUITETURA.md  PENDENCIAS.md  CREDITOS.md  MUDANCAS.md  CLAUDE.md
  1.0/
    motor.js                      carregador sob demanda (CamTV.usar)
    manifesto.json                módulos, dependências, bibliotecas e peso máximo
    sw-motor.js                   gaveta compartilhada no aparelho (como no motor-pixel)
    modulos/
      nucleo/        relógio, laço de quadros, vigia de desempenho, modo leve
      camera/        abrir a câmera certa (frente, trás, grande-angular), espelho, tela acesa, checagem de pronto
      corpo/         adaptador do Pose Landmarker (lite/full), formato único de saída
      maos/          adaptador do Hand Landmarker e dos 8 gestos prontos
      suave/         filtro One Euro, confiança, segurar-e-esquecer
      medidas/       ângulos, normalização, direções dos membros
      contagem/      máquina de estados de repetições, calibração por pessoa
      comparar/      semelhança de poses, janela de tempo, DTW curto
      ponteiro/      pulso → cursor, permanência, deslize
      ensinar/       modo ensinar: k-NN sobre pontos, salvar e carregar em JSON
      sala/          pareamento (QR + código), dois canos, relógio combinado, pacote compacto   [lado TV: JS antigo]
      tv/            teclas do controle, foco, sair, detectar o navegador da TV                 [JS antigo]
      desenho/       esqueleto e avatar simples em canvas, com interpolação                     [JS antigo]
      gravador/      gravar pontos em JSON e reproduzir sem câmera
    libs/            mediapipe (wasm + modelos .task), com licença
  cascas/            lg/ (appinfo, ícones, instalador), samsung/, android-tv/   (embalagens finas)
  exemplos/          diagnostico.html, esqueleto-na-tv.html, contar-agachamento.html, ensinar-gesto.html
  apps/              como cada app usa o motor (bolinho-de-chuva, riseone)
  docs/              relatório, decisões, roteiro de teste na TV
  teste/             testes automáticos com gravações (sem câmera)
  gravacoes/         clipes de pontos anotados (sem imagem, sem pessoa identificável)
```

### 13.3 Interface pública proposta (M2)

```js
import { CamTV } from 'https://solverone.com.br/motor-camtv/1.0/motor.js';

// ---- no celular ----
const { camera, corpo, suave, medidas, contagem } = await CamTV.usar('camera', 'corpo', 'suave', 'medidas', 'contagem');

const cam = await camera.abrir({ lado: 'frente', largura: 640, aoPronto, aoFaltarCorpo });
const leitor = await corpo.criar({ modelo: 'leve', pessoas: 1, ritmo: 15, aoProgresso });
const conta = contagem.criar({ junta: 'joelho', baixo: 100, alto: 160, calibrar: true });

leitor.aoLer(cam, function (pose) {                 // pose: { pontos[33], metros[33], confianca[33], hora }
  const p = suave.filtrar(pose);
  conta.alimentar(medidas.angulo(p, 'quadril', 'joelho', 'tornozelo'), p.hora);
});
conta.aoContar(function (n) { /* o app mostra "3" */ });

// ---- celular e TV ----
const { sala } = await CamTV.usar('sala');
const s = await sala.entrar({ codigo: 'ABCD', reserva: { url, chavePublica } });   // dados do app, não do motor
s.mandar('pose', p);            // escolhe o cano sozinho
s.aoReceber('estado', fn);

// ---- na TV (JS antigo) ----
var s = CamTV.sala.abrir({ aoParear: fn, aoSair: fn, reserva: { url: url, chavePublica: chave } });
s.mostrarQR(elemento);
s.aoReceber('pose', function (pose) { desenho.esqueleto(canvas, pose); });
```

Formato único da pose: 33 pontos sempre na mesma ordem, **sem espelhar**, com `x` e `y` de 0 a 1, `z` relativo, uma
confiança por ponto, os mesmos pontos em metros quando o modelo oferecer, e a hora da captura. Espelhar é decisão de
quem desenha. Trocar o motor de visão no futuro muda só o módulo `corpo`.

### 13.4 Como cada app consome (M3)

Duas opções, e a resposta muda por lado:

| | Endereço comum (`solverone.com.br/motor-camtv/1.0/`) | Cópia dentro do app |
|---|---|---|
| Uma cópia só no aparelho para todos os apps | Sim (gaveta compartilhada) | Não: 28 MB repetidos por app |
| Atualizar | Publica uma vez | App por app |
| Funciona fora do domínio da SolverONE | Não | Sim |

**Recomendação: endereço comum com versão presa**, igual ao `motor-pixel`. Como todos os apps moram na mesma origem,
os 28 MB do MediaPipe são baixados uma vez e servem para todos. Exceção: app que um dia saia do domínio leva cópia.

Interruptores do RootifyONE: o motor é neutro e não lê interruptor. **O app** pergunta antes de chamar o motor e
declara no seu `recursos-do-app.json`. Ids sugeridos, a confirmar com você: `camera-corpo` (ler o corpo), `camera-maos`,
`modo-tv` (já existe no RiseONE), `ensinar-gesto`. Trabalho demorado (baixar os 28 MB) passa pelo módulo `Fundo` do
app, com a regra de Wi-Fi.

### 13.5 O que sai dos apps quando o motor existir

`baguncolandia/esqueleto.js`, `peluqueria/esqueleto.js`, `vruuum/maos.js`, `rabiscolandia/mao.js`, as cinco `sala.js` /
`rede.js`, o bloco 43 do RiseONE e as quatro pastas `tv-lg*` viram chamadas ao motor. É a mesma regra do `motor-pixel`:
se dois apps usam, mora no motor.

---

## 14. Plano em fases

Os esforços são estimativas minhas [Suposição], em "dias de trabalho do Claude Code com você testando", e dependem da
fase 0.

| Fase | O que entrega | Esforço | Depende de |
|---|---|---|---|
| **0. Diagnóstico** | Uma página que roda na TV e no S24 e responde as 12 perguntas da seção 15 | 1 dia | você com a TV e o celular |
| **1. Protótipo mínimo** | Celular lê a pose, manda os pontos, TV desenha o esqueleto. Os dois canos, QR, relógio combinado | 1 a 2 dias | fase 0 |
| **2. Núcleo do motor 1.0** | `nucleo`, `camera`, `corpo`, `suave`, `medidas`, `gravador`, exemplos e testes com gravações | 4 a 6 dias | fase 1 |
| **3. RiseONE** | Espelho consertado, contagem de repetições com calibração, 5 exercícios, dicas simples | 4 a 6 dias | fase 2 |
| **4. `sala` + `tv` + `desenho`** | Segunda tela pronta para qualquer app; celular como controle (pendência 59 do `kids`) | 4 a 6 dias | fase 1 |
| **5. Purpurina** | Desfile na TV, camarim no celular | 3 a 5 dias | fase 4 |
| **6. Dança na Sala** | Comparação de poses, nota por passo, uma coreografia própria | 6 a 10 dias | fases 2 e 4 |
| **7. `ensinar` + `maos` + `ponteiro`** | Modo ensinar gesto, gestos prontos, ponteiro tipo Wii | 4 a 6 dias | fase 2 |
| **8. Outras marcas e loja** | Cascas Samsung e Android TV; decisão sobre a loja da LG | 3 a 5 dias + prazo da LG | fase 4 |

### O protótipo mínimo de 1 dia (fase 1)

Duas páginas no repositório do motor, em `exemplos/`:
- **TV** (`esqueleto-na-tv.html`, JS antigo): mostra um QR e um código; quando o celular entra, desenha o esqueleto em
  canvas com interpolação, e mostra na tela qual cano está em uso, o atraso medido e os quadros por segundo.
- **Celular** (`esqueleto-no-celular.html`): abre a câmera, roda o Pose Landmarker lite já hospedado, filtra, manda
  pacotes compactos a 15 por segundo, e mostra leituras por segundo e temperatura aproximada (tempo de leitura subindo).

Sucesso = esqueleto na TV acompanhando a pessoa com atraso abaixo de 200 ms por 5 minutos, nos dois canos.

---

## 15. Riscos e o que não foi possível confirmar sem testar na TV de verdade

### As 12 perguntas da página de diagnóstico (fase 0)

1. Qual é o navegador exato da TV (`navigator.userAgent`) e o que ele suporta: `?.`, `??`, `gap` em flex, `:has()`,
   `<dialog>`, módulos, WebAssembly, WebGL2, Web Audio, IndexedDB.
2. `RTCPeerConnection` e canal de dados existem e **conectam** com o S24? No navegador da TV e dentro da casca.
3. A TV resolve os endereços `.local` do WebRTC na sua rede?
4. Conexão `wss://` com o Supabase funciona dentro da casca hospedada?
5. O cache (service worker) registra na casca hospedada?
6. Quanto tempo leva para abrir: casca hospedada x arquivos empacotados?
7. Quantos quadros por segundo a TV desenha um canvas com esqueleto e um fundo simples?
8. O Three.js do `motor-pixel` roda no Chromium 79?
9. O modo jogo da TV vale para apps, ou só para HDMI?
10. No S24: quantas leituras por segundo nos modelos lite e full, com a aceleração gráfica ligada de verdade?
11. No S24: depois de 20 minutos, quanto cai o ritmo (calor)?
12. No S24: o navegador deixa escolher a câmera traseira grande-angular?

### Riscos

| Risco | Tamanho | O que fazer |
|---|---|---|
| WebRTC não funcionar ou ser barrado na LG | alto | dois canos desde o primeiro dia |
| Cota do Supabase estourar com vários clientes | médio | 15 por segundo no reserva; avaliar retransmissor próprio |
| Chromium 79 não rodar o que os jogos usam | médio/alto | "tela de TV" própria e leve; testar cedo |
| Modo Desenvolvedor não escalar para clientes | certo | navegador da TV ou loja oficial |
| Calor e bateria em sessões longas | médio | vigia de desempenho; teste de 20 minutos |
| Exercício visto de frente medir mal | certo | pedir a vista certa; dicas relativas; não prometer precisão |
| iPhone instalado na tela inicial perder a câmera | médio | manter um fluxo só; oferecer abrir no Safari |
| LGPD para imagem de criança | médio | processar no aparelho; aviso ao responsável; advogado antes de coletar |
| Loja da LG: Brasil, pessoa física, app infantil | desconhecido | perguntar no cadastro antes de investir |
| Biblioteca do MediaPipe em "pista" separada | baixo/médio | testar o contorno; senão, reduzir o ritmo |

### Não confirmado em fonte

- Quadros por segundo no S24 ou em celular médio, em qualquer motor.
- Aceleração WebGPU no S24 brasileiro.
- Atraso real de Miracast, AirPlay e Cast em medição de laboratório.
- Se o Pose Landmarker filtra o tremor sozinho na web.
- Se o webOS "6.5.3" mudou o navegador em relação ao 6.0 (a tabela da LG indica que não).
- Taxa, prazo, documentos e regras infantis da loja da LG no Brasil.
- Tudo sobre a caixinha BTV B13.
- Posição final da ANPD sobre biometria e os deveres exatos da lei de 2025 para crianças.
- Os limites numéricos das fórmulas (100°/160°, 0,90/0,75/0,55, ±250 ms): são pontos de partida, não valores de fonte.

---

## 16. Fontes

### Lista do Gemini, conferida

| Item | Resultado |
|---|---|
| Google for Developers, série "ML on Web with MediaPipe" | **Existe.** Ep. 8 "Pose landmark detection": https://www.youtube.com/watch?v=oaK74yozU9g (também há episódios de gestos e de mãos). Guia oficial: https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker/web_js |
| The Coding Train, ml5.js BodyPose | **Existe.** "Pose Estimation with ml5.js": https://www.youtube.com/watch?v=T99fNXTUUaQ . Os vídeos antigos de PoseNet usam interface desatualizada |
| Nicholas Renotte, "AI Gym Tracker" | **Existe.** "AI Pose Estimation with Python and MediaPipe | Plus AI Gym Tracker Project": https://www.youtube.com/watch?v=06TE_U21FK4 . É Python, não JavaScript |
| LearnOpenCV, comparação | **Existe em parte.** "YOLOv7 Pose vs MediaPipe" (18/Out/2022): https://learnopencv.com/yolov7-pose-vs-mediapipe-in-human-pose-estimation/ . Artigo sobre MoveNet: não encontrado |
| Café e Computação (@cafeecomputacao) | **O canal existe** (Python, em português). Vídeo de estimativa de pose nele: **não encontrado** |
| "Math.atan2 para ângulos", "cosseno", "DTW" | Corretos como técnica. Faltou dizer: corrigir a proporção do vídeo e usar a vista certa |
| "MoveNet rápido, bom para dança" | Verdade em 2021. Hoje a biblioteca está parada; não recomendo como base |

### Fontes principais (todas acessadas em 19/Nov/2026)

**TV LG e lojas**
- Tabela de navegador por versão do webOS: https://webostv.developer.lge.com/develop/specifications/web-api-and-web-engine
- WebRTC "só para parceiros" (07/Jan/2026): https://forum.webostv.developer.lge.com/t/inquiry-regarding-webrtc-support-on-webos-tv/28059
- WebRTC "não aberto" e relatos (2023): https://forum.webostv.developer.lge.com/t/webrtc-support-in-webos-tv/2306
- Módulos bloqueados em app empacotado (ago/2026): https://forum.webostv.developer.lge.com/t/module-script-loads-blocked-under-file-origin-for-installed-webos-app-no-cors-workaround-found/28351
- `appinfo.json`: https://webostv.developer.lge.com/develop/references/appinfo-json
- Tipos de app (hospedado x empacotado): https://webostv.developer.lge.com/develop/getting-started/web-app-types
- Controle remoto e navegação: https://webostv.developer.lge.com/develop/guides/magic-remote e https://webostv.developer.lge.com/develop/guides/back-button
- Modo Desenvolvedor: https://webostv.developer.lge.com/develop/getting-started/developer-mode-app ; 1000 horas: https://forum.webostv.developer.lge.com/t/whats-the-way-to-extend-dev-time-with-webos-cli/28033
- "A TV não pode ser servidor": https://forum.webostv.developer.lge.com/t/open-http-server/9585
- ACG: https://webostv.developer.lge.com/develop/guides/acg-guide
- Aprovação na loja: https://webostv.developer.lge.com/distribute/app-approval-process e https://webostv.developer.lge.com/distribute/app-self-checklist
- Homebrew Channel: https://github.com/webosbrew/webos-homebrew-channel
- Samsung, navegador por ano: https://developer.samsung.com/smarttv/develop/specifications/web-engine-specifications.html
- Android TV, requisitos: https://developer.android.com/docs/quality-guidelines/tv-app-quality
- Just Dance Now em TVs LG (webOS 22+): https://www.justdancenow.com/FAQ?lang=en

**Espelhar x transmitir**
- Página do produto 65UP7750PSB: https://www.lg.com/br/tvs-e-soundbars/tv-uhd-4k/65up7750psb/
- Google Cast só a partir de 2024: https://www.gadgetguy.com.au/lg-tv-chromecast-webos-24-update/
- Receptor Cast e registro (US$ 5): https://developers.google.com/cast/docs/web_receiver e https://developers.google.com/cast/docs/registration
- Adaptador USB-C → HDMI da Samsung: https://www.samsung.com/ph/support/mobile-devices/how-to-use-a-samsung-usb-c-to-hdmi-adapter/

**Motores de pose**
- Pose Landmarker: https://developers.google.com/edge/mediapipe/solutions/vision/pose_landmarker
- Cartão do modelo BlazePose GHUM 3D (16/Abr/2021): https://storage.googleapis.com/mediapipe-assets/Model%20Card%20BlazePose%20GHUM%203D.pdf
- BlazePose no navegador (mai/2021): https://blog.tensorflow.org/2021/05/high-fidelity-pose-tracking-with-mediapipe-blazepose-and-tfjs.html
- MoveNet (17/Mai/2021): https://blog.tensorflow.org/2021/05/next-generation-pose-detection-with-movenet-and-tensorflowjs.html
- Licença da Ultralytics: https://www.ultralytics.com/license
- RTMPose: https://github.com/open-mmlab/mmpose/tree/main/projects/rtmpose
- WebGPU no Android: https://developer.chrome.com/blog/new-in-webgpu-121
- Defeito do Web Worker: https://github.com/google-ai-edge/mediapipe/issues/5527

**Pareamento e lógica**
- Limites do Supabase Realtime: https://supabase.com/docs/guides/realtime/limits ; benchmark: https://supabase.com/docs/guides/realtime/benchmarks
- Permissão de rede local no Chrome: https://developer.chrome.com/blog/local-network-access
- Cloudflare TURN, preços: https://developers.cloudflare.com/realtime/pricing/
- Filtro One Euro: https://gery.casiez.net/1euro/
- Classificação de poses e contagem (Google ML Kit): https://developers.google.com/ml-kit/vision/pose-detection/classifying-poses
- Move Mirror (19/Jul/2018): https://blog.tensorflow.org/2018/07/move-mirror-ai-experiment-with-pose-estimation-tensorflow-js.html
- Just Dance com câmera, Ubisoft (28/Out/2025): https://www.ubisoft.com/en-sg/game/just-dance/2026/news-updates/2UBGhF4xWMLAVVFMmTCWwn/revolutionising-motion-gaming-the-camera-controller-feature
- Limite de 0,1 s (Nielsen): https://www.nngroup.com/articles/response-times-3-important-limits/
- Erro de ângulo com uma câmera: https://pmc.ncbi.nlm.nih.gov/articles/PMC13730411

**Modelo treinado**
- SignDETR: https://github.com/nicknochnack/SignDETR e https://www.youtube.com/watch?v=o_MGqeFMAGE
- Gesture Recognizer: https://developers.google.com/edge/mediapipe/solutions/vision/gesture_recognizer
- Model Maker quebrado: https://github.com/google-ai-edge/mediapipe/issues/5968
- k-NN no navegador: https://github.com/tensorflow/tfjs-models/tree/master/knn-classifier
- D-FINE: https://github.com/Peterande/D-FINE ; YOLOX: https://github.com/Megvii-BaseDetection/YOLOX
- MINDS-Libras: https://zenodo.org/records/2667329

**LGPD**
- Nota técnica da ANPD com contribuições sobre biometria (nov/2025): https://www.gov.br/anpd/pt-br/acesso-a-informacao/participacao-social/outras-acoes/ts-01-2025/nota-tecnica-no-23-2025-con1-cgn-anpd.pdf

### Fontes extras que valem o tempo

- "7 dos and don'ts of using ML on the web with MediaPipe" (Google, 05/Out/2023): https://developers.googleblog.com/2023/10/7-dos-and-donts-of-using-ml-on-web-with-mediapipe.html
- LiteRT.js, o novo motor do Google para navegador (09/Jul/2026): https://developers.googleblog.com/en/litertjs-googles-high-performance-web-ai-inference/
- Contador de repetições em JavaScript com MediaPipe Tasks: https://github.com/1yakub/PoseEstimationProject
- Clone aberto de Just Dance (Python; serve pelas ideias de pontuação): https://github.com/diegoperea20/OpenJustDance
- UFSC, material de aula sobre reconhecimento de poses: https://lapix.ufsc.br/ensino/visao/visao-computacionaldeep-learning/deep-learningreconhecimento-de-poses/
