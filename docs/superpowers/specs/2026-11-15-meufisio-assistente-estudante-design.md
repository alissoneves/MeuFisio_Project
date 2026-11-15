# MeuFisio — Assistente de raciocínio clínico para estudantes (v1)

Data: 2026-11-15 · Domínio: meufisio.com.br

## 1. Intenção e o porquê

A esposa do autor é estudante de fisioterapia em fase de estágio. Hoje, para investigar cada caso clínico, ela junta informação de vídeos do YouTube e livros — lento e disperso. O MeuFisio reúne isso num só lugar com uma **consulta rápida guiada por IA**: o estudante descreve a queixa (ex.: "dor no ombro"), a IA conduz perguntas como um preceptor faria ("qual região?", "dói ao apertar?", "em qual movimento piora?"), chega a **hipóteses** de condição e mostra **testes clínicos** para confirmar e **condutas/exercícios** baseados em evidência.

Ela é a primeira usuária; o objetivo seguinte é vender como SaaS. Depois deste produto virá um segundo, separado: gestão de consultório para fisioterapeutas formados (prontuário, agenda, financeiro, WhatsApp — referência: vedius.com.br). **Este documento cobre apenas o assistente do estudante.**

Contexto de uso: no celular, com uma mão, em qualquer lugar — chegando ao hospital, na sala de aula, em casa, no ônibus. O autor pediu um design **inovador e com ótima experiência de uso**.

## 2. Metas, não-metas, anti-metas

**Metas (v1)**
- A estudante consegue, em poucos minutos no celular, ir da queixa às hipóteses, testes e conduta com fontes citadas.
- Casos ficam salvos e reencontráveis; o app avisa quando um caso novo parece com um antigo.
- Visual marcante e agradável, instalável no celular.

**Não-metas (v1)** — ficam para depois, nesta ordem aproximada:
- v1.1: código de amigo, "Enviar para amigo", área "Recebidos" (seção 6.6 descreve o desenho já acordado, para o modelo de dados não fechar portas).
- Depois: busca por voz; integração com PEDro; bibliotecas próprias de Testes clínicos e Exercícios por região; corpo 3D; planos pagos/assinatura; toda a parte de gestão de consultório; app nativo nas lojas.

**Anti-metas** (falharia mesmo "funcionando"):
- Apresentar resultado como **diagnóstico fechado**. São hipóteses para o estudante confirmar com testes e com o preceptor.
- Conduta **sem fonte**, ou citação inventada (referência que não existe).
- Guardar **dados que identificam o paciente**.
- Experiência lenta/travada no celular ou que exija as duas mãos para o fluxo principal.

## 3. Restrições

- **Idioma**: interface e respostas em português do Brasil, mesmo quando a fonte está em inglês.
- **LGPD / dados de saúde**: nenhum campo de nome do paciente; apenas sexo e idade. Aviso visível no texto livre: "não inclua dados que identifiquem o paciente". Dados hospedados no Brasil (Supabase região São Paulo).
- **Fontes**: v1 usa **PubMed** (API oficial E-utilities, gratuita). PEDro só depois de verificar os termos de uso para consulta automatizada em produto comercial.
- **Custo**: começa com poucos usuários; hospedagem em camadas gratuitas/baratas. Evitar chamadas de IA repetidas para o mesmo conteúdo (cache de glossário e de evidências).
- O autor é programador experiente e vai acompanhar e revisar o código.
- **Cadastro aberto ao público desde a v1** (decisão do autor). Por isso a v1 precisa de: confirmação de e-mail; aceite de termos de uso e política de privacidade (com o consentimento da seção 5); **limites de uso de IA por usuário** (ex.: investigações e consultas ao glossário por dia) para controlar custo, com mensagem amigável ao atingir o limite; *rate limiting* e proteção básica contra abuso nas rotas de IA; recuperação de senha. A v1 é **gratuita** — sem cobrança —, mas o cadastro já guarda um campo de **plano** (ex.: `gratuito`) para a cobrança futura não exigir migração. Os valores exatos dos limites ficam a critério de quem construir, ajustáveis por configuração.

## 4. Decisões já tomadas (pelo autor, nesta conversa)

### 4.1 Stack
| Camada | Escolha |
|---|---|
| Front + back | Next.js (React + TypeScript), App Router |
| Mobile | PWA na v1; React Native/Expo só se fizer falta |
| Banco + auth + storage | Supabase (Postgres), região São Paulo |
| IA | API do Claude com *tool use* e respostas em streaming |
| Evidência | PubMed na v1; PEDro e outras depois |
| Hospedagem | Vercel (app) + Supabase (dados) |
| Pagamentos (futuro) | Asaas ou Mercado Pago (Pix/boleto) |

Uma só linguagem (TypeScript). Back-end Python foi considerado e descartado.

### 4.2 Experiência — aprovada via mockups
Mockups de referência em `docs/superpowers/specs/mockups/`:
- `pagina-inicial.html` — página inicial (layout aprovado).
- `mapa-corpo-v2.html` — tela de investigação (direção "B", aprovada).
- `icones.html` — conjunto de ícones; **estilo escolhido: C (círculo com gradiente)**.
- `meus-casos.html` — aviso de caso parecido, lista e detalhe de caso (já com o estilo de ícones escolhido).

**Página inicial**: logo "MeuFisio" ("Meu" fino + "Fisio" em negrito, símbolo de figura humana alongando — rascunho, pode ser refinado); saudação "Qual é o caso de hoje?"; **uma barra de busca única** que aceita texto e tem atalho para o mapa do corpo (o botão de voz aparece só quando a voz existir); cartão de destaque **"Investigar caso"** (nome escolhido); atalhos **Glossário**, **Meus casos** (Testes clínicos e Exercícios aparecem nos mockups mas são pós-v1 — ver 2); navegação inferior Início · Buscar · Casos · Perfil.

**Investigar caso** (direção B): silhueta humana 2D com alternância **frente/costas**; o estudante toca onde dói (ponto pulsante + rótulo, ex. "Ombro D · anterior") **ou** digita a queixa — os dois iniciam a mesma investigação, e o texto já pode pular etapas (ex.: se disse "ombro", não pergunta a região). A IA faz uma pergunta por vez, oferecendo respostas prontas para tocar além do texto livre. Uma **gaveta inferior** mostra as **hipóteses** com níveis **Alta / Média / Baixa** (não porcentagens), reordenando a cada resposta. Cartão de **próximo teste sugerido** com vídeo curto. Ao final: conduta/exercícios com **citação** de cada fonte.

**Meus casos**: salvar o caso ao final (ou a qualquer momento, como "Em investigação"); lista com busca por queixa/hipótese/teste e filtros por região; status **Confirmado** / **Em investigação**; detalhe com ficha (sexo, idade, região, duração), hipótese final, testes realizados com resultado (positivo/negativo), conduta com fonte, e **anotações pessoais** livres.

**Aviso de caso parecido**: enquanto digita a queixa na página inicial, se houver caso salvo semanticamente parecido, aparece um cartão "Você teve um caso parecido há N dias" com **Ver caso**, **Comparar** (lado a lado) e **Novo caso**. A similaridade é por **sentido**, não por palavras exatas ("ombro dói quando levanta" ≈ "dor no ombro ao elevar o braço").

**Glossário**: busca de termo/sigla; a IA explica em português; a resposta é guardada e reaproveitada na próxima busca do mesmo termo.

**Visual**: tema claro e escuro seguindo o sistema, com alternância manual; verde-menta + azul como acentos; ícones próprios em círculo com gradiente nos itens de menu e navegação, versões mais simples nos ícones pequenos.

### 4.3 Conteúdo e IA
- A IA "se vira sozinha" (não há curadoria humana de uma base própria na v1): conduz a investigação com o conhecimento do modelo e busca evidência para a conduta via ferramenta de busca no PubMed, citando os artigos (título, ano, link/PMID).
- Todo resultado deixa claro que são hipóteses para confirmação.

## 5. Decisões minhas (Claude), revisáveis
- **Vídeos de testes na v1**: vídeos do YouTube incorporados (embed). Uma tabela `teste → vídeo` é mantida manualmente (pela estudante/autor) por uma tela simples de administração ou direto no banco; quando um teste não tem vídeo cadastrado, mostra só a descrição textual de como fazer e interpretar. Sem produção de vídeo própria.
- **Login**: e-mail e senha (Supabase Auth). Login social fica para depois.
- **Citações verificáveis**: só exibir referência que veio de fato da ferramenta de busca (PMID real), nunca texto de referência gerado livremente pelo modelo. Além do PMID real, a recomendação precisa estar **apoiada no resumo (abstract)** do artigo: a ferramenta busca os abstracts e a IA só cita um artigo para uma afirmação que o abstract sustenta. A v1 mostra **apenas fontes do PubMed** (o "PEDro 7/10" do mockup `meus-casos.html` é ilustrativo e fica para depois).
- **Sem evidência encontrada**: a recomendação **não aparece como conduta**. O app diz claramente "Não encontrei evidência no PubMed para …" e sugere discutir com o preceptor. Nunca mostra conduta sem fonte.
- **Lista fixa de testes clínicos**: a v1 tem uma lista semeada de testes (nome canônico em português, sinônimos/nome em inglês, região do corpo, como fazer e interpretar, vídeo opcional). A IA escolhe testes **dessa lista** por meio de uma ferramenta; isso garante que o vídeo e a busca em Meus casos funcionem. Sem telas de biblioteca de testes na v1. Se a IA julgar necessário um teste fora da lista, ele aparece como texto, sem vídeo, e entra numa fila para ser cadastrado.
- **Sinais de alerta (red flags)**: se as respostas indicarem sinais de alerta (ex.: suspeita de fratura, déficit neurológico progressivo, sinais de síndrome da cauda equina, dor torácica), a IA **interrompe o fluxo** com um alerta destacado orientando a encaminhar/avisar o preceptor imediatamente. Faz parte da saída estruturada da IA desde a v1.
- **Conexão**: a investigação exige internet. Cada pergunta/resposta é **salva no servidor a cada turno**, para que o caso possa ser retomado se o sinal cair. Leitura offline de casos já vistos (cache do PWA) é desejável, mas não obrigatória na v1.
- **Dados fora do Brasil (LGPD)**: o banco fica em São Paulo e as funções da Vercel são fixadas na região de São Paulo (`gru1`). O texto do caso precisa ir para a API do Claude (fora do Brasil) e possivelmente para um provedor de embeddings. Isso é aceito, com estas salvaguardas: nenhum campo identificador; aviso no texto livre; tela de **consentimento + política de privacidade** no cadastro informando o processamento por provedores no exterior. Antes de vender o produto, recomenda-se revisão por um advogado de LGPD.

## 6. Comportamentos detalhados

### 6.1 Ciclo de uma investigação
1. Entrada: texto livre e/ou toque no corpo (região + lado + face anterior/posterior).
2. Dados mínimos do caso: sexo e idade (perguntados no início ou na primeira oportunidade).
3. A IA pergunta uma coisa por vez, com opções de resposta prontas quando fizer sentido.
4. A cada resposta, atualiza a lista de hipóteses (Alta/Média/Baixa) e, quando útil, sugere o próximo teste clínico; o estudante pode registrar o resultado do teste (positivo/negativo/não fez).
5. Ao final: hipóteses, testes realizados, conduta/exercícios com fontes do PubMed.
6. O estudante salva em Meus casos (pode salvar a qualquer momento e continuar depois).

### 6.2 Meus casos
Ver 4.2. Status muda manualmente entre "Em investigação" e "Confirmado" (o estudante marca quando confirmar com o preceptor). Hipótese final pode ser escolhida/editada pelo estudante.

### 6.3 Caso parecido
Disparado durante a digitação na busca inicial (com debounce) e também ao iniciar uma investigação. Considera só os casos do próprio usuário (e, a partir da v1.1, os que ele recebeu e salvou).

### 6.4 Glossário
Termo → explicação curta em português (definição, contexto clínico, sinônimos/sigla). Cache compartilhado entre usuários.

### 6.5 Privacidade
Sem campo de nome. Aviso no texto livre. Casos são privados do usuário na v1.

### 6.6 (v1.1) Compartilhamento entre amigos — desenho acordado
- Cada usuário tem um **código de amigo curto e fixo**, estilo ICQ, ex.: `MF-7K3Q` (4–6 caracteres, sem caracteres ambíguos como O/0, I/1). Pode ser compartilhado por link ou QR code.
- Adicionar amigo exige **aceitar convite**.
- "Enviar para amigo" em um caso salvo → o amigo recebe uma **cópia** em "Recebidos", com notificação no app; pode salvar no próprio Meus casos.
- **Anotações pessoais não vão por padrão**; o remetente escolhe incluir ao enviar.
- Edições posteriores do original não alteram a cópia.

## 7. Deixado para quem for construir
- Estrutura de pastas, modelo de dados detalhado, nomes de tabelas, RLS do Supabase (deve garantir que cada usuário só lê os próprios casos).
- Qual modelo Claude usar em cada etapa (equilíbrio custo × qualidade), desenho das ferramentas (*tools*) e do formato estruturado das hipóteses.
- Provedor de *embeddings* para o "caso parecido" e para o cache do glossário — precisa funcionar bem em **português**; armazenamento com pgvector no Supabase.
- Critério de quando a investigação "termina" e número típico de perguntas.
- Bibliotecas de UI/animação, implementação do PWA, estratégia de testes, CI/deploy.
- Desenho final do logo (o rascunho dos mockups serve de ponto de partida).
