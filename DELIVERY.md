# Assistente de Investimentos com IA - Decisões Técnicas de Integração no n8n

Este documento descreve as estratégias de engenharia e regras de negócio implementadas na orquestração do fluxo RPA + IA utilizando n8n, LangChain e OpenAI, cumprindo os requisitos do Desafio Completo.

> 📁 **Nota sobre o Workflow:** O arquivo `.json` contendo o fluxo exportado (workflow completo do n8n) está disponibilizado neste repositório para referência, consulta das configurações dos nodes e importação direta na plataforma.

---

## 1. Ingestão e Resiliência de Dados (Nodes: Merge e CODE JS 2)
* **Desafio:** O node `Merge` original combinava dados estruturados (CSV de produtos) e não estruturados (tabela HTML do Webhook), gerando aninhamentos imprevisíveis de propriedades JSON.
* **Solução:** Implementação de um algoritmo de "Caça de Variáveis" (Recursive Fetch) no `CODE JS 2`. Em vez de buscar dados em caminhos estáticos (hardcoded), o script utiliza uma função recursiva para varrer qualquer nível de profundidade do JSON em busca das chaves `nome`, `saldo` e `perfil`.
* **Benefício:** O fluxo tornou-se agnóstico à forma como o Payload é entregue. Ele processa perfeitamente os clientes quer eles cheguem em um Array limpo, dentro de objetos aninhados (como `body.data`), ou diretamente do Webhook original.

## 2. Higienização e Tipagem (Node: CODE JS 3)
* **Desafio:** O saldo dos clientes vinha formatado como string localizada (`"R$ 12.500,00"`), impedindo comparações matemáticas com a tabela de produtos.
* **Solução:** Aplicação de expressões regulares (Regex) `.replace(/[^\d.]/g, "")` para limpar símbolos de moeda e converter o valor string para o tipo numérico de ponto flutuante (Float/Number).

## 3. Prevenção de Alucinação da IA (Lógica Pré-LLM)
* **Decisão Técnica:** A filtragem de investimentos **não** foi delegada à Inteligência Artificial.
* **Solução:** A regra de negócio (`inv.perfil === c.perfil && saldoNum >= inv.minimo`) foi executada via código no `CODE JS 3` de forma estritamente determinística.
* **Benefício:** Isso impede totalmente as alucinações matemáticas da IA (como recomendar um produto com mínimo de R$ 5.000 para um cliente com R$ 1.500) e garante 100% de compliance com as regras da corretora. O LLM atua apenas como um "Motor Criativo", não como um banco de dados relacional.

## 4. Engenharia de Prompt e Integração LLM (Node: Message a Model)
* **Geração Dinâmica:** O prompt foi construído com variáveis injetadas (`{{ $json.prompt_llm }}`), entregando ao modelo um contexto fechado contendo apenas os dados do cliente e os produtos *já validados* na etapa anterior.
* **Estruturação Forçada (JSON Output):** O prompt exige que a saída seja estritamente um objeto JSON com as chaves `subject`, `text_body` e `html_body`. Isso permite mapear a saída diretamente para a API de envio de e-mails sem parsing complexo.
* **Segurança Jurídica:** Adição da restrição explícita *"sem prometer ganhos e sem garantias"* no prompt do sistema, mitigando riscos legais nas comunicações automatizadas da corretora.

## 5. Parse, Tratamento de Erros e Envio (Nodes: CODE JS 4, IF e Gmail)
* **Correlação de Identidade:** Como o LLM processa apenas o corpo do e-mail (sem saber o endereço de destino), o `CODE JS 4` cruza a saída gerada pela OpenAI com a lista de clientes originais baseada no index (`clientesOriginais[index]`), reconstruindo a propriedade `to` (destinatário).
* **Tratamento de Exceções (`try/catch`):** Caso a API da OpenAI falhe ou devolva um JSON malformado, o bloco `catch` intercepta o erro, preenche os campos com uma mensagem de fallback e marca o objeto como `ok: false`, evitando a quebra (crash) do workflow inteiro.
* **Validação de Destino (Regex):** O node `IF` utiliza o regex `^[^@\s]+@[^@\s]+\.[^@\s]+$` para validar se o endereço de e-mail é real.
* **Tratamento de Falhas (Consolidação):** Se o e-mail for inválido ou a geração falhar, o fluxo não descarta a execução. Ele roteia esses casos para um node de Código secundário que agrega (concatena via `<hr>`) os resultados e envia um relatório consolidado de falhas para o administrador, garantindo governança total.