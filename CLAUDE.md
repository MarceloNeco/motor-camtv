# Motor CamTV — regras do repositório para o Claude Code

- **O que é:** o motor de câmera e TV da SolverONE. Lê o corpo e as mãos pela câmera do celular e faz o celular
  conversar com uma TV (segunda tela, celular como controle). Não pertence a nenhum app: cada app é **cliente**.
  Comece pelo `ARQUITETURA.md` e por `docs/RELATORIO-motor-de-pose-TV-LG-2026-11-19.md`.
- **Estado em 19/Nov/2026:** só documentação. **Nenhum código do motor existe.** A estrutura de pastas e a interface
  do `ARQUITETURA.md` foram **aprovadas pelo dono em 19/Nov/2026**. Próximo passo: `PENDENCIAS.md` #2.
- O dono (Marcelo) não é desenvolvedor: responda em português simples, passo a passo.
- **Regras do motor** (mesmas do `motor-pixel`): sem build; JavaScript puro; nomes de funções em português;
  **nenhum nome de app dentro de `1.0/`** (o que só um app usa fica no app); nenhuma chave ou segredo (o endereço e a
  chave pública do cano reserva são passados pelo app); bibliotecas e modelos copiados em `1.0/libs/` (sem CDN); peso de
  cada módulo no `manifesto.json`; o motor não guarda dado de pessoa.
- **Dois lados, duas regras:** o lado do celular pode usar JavaScript moderno. O lado da TV (`sala`, `tv`, `desenho`) é
  escrito para Chromium 76/79 (TVs de 2021): sem `?.`, `??`, `:has()`, `gap` em flex.
- **Privacidade:** a imagem da câmera nunca sai do aparelho e nunca é gravada. Gravações de teste em `gravacoes/` têm só
  pontos (números), sem imagem e sem pessoa identificável.
- **Repositório público:** nunca escrever IP de casa, passphrase da TV, senha, token ou dado pessoal.
- **Versões:** os apps apontam para `/1.0/`. Acrescentar = 1.1 (pasta nova, a anterior continua no ar); quebrar = 2.0.

---

# SolverONE — regras do projeto para o Claude Code

Antes de qualquer alteração neste repositório, leia e siga **inteira** a diretriz geral da plataforma em
`.claude/skills/diretrizes-gerais-apps/SKILL.md` (ela também é carregada como skill `diretrizes-gerais-apps`).

Resumo do que nunca pode ser esquecido (o detalhe está na diretriz):

- **Regra zero**: antes de implementar no app um requisito da diretriz que ainda não existe aqui, liste o que
  falta e confirme com o dono. Não crie repositórios sem permissão explícita.
  **Publicar é sempre autorizado** (dono, 02/Nov/2026): ao terminar e testar, faça o pull request e o merge sem
  perguntar e conte o que mudou.
- **Segurança**: nenhuma senha, chave de API, token ou segredo entra no código, no repositório, no chat ou em
  arquivo. Chaves de IA ficam só no cofre do navegador da pessoa. Se uma chave aparecer no chat, peça para revogar.
- **Idioma** (dono, 14/Nov/2026): textos para o usuário em **PT (Brasil), EN (EUA) e ES (Espanha)**, tradução
  humanizada; **um seletor só em todo lugar**, bandeira + sigla (🇧🇷 PT · 🇺🇸 EN · 🇪🇸 ES), componente do módulo comum,
  nunca toggle ou dropdown próprio; ES só aparece num app quando ele está 100 % traduzido (até lá, PT|EN). Detalhe na
  diretriz, "Idioma e datas". Instruções para o dono em português, passo a passo, como para iniciante (ele não é
  desenvolvedor).
- **Entrega**: um `.zip` por app, nome `<NOME APP> <VERSAO> <dd-Mmm-aaaa> <HHhMMm>.zip`, arquivos soltos na
  raiz; atualizar `versoes.json`, a versão no código, o `sw.js` e `ARQUITETURA.md`/`PENDENCIAS.md`.
- **Trabalho demorado** (foto, OCR, IA, importação, áudio) roda no módulo `Fundo`: nunca morre ao sair da tela,
  Wake Lock, aviso antes de fechar, pílula de andamento, resultado entregue onde a pessoa estiver.
- **Feedback dos usuários**: pergunta do dia, 💬 em toda tela e no AssistONE, tudo estruturado no RootifyONE.


## Aviso do dono (13/Nov/2026) — Definition of done da plataforma (vale até sair a diretriz nova; quando sair, siga-a)

A DoD da SolverONE passa a ter 12 pontos: manual e orientações, AssistONE, Novidades da versão, termos, comunicações e
e-mails, RootifyONE, ArchONE, EaglePlanONE (roadmap), retrocompatibilidade, segurança e LGPD, qualidade verificável e
registro. Vai nascer uma teia de integrações entre os apps, o RootifyONE, o ArchONE e o EaglePlanONE (eventos e comandos
com aprovação).

- **Aplicabilidade**: cada ponto vale quando se aplica ao estágio do app (com ou sem banco, só no GitHub, site aberto sem
  login). **Não** criar login, migrar para banco nem criar integração por causa deste aviso. O que não se aplica hoje:
  dizer "não aplicável hoje" e o motivo em uma linha. Quando o app ganhar login, banco ou integração nova, já nasce
  cumprindo os pontos que passaram a valer.
- **Em cada entrega**, dizer no resumo o que o RootifyONE, o ArchONE ou o EaglePlanONE precisariam saber (integração
  nova ou removida, recurso novo, item do roadmap concluído).
- **Nada de integração nova sem a regra zero.**
