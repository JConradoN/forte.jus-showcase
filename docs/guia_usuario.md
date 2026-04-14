# Forte.jus — Guia do Usuário

**Para advogados e equipe do escritório**

---

## O que é o Forte.jus?

O Forte.jus é um assistente jurídico com inteligência artificial que funciona **dentro do seu escritório**, no seu próprio servidor. Ele se conecta diretamente ao PJe do TJAP, baixa seus processos e intimações automaticamente, e responde suas perguntas em português — citando a página exata onde encontrou cada informação.

Nenhum dado sai do escritório. Nenhuma mensalidade. Funciona mesmo quando a internet cai.

---

## Como o processo entra no sistema

O Forte.jus oferece **duas formas** de carregar um processo:

### Forma 1 — Download direto do PJe (integração automática)

Quando a integração com o TJAP estiver ativa, basta informar o número do processo e o sistema baixa tudo automaticamente:

- Peças processuais
- Intimações pendentes
- Movimentações recentes

O processo fica disponível para consulta em minutos, sem você precisar entrar no PJe.

### Forma 2 — Upload manual de PDF

Você exporta o PDF do PJe normalmente e carrega no sistema. Feito uma única vez por arquivo.

---

## Monitoramento automático de processos

Uma vez que um processo está no sistema, o Forte.jus **continua monitorando** o PJe em segundo plano.

Quando houver uma nova movimentação — despacho, decisão, intimação — você é avisado automaticamente:

```
⚖️ Nova intimação — Processo 0034567-89.2023.8.03.0001
Tipo: Despacho
Prazo: 5 dias úteis — vence em 02/04/2026
Resumo: O juiz solicita complementação dos documentos (fl. 43).
O processo foi atualizado e já está disponível para consulta.
```

Sem precisar verificar o PJe manualmente. Sem correr o risco de perder prazo.

---

## Como usar o chat

### 1. Acessar o sistema

Abra o navegador e acesse: **http://[endereço-do-servidor]:3000**

### 2. Escolher o modelo correto

No campo de seleção de modelo, escolha **forte.jus**.

> Os outros modelos (deepseek, qwen) são gerais. Só o **forte.jus** conhece seus processos e tem acesso ao PJe.

### 3. Fazer perguntas

Inclua o número do processo na pergunta:

> *"No processo 0034567-89.2023.8.03.0001, quais são os fatos narrados na denúncia?"*

---

## Exemplos de perguntas

### Entender o processo rapidamente

> *"No processo 0034567-89.2023.8.03.0001, quem são as partes e qual é o pedido?"*

> *"No processo 0034567-89.2023.8.03.0001, qual foi a última decisão proferida?"*

> *"No processo 0034567-89.2023.8.03.0001, há alguma perícia mencionada?"*

### Preparar audiências e prazos

> *"No processo 0034567-89.2023.8.03.0001, há alguma menção a prazo ou data de audiência?"*

> *"No processo 0034567-89.2023.8.03.0001, quais documentos foram juntados pela parte contrária?"*

### Identificar teses e argumentos

> *"No processo 0034567-89.2023.8.03.0001, quais são os argumentos da defesa?"*

> *"No processo 0034567-89.2023.8.03.0001, o juiz aplicou o princípio da insignificância? Em qual página?"*

### Revisão antes de protocolar

> *"No processo 0034567-89.2023.8.03.0001, há contradições nos fatos apresentados pelo autor?"*

### Consultar legislação

O sistema conhece o texto completo de várias leis e súmulas — e cita o artigo real, nunca inventa penas, prazos ou redação:

> *"O que diz o art. 155 do Código Penal?"*

> *"Qual é a pena para furto qualificado?"*

> *"No processo 0034567-89.2023.8.03.0001, qual artigo do CP é aplicável aos fatos narrados?"*

> *"O que diz a Súmula 231 do STJ?"*

**Legislação disponível:** Código Penal (CP), Código de Processo Penal (CPP), Código Civil (CC), Código de Processo Civil (CPC), Lei de Execução Penal (LEP), Constituição Federal (CF) e Súmulas do STJ.

Quando o sistema encontra referências a artigos no texto do processo (ex: "art. 155 do CP"), ele busca automaticamente o texto exato da lei e inclui na resposta — sem depender da memória do modelo.

---

### Elaborar minutas de petições

Você pede em linguagem natural e o sistema redige com base nos fatos reais do processo:

> *"Com base no processo 0034567-89.2023.8.03.0001, elabore uma minuta de contestação focando no princípio da insignificância."*

> *"Redija um recurso de apelação para o processo 0034567-89.2023.8.03.0001, questionando a dosimetria da pena."*

> *"Faça um resumo executivo do processo 0034567-89.2023.8.03.0001 para eu apresentar ao cliente."*

O sistema usa apenas os fatos que estão no processo — nunca inventa datas, valores ou argumentos. O advogado revisa, ajusta e assina.

---

## Como interpretar as respostas

O sistema sempre cita a página onde encontrou a informação: **(Forte.jus p. 42)** ou **(Forte.jus pp. 10-15)**.

Ao final de cada resposta há uma seção **[PONTOS DE REFLEXÃO]** com aspectos que merecem atenção do advogado. O sistema nunca toma decisões por você — ele organiza e aponta, você decide.

**Exemplo real de resposta:**

> O réu é acusado de furto qualificado com abuso de confiança (art. 155, §4º, II do CP), com bens avaliados em R$ 97,34 subtraídos das dependências do sindicato **(Forte.jus pp. 1-2)**.
>
> **[PONTOS DE REFLEXÃO]**
> - O valor é próximo ao limiar do princípio da insignificância. Vale verificar reincidência.
> - Os bens foram recuperados e devolvidos **(Forte.jus p. 2)** — relevante para dosimetria.
> - Grande intervalo entre o fato (2004) e a petição atual (2018) — verificar prescrição.

---

## O que o sistema NÃO faz

- Não protocola documentos automaticamente
- Não substitui a análise e assinatura do advogado
- Não responde sobre processos que não foram carregados

---

## Roadmap — O que vem a seguir

### Agente no WhatsApp / Telegram

Você poderá consultar processos e receber alertas **direto no celular**, sem abrir o navegador:

```
Você (WhatsApp):
  "Tem alguma coisa nova no processo do João Silva?"

Forte.jus:
  "Sim. Despacho de 25/03/2026: juiz determina realização
   de perícia. Prazo para indicar assistente técnico:
   10 dias úteis (vence 07/04/2026)."
```

Intimações chegarão como mensagem assim que forem publicadas no PJe:

```
⚖️ Nova intimação recebida
Processo: 0034567-89.2023.8.03.0001
Prazo: 5 dias úteis — vence 02/04/2026
Resumo: Juiz solicita complementação dos documentos (fl. 43).
```

### Geração de minutas (em desenvolvimento)

Em breve o sistema terá templates específicos para os tipos de peça mais comuns (contestação, apelação, agravo, memoriais), com exemplos reais do escritório como referência. Por enquanto já é possível pedir minutas livres via chat.

### Dashboard de prazos

Painel visual com todos os processos monitorados, prazos próximos e intimações pendentes.

---

## Dúvidas frequentes

**O sistema pode errar?**
Sim. Sempre confira a página citada no PDF original antes de usar qualquer informação em peça processual.

**E se o processo for muito grande?**
Processos de até 500 páginas funcionam normalmente.

**Posso usar para qualquer tipo de processo?**
Sim — cível, criminal, trabalhista, previdenciário.

**Meus dados estão seguros?**
O sistema roda no servidor do escritório. Nenhum dado, nenhum processo, nenhuma pergunta sai da sua rede local.

---

*Forte.jus — Leia menos, entenda mais.*
