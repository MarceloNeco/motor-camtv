# Motor CamTV — arquitetura (leia antes de mexer)

Briefing para qualquer pessoa ou IA. Escrito em 19/Nov/2026, junto com a pesquisa
`docs/RELATORIO-motor-de-pose-TV-LG-2026-11-19.md`.

> **Estado: aprovado pelo dono em 19/Nov/2026, ainda sem código.** A estrutura de pastas e a interface abaixo foram
> aprovadas ("aprovo tudo"). Onde o texto diz "proposta", leia "aprovado, a implementar". Mudança de rumo depois dos
> testes na TV volta a passar pelo dono. Quem for implementar começa pelo `PENDENCIAS.md` (#2).

## 1. O que é e para quem serve

O motor de **câmera e TV** da SolverONE. Faz duas coisas:

1. **Enxergar** o corpo e as mãos pela câmera do celular, no próprio aparelho, e transformar isso em medidas úteis
   (ângulos, repetições, semelhança com uma pose, posição de um ponteiro, "meu gesto").
2. **Ligar celular e TV**: parear por QR ou código, e trocar mensagens pequenas com pouco atraso, para a TV ser a
   segunda tela e o celular ser o sensor ou o controle.

Não pertence a nenhum app. É irmão do `motor-pixel` (desenho vetorial e 3D) e do `leitor-ocr` (leitura de documentos).

| Cliente | O que usa |
|---|---|
| RiseONE | corpo, contagem de repetições, dicas de postura; TV opcional |
| Bolinho de Chuva (repositório `kids`): Bagunçolândia, Peluqueria, Dança na Sala | corpo + segunda tela |
| Bolinho de Chuva: Vruuum!, Rabiscolândia, GLUB GLUB, Purpurina | celular como controle + segunda tela; mãos |
| Qualquer app futuro com câmera ou TV | o que precisar |

**Três camadas** (igual ao `motor-pixel`):

| Camada | O que tem |
|---|---|
| Motor (este repositório) | Técnicas neutras: ler pose, filtrar, medir, contar, comparar, parear, transportar |
| Dados de cada app | Regras de cada exercício, coreografias, gestos ensinados, arte, textos |
| App | Telas, idioma, contas, anúncios, interruptores do RootifyONE |

**Regra de ouro:** se só um app usa, fica no app.

## 2. O que já foi provado

- **17/Nov/2026:** um app HTML empacotado (`.ipk`) foi instalado e aberto numa **LG 65UP7750PSB, webOS 6.5.3**, pelo
  Modo Desenvolvedor oficial, a partir de um computador Windows, com o `install.py` do projeto free-my-tv (MIT). Isso
  valida conta LG, Developer Mode, Key Server e a instalação pelo Wi-Fi.
- **Já existe no repositório `kids`** (não testado em TV real): quatro "cascas" de app LG (`tv-lg*`), o MediaPipe
  hospedado em `baguncolandia/libs/mediapipe/`, e a ideia de mandar só pontos para a TV (`baguncolandia/esqueleto.js` +
  `sala.js`).
- **Não provado:** WebRTC entre o celular e a TV de verdade; desempenho do Chromium 79 da TV; leituras por segundo no
  Galaxy S24. Ver `PENDENCIAS.md` #1.

## 3. Arquitetura recomendada pela pesquisa

```
        CELULAR (Chrome / Samsung Internet)                    TV (LG webOS 6.5 = Chromium 79)
 ┌──────────────────────────────────────────┐          ┌──────────────────────────────────────┐
 │ camera   640x480, 30 quadros/s           │          │ sala     mostra QR + código           │
 │ corpo    MediaPipe Pose, 15 a 20 leit./s │          │          recebe, ordena, combina relógio│
 │ suave    filtro One Euro + confiança     │          │ desenho  amortece 50-100 ms, interpola │
 │ medidas  ângulos, normalização           │   cano   │          esqueleto / avatar em canvas  │
 │ sala     pacote ~240 bytes, com relógio  │ ───────▶ │ (regras e placar são do app)          │
 │                                          │ ◀─────── │ tv       setas, OK, Voltar (461), foco │
 └──────────────────────────────────────────┘          └──────────────────────────────────────┘
        cano A: direto, WebRTC canal de dados (mais rápido; a LG não garante)
        cano B: pela nuvem, Supabase Realtime Broadcast (reserva que sempre funciona)
        troca automática: tenta o A por 3 a 5 s; se não abrir, usa o B
```

Decisões e o porquê:

| Decisão | Tipo | Por quê |
|---|---|---|
| Visão só no celular | decisão | TV com navegador de 2019 e processador fraco |
| Trafegam só pontos, nunca vídeo | decisão | privacidade, leveza, menos calor |
| MediaPipe Pose Landmarker como motor de visão | decisão | 33 pontos com 3D e confiança, Apache-2.0, vivo (v1.2.3 em nov/2026), já hospedado no `kids` |
| Um adaptador (`corpo`) com formato único de saída | decisão | trocar de motor de visão no futuro sem mexer nos apps |
| Dois canos com troca automática | decisão | a LG declara WebRTC "só para parceiros" |
| Página da TV hospedada (https), aberta por casca | decisão | app empacotado bloqueia módulos JS, cookies e cache |
| Lado da TV escrito para Chromium 76/79 | decisão | cobre LG 2021 e Samsung 2021 |
| WebRTC funciona na TV do dono | **suposição** | os jogos atuais usam, mas nunca foi testado na TV real |
| 15 a 20 leituras/s bastam com interpolação | **suposição** | a medir no S24 |
| Atraso total de 80 a 150 ms no cano direto | **suposição** | soma de estimativas; a medir |
| Cache (service worker) funciona na casca hospedada | **suposição** | a testar na TV |

## 4. Estrutura de pastas (proposta)

```
motor-camtv/                      publicado em solverone.com.br/motor-camtv/ (quando o dono mandar)
  README.md  ARQUITETURA.md  PENDENCIAS.md  CREDITOS.md  MUDANCAS.md  CLAUDE.md
  1.0/
    motor.js                      carregador sob demanda (CamTV.usar)
    manifesto.json                módulos, dependências, bibliotecas e peso máximo
    sw-motor.js                   gaveta compartilhada no aparelho (como no motor-pixel)
    modulos/
      nucleo/        relógio, laço de quadros, vigia de desempenho, modo leve
      camera/        abrir a câmera certa (frente, trás, grande-angular), espelho, tela acesa, checagem de pronto
      corpo/         adaptador do Pose Landmarker (leve/completo), formato único de saída
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
  cascas/            lg/ (appinfo, ícones, instalador), samsung/, android-tv/
  exemplos/          diagnostico.html, esqueleto-na-tv.html, contar-agachamento.html, ensinar-gesto.html
  apps/              como cada app usa o motor
  docs/              relatório, decisões, roteiro de teste na TV
  teste/             testes automáticos com gravações (sem câmera)
  gravacoes/         clipes de pontos anotados (só números)
```

O que é o quê:
- **Núcleo:** `nucleo`, `camera`, `corpo`, `suave`, `medidas`, `gravador`.
- **Lógica reutilizável:** `contagem`, `comparar`, `ponteiro`, `ensinar`, `maos`.
- **Adaptador de TV:** `sala`, `tv`, `desenho`, e a pasta `cascas/`.
- **Exemplo:** tudo em `exemplos/`. Nunca vira dependência de app.

## 5. Interface pública (proposta)

```js
import { CamTV } from 'https://solverone.com.br/motor-camtv/1.0/motor.js';

// ---- no celular ----
const { camera, corpo, suave, medidas, contagem } = await CamTV.usar('camera', 'corpo', 'suave', 'medidas', 'contagem');

const cam = await camera.abrir({ lado: 'frente', largura: 640, aoPronto, aoFaltarCorpo });
const leitor = await corpo.criar({ modelo: 'leve', pessoas: 1, ritmo: 15, aoProgresso });
const conta = contagem.criar({ junta: 'joelho', baixo: 100, alto: 160, calibrar: true });

leitor.aoLer(cam, function (pose) {
  const p = suave.filtrar(pose);
  conta.alimentar(medidas.angulo(p, 'quadril', 'joelho', 'tornozelo'), p.hora);
});
conta.aoContar(function (n) { /* o app mostra o número */ });

// ---- celular e TV ----
const { sala } = await CamTV.usar('sala');
const s = await sala.entrar({ codigo: 'ABCD', reserva: { url, chavePublica } });   // dados do app, não do motor
s.mandar('pose', p);
s.aoReceber('estado', fn);

// ---- na TV (JS antigo) ----
var s = CamTV.sala.abrir({ aoParear: fn, aoSair: fn, reserva: { url: url, chavePublica: chave } });
s.mostrarQR(elemento);
s.aoReceber('pose', function (pose) { desenho.esqueleto(canvas, pose); });
```

**Formato único da pose** (contrato estável entre o motor e os apps):

| Campo | O que é |
|---|---|
| `pontos[33]` | `{x, y, z}` com x e y de 0 a 1 na imagem, **sem espelhar**; z relativo ao quadril |
| `metros[33]` | os mesmos pontos em metros, com origem no meio do quadril, quando o modelo oferece |
| `confianca[33]` | 0 a 1 por ponto |
| `hora` | relógio do celular na captura, em milissegundos |
| `quadro` | número de ordem, para a TV descartar o que chega atrasado |

Espelhar é decisão de quem desenha. Trocar o motor de visão muda só o módulo `corpo`.

## 6. Como um app usa o motor

- **Endereço comum com versão presa** (`solverone.com.br/motor-camtv/1.0/`), como o `motor-pixel`. Todos os apps moram
  na mesma origem, então os ~28 MB do MediaPipe baixam **uma vez** para todos (gaveta `motor-camtv-<versão>`).
  Cópia dentro do app só se um app sair do domínio.
- **Interruptores do RootifyONE:** o motor não lê interruptor. O **app** pergunta
  (`SolverRecursos.ligado('camera-corpo', true)`) antes de chamar o motor e declara no seu `recursos-do-app.json`.
  Ids sugeridos, a confirmar com o dono: `camera-corpo`, `camera-maos`, `modo-tv` (já existe no RiseONE),
  `ensinar-gesto`.
- **Trabalho demorado:** baixar o motor de visão passa pelo módulo `Fundo` do app e respeita a regra de Wi-Fi.
- **Textos, idiomas, regras de exercício, coreografias e arte ficam no app.**
- **Privacidade:** o app mostra o aviso antes de pedir a câmera; o motor nunca grava nem envia imagem.

## 7. O que sai dos apps quando o motor existir

| Hoje | Vira |
|---|---|
| `kids`: `baguncolandia/esqueleto.js`, `peluqueria/esqueleto.js` | `corpo` + `suave` |
| `kids`: `vruuum/maos.js`, `rabiscolandia/mao.js` | `maos` |
| `kids`: `baguncolandia/sala.js`, `peluqueria/sala.js`, `purpurina/sala.js`, `rabiscolandia/sala.js`, `batalha-naval/rede.js` | `sala` |
| `kids`: `tv-lg`, `tv-lg-kids`, `tv-lg-baguncolandia`, `tv-lg-peluqueria` | `cascas/lg/` |
| `kids`: `baguncolandia/libs/mediapipe/` (28 MB) | `1.0/libs/` (uma cópia para todos) |
| `rise-one`: bloco "43. Espelho com câmera" do `index.html` (MediaPipe vindo de CDN) | `camera` + `corpo` + `medidas` + `contagem` |

Defeitos conhecidos nesses códigos, que o motor corrige por construção: seção 3.2 do relatório.

## 8. Limites da TV que todo mundo precisa saber

- LG webOS 6.x = **Chromium 79**. Samsung 2021 = 76. Sem `?.`, `??`, `:has()`, `gap` em flex, WebGPU.
- O módulo comum da SolverONE (`janela.js`) exige Chrome 105: **a tela de TV não carrega o módulo comum inteiro**.
- App web na LG desenha em 1920x1080 ou 1280x720, nunca em 4K.
- Todo app de TV precisa funcionar com setas (37 a 40), OK (13) e Voltar (461), além do ponteiro.
- A LG declara WebRTC "só para apps de parceiros". Nunca depender só dele.
- A TV não deve ser servidor. Ela só se conecta para fora.
- **Modo Desenvolvedor:** 1000 horas (~41 dias), renovar no EXTEND; apps somem se expirar; uma conta por TV.
  Serve para a casa do dono e para testes, não para clientes.

## 9. Onde estão os materiais

| Material | Onde |
|---|---|
| Relatório completo (cópia) | `docs/RELATORIO-motor-de-pose-TV-LG-2026-11-19.md` |
| Relatório (Google Doc) | https://docs.google.com/document/d/1mua3MU3Ql3nGsyBFD4ryhy0ov74Df767OJENj9cdYik/edit |
| Pasta da pesquisa no Drive do dono: `SolverONE / 0-PESQUISAS / MOTORES / CelCamTV and FREE MY TV` | id `14Z9iy4K7BorK4CAJWTIQNfy9Bj6oWA7R` |
| Brief original (17/Nov) | arquivo `BRIEF-FABLE-motor-de-pose-TV-LG.md`, id `1o_Uoy_P4SrZdTSf9D4RhDTiAB1r9Q38U` |
| Adendo de escopo (17/Nov) | `ADENDO-escopo-2026-11-17.md`, id `1qJQyuxTVahbXbKs8QkB5Qbbvd_bmGJC5` |
| Resumos do vídeo SignDETR (18/Nov) | ids `1nvjDdK6mK2aej55VJ4q3dthZ_4BTy6TX` e `1bZpPmG9zjUWNes158U4fQ9qtg4OVyyx_` |
| Passagem para outra conta do Claude | `HANDOFF-para-conta-marceloneco-2026-11-19.md`, na mesma pasta |
| Backlog da plataforma | planilha "0-SolverONE DIRETRIZES E BACKLOG v12", aba "Ideias & Pesquisas", itens 61, 62 e 63 |
| Pendências irmãs | repositório `kids`, `PENDENCIAS.md` #7, #59 e #60 |

## 10. Próximos passos

1. ~~O dono aprova a estrutura e a interface~~ feito em 19/Nov/2026.
2. **Fase 0:** página de diagnóstico de 1 dia, na TV e no S24 (12 perguntas, seção 15 do relatório).
3. **Fase 1:** protótipo de 1 dia: o celular lê a pose e manda os pontos, a TV desenha o esqueleto, com os dois canos.
4. **Fase 2:** núcleo do motor 1.0, com gravações de teste.
5. **Fase 3:** RiseONE (espelho consertado e contagem de repetições).
6. Depois: `sala` + `tv` + `desenho`, Purpurina, Dança na Sala, modo ensinar, outras marcas e loja.

Detalhe, esforço estimado e dependências: seção 14 do relatório e `PENDENCIAS.md`.
