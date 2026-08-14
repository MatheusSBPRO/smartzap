# Operação Blessy — SmartZap para infoprodutores

> Documento de adoção. Escrito em 14/08/2026, antes de qualquer adaptação de código.
> Base: fork de `thaleslaray/smartzap` em `MatheusSBPRO/smartzap`.

## O que foi verificado na base

Antes de decidir qualquer coisa, rodei a base como está:

| Verificação | Resultado |
|---|---|
| `npm install` | 1.143 pacotes, 26s, sem erro |
| `npm run test` | **3.404 testes passando**, 109 arquivos, 2 skipped |
| `npm run build` | build de produção limpo, sem variável de ambiente configurada |
| Node | v22.14.0 (Next 16 exige 20.9+) |

A base é sólida e roda desconfigurada de propósito, porque o wizard `/install` precisa subir antes de
existir banco.

## A decisão de arquitetura: uma instância por cliente

**Verificado, não presumido.** O schema tem 37 tabelas e **nenhuma delas tem coluna de tenant**
(`tenant_id`, `account_id`, `client_id`). As credenciais do WhatsApp são únicas por instalação, lidas
de `settings` com fallback em env e cache no Redis. A rota `/api/phone-numbers` lista os números de
**um** WABA, o da instalação. O `CLAUDE.md` do projeto declara: "Single-tenant: no user accounts".

Ou seja: transformar isso em multi-tenant é reescrever a camada de dados inteira e brigar com todo
update do upstream. **Não fazemos isso.** Cada infoprodutor ganha a própria instância.

Isso não é contorno, é a postura certa por três motivos:

1. **O número fica no CNPJ do cliente.** WABA, número e histórico são dele. Se a conta dele tomar
   restrição, não contamina os outros nem a Blessy.
2. **O custo é dele.** Meta cobra por mensagem entregue, e cobrança de mídia é do anunciante.
3. **Vazamento entre clientes vira impossível por construção**, não por `WHERE tenant_id = ?`.

## Quem é dono de quê

| Item | Dono | Por quê |
|---|---|---|
| WABA + número + token da Meta | **cliente** | é a identidade comercial dele, e o custo por mensagem |
| Projeto Supabase | cliente (ou Blessy, se ele não tiver) | contém a base de contatos dele |
| Deploy (Vercel ou VPS) | a definir por cliente | ver seção de custo |
| Código do fork | Blessy | as adaptações são nossas |
| Operação (campanha, template, laudo) | Blessy | é o serviço que a gente vende |

## Custo por instância

O wizard provisiona Supabase e Upstash automaticamente na conta de quem instala. Os três limites que
estavam em aberto foram confirmados na fonte em 14/08/2026:

- **Supabase Free: 2 projetos ativos por organização**, e **projeto free é pausado após 7 dias sem
  atividade**. Duas consequências: a partir do 3º cliente é organização nova (grátis) ou Pro; e
  instância de cliente que fica uma semana sem campanha **acorda com o banco pausado**. Para cliente
  pagante, o banco é Pro — não é opcional.
- **Upstash Free: Redis com 1 banco, 500 mil comandos/mês, 256 MB e 10 GB de banda; QStash com 1.000
  mensagens/dia.** Retry conta como mensagem nova. O teto de 1 banco no free significa uma conta
  Upstash por cliente, não uma conta da Blessy servindo várias instâncias.
- **Vercel Hobby é explicitamente não-comercial.** A regra de uso justo lista "receber pagamento para
  criar, atualizar ou hospedar o site" como uso comercial — ou seja, instância de cliente pagante
  exige Pro. **A conta `matheussbpros-projects` está em Hobby hoje**, com os painéis e páginas de
  cliente já rodando nela. Resolver antes de subir a primeira instância de cliente.

O que eu confirmei na documentação da Meta em 14/08/2026:

- Cobrança é **por mensagem entregue**, não mais por conversa, desde 01/07/2025.
- **Marketing sempre é cobrado.** Utilidade e Autenticação são gratuitas dentro da janela de
  atendimento. Mensagem livre (não-template) dentro da janela é gratuita.
- Conversas de **serviço são gratuitas** desde 01/11/2024.
- A localização de cobrança em BRL para o Brasil entrou em 01/07/2026. O valor exato por mensagem sai
  do rate card da Meta, que precisa ser baixado por conta. **Não estimar de cabeça na proposta.**

Consequência prática pro produto: campanha de lançamento é categoria Marketing, logo é sempre paga.
Recuperação de venda dentro de 24h da interação do cliente pode cair em Utilidade e sair de graça. Isso
muda o preço do serviço e precisa estar na conta antes de vender.

## Regras de conformidade que a gente não negocia

A base já tem as peças (`ContactStatus: OPT_IN | OPT_OUT | UNKNOWN`, tabela `phone_suppressions`,
`isOptOutError()` mapeando erro da Meta). O que falta é a política escrita:

1. **Nenhum disparo sem opt-in registrado**, com data e origem do consentimento.
2. **Opt-out em toda campanha de marketing**, e o número entra na supressão na hora, não no dia seguinte.
3. **Lista comprada não entra.** Nunca. Queima o número do cliente e leva a conta junto.
4. **Categoria de template correta.** Marcar marketing como utilidade pra fugir de cobrança é o
   caminho mais rápido pra perder a WABA do cliente.
5. **Um número por operação.** O número da bridge da Blessy (o 8080, que sustenta os 13 clientes)
   nunca entra nisso. São mundos separados: Cloud API oficial aqui, bridge lá.

## Instância piloto da Blessy (14/08/2026)

Antes de tocar em conta de cliente, a primeira instância é nossa. Estado atual:

| Item | Valor |
|---|---|
| Projeto Vercel | `smartzap-blessy` (`prj_L4mVmIqBG8WmGnx4gIaQt6dtIYqW`), time `matheussbpros-projects` |
| URL | https://smartzap-blessy.vercel.app |
| Deploy | production READY, feito pelo CLI a partir do local na branch `blessy/operacao` |
| `/install` | acessível, sem Deployment Protection |
| `/api/health` | responde `unhealthy` com tudo `not_configured` — estado esperado antes do wizard |
| Wizard | **pendente** — falta Supabase PAT, QSTASH_TOKEN e as duas credenciais do Redis |

O deploy foi feito pelo CLI, sem conectar o Git. Isso não quebra o wizard: o
`triggerProjectRedeploy` recria a partir do último deployment de produção, não a partir do repo.
Conectar ao GitHub depois, quando fizer sentido ter deploy automático por push.

## Armadilhas do wizard, aprendidas na primeira instalação (14/08/2026)

A primeira rodada do `/install` falhou três vezes. O que aconteceu, e o que fazer com isso:

**1. O provision reprovava um token QStash que o próprio wizard tinha aprovado.** A etapa 4 do
wizard valida em `/api/installer/qstash/validate`, que bate em `qstash-us-east-1.upstash.io`. O
`validateQStashToken` do provision dizia seguir "o mesmo padrão", mas montava a URL a partir do campo
`iss` do JWT, com fallback no host global — outro endpoint. Dava "Erro ao validar token QStash" no
step 6/12, **depois** de já ter criado o projeto Supabase. Corrigido no fork em `a32f90b`: tenta
us-east-1 primeiro, depois o issuer, e agora o erro carrega status e corpo da resposta.

**2. Cada tentativa cria um projeto Supabase novo, e nunca reusa.** O código é explícito: "SEMPRE
cria um projeto novo para evitar herdar lixo". Se o nome `smartzap` existe, vira `smartzap-v2`,
`smartzap-v3`, e assim por diante. Isso não é capricho — a senha do Postgres só existe no momento da
criação, e sem ela o step 5 não monta a DB URL. **Consequência:** toda falha no meio do provision
deixa um projeto órfão ocupando uma das duas vagas do free. Antes de tentar de novo, pausar ou
apagar o órfão.

**3. O provision ignora a organização sugerida.** O preflight calcula `suggestedOrg`, mas o provision
usa `orgs[0]`. No caso não fez diferença: o limite do free é **por usuário** ("2 project limit" para
o `MatheusSBPRO`, somando todas as orgs onde ele é admin ou owner), então trocar de organização não
libera vaga. Organização nova só resolve se o dono for outra pessoa — no nosso caso, o cliente.

**4. O banco nasce em `us-east-1` por causa do plano Hobby.** A região do Supabase é derivada de
`VERCEL_REGION`, que em Hobby é sempre `iad1`. Com Pro e a função em `gru1`, o banco nasceria em
`sa-east-1`. É mais um item para a conta do upgrade: hoje toda query do inbox atravessa o Atlântico
Norte e volta.

Estado dos projetos Supabase depois da limpeza: `smartzap-v2` e `smartzap-v3` pausados (órfãos das
tentativas falhas, sem migrations aplicadas — podem ser apagados), `nossocrm` pausado, e apenas
`DASHBOARD BLESSYMIDIAS` ativo.

## Provisionamento de um cliente novo

Sequência, ainda manual, que o backlog abaixo pretende automatizar:

1. Cliente cria/entrega o Meta Business com WABA e número verificado.
2. Fork do nosso fork ou novo deploy a partir dele.
3. Deploy (Vercel ou VPS) e abrir `/install`.
4. Wizard provisiona Supabase e Upstash, roda migrações e grava credenciais.
5. `SETUP_COMPLETE=true` pra blindar as rotas de setup.
6. Registrar a instância no `registry/clients.json` da operação (ver backlog 1).
7. Subir os templates e mandar pra aprovação da Meta. Aprovação leva tempo, então é o primeiro passo
   depois do acesso, não o último.

## Backlog priorizado

**1. Instância no registry da operação.** Hoje `registry/clients.json` amarra cliente → conta de
anúncio → grupo → painel. Falta cliente → instância SmartZap (URL, WABA, número, data). Sem isso, na
terceira instância ninguém lembra qual é de quem. É o primeiro item porque é o que evita bagunça
depois, e é barato.

**2. Painel consolidado das instâncias.** Uma leitura só mostrando entregue/lido/falha por cliente,
no padrão que já existe em `blessy-paineis`. Cada instância expõe `/api/campaigns/[id]/metrics` e
`/api/health`, então dá pra ler por API com a `SMARTZAP_API_KEY` de cada uma.

**3. Pré-flight de conformidade antes do disparo.** A base já tem `/api/campaign/precheck`. Estender
pra travar: contato sem opt-in, categoria de template incompatível com o conteúdo, e ausência de
opt-out no corpo. Trava, não avisa.

**4. Tom de voz por cliente.** Template tem que soar como o cliente, não como IA e não como o Matheus.
A régua de `automations/lib/tom-de-voz.md` vale pro Matheus; cada infoprodutor precisa da sua.

**5. Provisionador.** Só depois do terceiro cliente manual, quando o padrão já estiver claro.

## O que NÃO fazer

- **Não reescrever pra multi-tenant.** Briga com todo update do upstream e quebra o isolamento que
  hoje é de graça.
- **Não migrar tudo pra VPS agora.** O `output: 'standalone'` deixa a app pronta pra Docker, mas
  QStash aparece em 27 arquivos e o Realtime do Supabase em 20. Sair da Vercel é trocar dois serviços
  gerenciados, não trocar hospedagem. Reavaliar quando o custo por instância doer.
- **Não usar isso pra prospecção fria.** É o jeito mais rápido de perder a WABA de um cliente pagante.

## Sincronizar com o upstream

O fork mantém `upstream` apontando pro repo do Thales, que está parado desde abril de 2026. Adaptação
nossa vive em branch própria, pra que `git merge upstream/main` continue barato quando ele voltar a
publicar.

```bash
git fetch upstream && git merge upstream/main
```
