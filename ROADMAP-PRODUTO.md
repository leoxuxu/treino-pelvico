# Do protótipo ao produto

Notas para a hipótese de transformar este treino num aplicativo comercial.
Escrito a partir do código que existe hoje, com estimativas de esforço marcadas como tal.

---

## 1. Onde o código está hoje

O que existe é um **guia de sessão** muito bem acabado para **um** protocolo fixo: conduz o
exercício com precisão de segundo, fala, apita e cabe numa tela. Isso já é a parte que a
maioria dos apps do gênero faz mal.

O que ele **não** é: não guarda nada, não conhece o usuário, não tem protocolo configurável,
não manda lembrete e não tem como cobrar. São quatro ausências diferentes, com custos
bem diferentes entre si.

Um detalhe a favor: a arquitetura separa **o protocolo** (a lista de blocos),
**a condução** (o relógio e as fases) e **a voz** (um banco de clipes trocável).
Essas três fronteiras são exatamente as que um produto precisaria mexer, e elas já estão limpas.

---

## 2. Relatórios de realização

É o pedido mais concreto e, felizmente, o mais barato de atender.

**O que vale registrar**, por sessão: data e hora de início, modo escolhido, se chegou ao fim ou
onde parou, quantas pausas, duração real (que difere da prevista quando há pausas) e,
se um dia houver escala de esforço, a percepção do usuário ao final.

**Dois níveis, com custos muito diferentes:**

| Nível | Como funciona | Dá para quê | Esforço |
|---|---|---|---|
| **Local** | grava em `localStorage` no próprio aparelho | histórico, sequência de dias, gráfico mensal, exportar CSV/PDF para levar à consulta | ~2 a 3 dias |
| **Com conta** | servidor, login, sincronismo entre aparelhos | acompanhar em mais de um aparelho, o profissional ver a aderência do paciente, backup | ~3 a 5 semanas, mais custo mensal de infraestrutura |

O nível local não exige servidor nenhum: o app continua um arquivo estático. Para a maioria dos
usuários, "ver meu histórico e mostrar para a fisioterapeuta" já se resolve aí — e o
**relatório exportável** é o que o profissional realmente quer na consulta.

O salto para contas só se justifica quando entrar o lado B2B (a clínica acompanhando pacientes),
porque aí vêm junto: cadastro, recuperação de senha, política de privacidade, termos de uso,
exclusão de dados, suporte. É uma mudança de natureza, não de tamanho.

---

## 3. Voz humana

A estrutura já está pronta: trocar a voz é trocar 23 arquivos de áudio. Três caminhos:

| Caminho | Resultado | Custo | Esforço |
|---|---|---|---|
| Voz premium da Apple | melhor que hoje, ainda sintética | grátis | 1 hora |
| TTS neural (ElevenLabs, OpenAI, Google) | muito próximo de humano | centavos para 23 frases | meio dia |
| Locutor profissional em estúdio | humano de verdade, com a entonação certa | algumas centenas de reais | 1 a 2 dias |

Para um produto comercial, a terceira opção é a que se paga: a voz **é** a experiência num app
que o usuário ouve de olhos fechados. Vale gravar mais do que as 23 frases — variações de
incentivo ("faltam duas", "última"), para não ficar repetitivo em uso diário.

Atenção a um ponto contratual: se for locutor, feche os direitos de uso comercial por escrito.
Se for TTS, confira na licença do serviço se o áudio gerado pode ser usado comercialmente
e redistribuído dentro do app — alguns permitem, outros limitam.

---

## 4. Plataforma

Hoje é uma página web que funciona bem no iPhone, mas não é um app.

**Caminho curto — PWA.** Falta um `manifest.webmanifest` com nome e ícones, um service worker
para funcionar offline de verdade e uma tela de abertura. São ~2 dias e transformam o site num
ícone que abre em tela cheia, sem barra do navegador. Resolve bem o uso pessoal e por indicação.

**Caminho longo — app nativo.** Só vale se o objetivo for vender na App Store e na Play Store,
ou usar recursos que a web não dá: lembretes push confiáveis no iOS, HealthKit, cobrança
pela loja. Com Capacitor, reaproveita-se quase todo o código atual; ainda assim, some contas de
desenvolvedor (US$ 99/ano na Apple, US$ 25 uma vez no Google), revisão das lojas e
atualizações. Estimativa: 3 a 4 semanas até a primeira submissão.

**Um problema técnico que precisa ser resolvido antes de qualquer venda:** a sessão congela
quando o app vai para segundo plano, porque o relógio depende do `requestAnimationFrame`.
Para uso pessoal isso é tolerável; num produto, não. A correção é recalcular o tempo a partir
do relógio do sistema ao voltar, e tocar o áudio por uma trilha agendada em vez de evento a
evento. Estimativa: 2 a 3 dias, e é pré-requisito de tudo.

---

## 5. O que um produto precisaria além disso

- **Protocolos configuráveis.** Hoje os tempos estão no código. Um produto precisa deixar o
  profissional montar o protocolo do paciente (tempos, repetições, progressão semanal) —
  idealmente gerando um link ou código que o paciente abre e já vem configurado.
- **Progressão.** O esquema PERFECT pressupõe reavaliação e aumento gradual. Um app que não
  evolui com o paciente vira um cronômetro bonito.
- **Lembretes.** "2× ao dia" só acontece se algo lembrar. É a diferença entre aderência e desistência.
- **Acessibilidade.** Público adulto e idoso: fontes maiores, contraste, modo só áudio,
  compatibilidade com leitor de tela.
- **Onboarding.** Explicar o que é contração correta, o erro comum de contrair glúteo e
  abdômen, e a instrução de não prender a respiração.

---

## 6. Jurídico, antes de cobrar

Três pontos que mudam o projeto se forem ignorados — e aqui vale consultoria de verdade,
nada do que segue é orientação jurídica:

1. **Dado de saúde é dado sensível** na LGPD. Guardar histórico de exercícios de incontinência
   numa conta é tratar dado sensível, com base legal, finalidade declarada, política de
   privacidade, direito de exclusão e segurança compatível. Enquanto tudo ficar no aparelho do
   usuário, o risco é muito menor — mais um argumento para começar pelo armazenamento local.
2. **Alegações terapêuticas.** Dizer que o app "trata incontinência" é diferente de dizer que
   ele "ajuda a seguir o protocolo prescrito". A primeira formulação aproxima o produto de
   software como dispositivo médico, com implicações regulatórias; a segunda, não. A escolha
   das palavras na loja e no site importa.
3. **Propriedade do método.** O esquema PERFECT é literatura científica publicada e pode ser
   citado, mas o material impresso da clínica (a folha de exercícios) é de autoria de quem o
   elaborou. Um produto comercial precisa do seu próprio material.

---

## 7. Monetização

| Modelo | Encaixa quando | Observação |
|---|---|---|
| Pago único (app nas lojas) | produto fechado, sem custo recorrente | não paga servidor; combina com armazenamento local |
| Assinatura | há conta, sincronismo e conteúdo novo | só se justifica com a camada de servidor |
| B2B para clínicas | a clínica paga por paciente acompanhado | ticket maior, venda mais lenta, exige contas e relatórios |
| Grátis com venda do protocolo personalizado | o app é a isca; a prescrição é o produto | depende de parceria com profissionais |

O caminho de menor atrito, dado o que existe: **PWA grátis** com histórico local e relatório
exportável, divulgado por fisioterapeutas, e a cobrança entrando depois — ou na versão de loja,
ou no painel para a clínica.

---

## 8. Ordem sugerida

| Etapa | Entrega | Estimativa |
|---|---|---|
| 1 | Corrigir o congelamento em segundo plano | 2–3 dias |
| 2 | Histórico local + relatório exportável | 2–3 dias |
| 3 | PWA (manifest, ícones, offline) | ~2 dias |
| 4 | Voz profissional gravada | 1–2 dias + custo de estúdio |
| 5 | Protocolos configuráveis por link | ~1 semana |
| 6 | Validar com fisioterapeutas de verdade, com pacientes reais | contínuo |
| 7 | Decidir entre loja e painel para clínica, com base no passo 6 | — |

As etapas 1 a 4 entregam um produto completo para uso pessoal **sem servidor, sem conta e sem
dado sensível guardado por você** — e é possível testar com usuários reais antes de qualquer
investimento maior.

---

## 9. Riscos e perguntas em aberto

- **Já existem muitos apps de exercício pélvico.** O diferencial não será o timer, e sim a
  ligação com o protocolo prescrito pelo profissional. Isso aponta para o B2B, que é venda mais
  lenta e exige material clínico próprio.
- **Validação clínica.** Sem fisioterapeutas usando e indicando, o produto é um cronômetro com
  voz. Testar com profissionais deveria vir antes de investir em lojas e infraestrutura.
- **Tema sensível.** Marketing, linguagem e privacidade precisam de cuidado especial;
  parte do público não quer o ícone visível na tela inicial.
- **Manutenção.** Loja significa atualizações periódicas, suporte e compatibilidade com cada
  nova versão de iOS e Android — custo contínuo que o protótipo atual não tem.
