<div align="center">

# ☁️ AWS Serverless Order Processing System

Arquitetura Serverless Orientada a Eventos para Processamento de Pedidos utilizando AWS

![AWS](https://img.shields.io/badge/AWS-Cloud-orange)
![Lambda](https://img.shields.io/badge/AWS-Lambda-yellow)
![SQS](https://img.shields.io/badge/Amazon-SQS-red)
![EventBridge](https://img.shields.io/badge/EventBridge-Events-blue)
![DynamoDB](https://img.shields.io/badge/DynamoDB-NoSQL-green)
![SNS](https://img.shields.io/badge/SNS-Notifications-orange)

</div>


## 📖 Sobre o Projeto

Este projeto implementa uma arquitetura Serverless orientada a eventos para processamento de pedidos utilizando serviços gerenciados da AWS.

A solução foi desenvolvida com o objetivo de simular um cenário corporativo de processamento assíncrono de pedidos, permitindo que requisições sejam recebidas por APIs ou por arquivos JSON armazenados em um Data Lake no Amazon S3.

Durante a implementação foram aplicados conceitos de:

* Event Driven Architecture (EDA)
* Microsserviços desacoplados
* Processamento assíncrono
* Filas de mensagens
* Persistência NoSQL
* Tratamento de falhas
* Observabilidade
* Arquiteturas Cloud Native

---

# 🎯 Objetivo

Construir uma plataforma capaz de:

* Receber pedidos através de uma API REST.
* Receber pedidos através de arquivos JSON enviados ao Amazon S3.
* Validar dados antes do processamento.
* Publicar eventos para múltiplos consumidores.
* Persistir informações em banco NoSQL.
* Notificar falhas automaticamente.
* Garantir resiliência através de Dead Letter Queues (DLQ).

---

# 🏗️ Arquitetura da Solução

<img src="architecture/01-arquitetura-completa.png" width="900">

A arquitetura foi construída utilizando serviços totalmente gerenciados da AWS, eliminando a necessidade de administração de servidores e permitindo escalabilidade automática conforme a demanda.

Fluxo principal:

API Gateway → Lambda → SQS FIFO → Lambda → EventBridge → SQS → Lambda → DynamoDB

Fluxo de arquivos:

S3 → SQS → Lambda → SNS → DynamoDB

---

# 🚀 Etapa 1 — Criação da API de Entrada

O primeiro passo consistiu na criação de uma API REST utilizando Amazon API Gateway.

A API foi configurada para receber requisições HTTP contendo informações de pedidos.

Responsabilidades:

* Receber requisições dos clientes.
* Encaminhar eventos para a camada de processamento.
* Disponibilizar endpoint público.

<p align="center">
  <img src="images/02-api-gateway.png" width="900">
</p>

---

# 🧪 Etapa 2 — Teste da API

Após a publicação da API foi realizado um teste utilizando uma requisição HTTP POST contendo informações de um pedido.

Payload utilizado:

{
"pedidoId": "1001",
"clienteId": "cliente001"
}

Resultado:

* Pedido recebido com sucesso.
* Resposta HTTP retornada pela API.
* Fluxo iniciado corretamente.

<p align="center">
  <img src="images/03-api-test.png" width="900">
</p>

---

# ⚙️ Etapa 3 — Pré-Validação com AWS Lambda 

Para atuar como uma camada de borda (edge validation) logo após o API Gateway, foi implementada a função `pre-validacao.py`. A escolha por uma Lambda aqui garante escalabilidade imediata para picos de requisições e evita custos desnecessários com mensagens inválidas trafegando na infraestrutura.

Objetivos:

* Garantir que os dados mínimos estruturais estejam presentes no payload.
* Registrar logs no CloudWatch para auditoria.
* Encaminhar a mensagem sanitizada para a fila SQS FIFO.

Após validação, o pedido é enviado para uma fila FIFO.

<p align="center">
  <img src="images/04-pre-validacao-log.png" width="900">
</p>

---

# 📬 Etapa 4 — Mensageria com Amazon SQS FIFO

A fila FIFO foi utilizada para garantir:

* Ordenação das mensagens.
* Evitar duplicidade de processamento.
* Desacoplamento entre componentes.

Essa abordagem permite absorver picos de requisições sem impactar o restante da aplicação.

<p align="center">
  <img src="images/05-sqs-fifo.png" width="900">
</p>

---

# ✅ Etapa 5 — Validação de Negócio

Para processar as mensagens enfileiradas de forma assíncrona, foi desenvolvida a função `validacao-pedidos.py`. Separar essa validação da Lambda inicial segue o princípio de responsabilidade única (SRP), isolando as regras de negócio da camada de infraestrutura web.

Responsabilidades:

* Executar validações de negócio.
* Garantir consistência dos pedidos.
* Publicar eventos para o EventBridge.

<p align="center">
  <img src="images/06-validacao-log.png" width="900">
</p>


---

# 📡 Etapa 6 — Event Driven Architecture com EventBridge

Após validação, eventos são publicados no Amazon EventBridge.

O EventBridge atua como barramento central de eventos, permitindo desacoplamento total entre produtores e consumidores.

Benefícios:

* Escalabilidade.
* Baixo acoplamento.
* Facilidade de evolução da arquitetura.

<p align="center">
  <img src="images/07-event-bus-rules.png" width="900">
</p>
---

# 🔄 Etapa 7 — Processamento Assíncrono dos Pedidos

Nesta etapa, o roteamento do EventBridge aciona microsserviços especializados. A utilização de múltiplas funções Lambda isoladas foi escolhida para que o processamento, alteração e cancelamento escalem de forma independente.

Atividades realizadas pelos códigos:

* `rocessa-pedidos.py`: Consome a fila principal, atualiza o status de novos pedidos e centraliza o core da aplicação.
* `altera-pedido.py`: Acionada apenas para eventos de atualização, garantindo a consistência eventual dos dados alterados.
* `altera-pedido.py`: Acionada apenas para eventos de atualização, garantindo a consistência eventual dos dados alterados.

<p align="center">
  <img src="images/09-pedidos-pendentes.png" width="900">
</p>


<p align="center">
  <img src="images/10-processamento-log.png" width="900">
</p>


---

# 💾 Etapa 8 — Persistência com DynamoDB

Os dados gerados pelas funções Lambda são persistidos em tabelas NoSQL do DynamoDB, garantindo leitura e gravação com latência de milissegundos.

Campos armazenados:

* pedidoId
* clienteId
* statusPedido
* origem
* timestampProcessamento

<p align="center">
  <img src="images/11-dynamodb-pedidos.png" width="900">
</p>

---

# 📂 Etapa 9 — Processamento de Arquivos via Amazon S3

A arquitetura também atende fluxos em lote (batch). Arquivos JSON com múltiplos pedidos são depositados em um bucket do Amazon S3, servindo como uma segunda porta de entrada para o sistema.

Arquivos utilizados durante os testes:

* arquivo_com_pedidos.json
* s3P003-julioleite.json
* arquivo_schema_invalido.json

<p align="center">
  <img src="images/12-s3-bucket.png" width="900">
</p>

---

# 🔍 Etapa 10 — Validação de Arquivos

Assim que um arquivo chega ao S3, um evento nativo dispara automaticamente a função `validacao-s3-arquivos.py`. Utilizar uma Lambda engatilhada pelo S3 elimina a necessidade de servidores monitorando o bucket ativamente (polling).

Objetivos do código:

* Validar o schema estrutural do arquivo JSON.
* Fazer a extração (parsing) de todos os pedidos contidos no lote.
* Enviar cada pedido individualmente para a esteira principal do SQS, reaproveitando todo o fluxo já construído.

<p align="center">
  <img src="images/13-validacao-arquivos.png" width="900">
</p>

---

# 📊 Etapa 11 — Histórico de Processamento

Todas as execuções de arquivos via S3 são gravadas em uma tabela DynamoDB apartada, focada exclusivamente em auditoria e rastreabilidade de lotes.

<p align="center">
  <img src="images/14-historico-arquivos.png" width="900">
</p>

---

# 📧 Etapa 12 — Notificações com SNS

Todas as execuções de arquivos via S3 são gravadas em uma tabela DynamoDB apartada, focada exclusivamente em auditoria e rastreabilidade de lotes.

Sempre que um arquivo inválido é detectado:

* Um evento é gerado.
* Uma notificação é enviada por e-mail.
* O erro é registrado para análise.

<p align="center">
  <img src="images/15-sns-topico.png" width="900">
</p>

<p align="center">
  <img src="images/16-email-erro-validacao.png" width="900">
</p>


---

# 🛡️ Etapa 13 — Tratamento de Falhas com DLQ

Para garantir tolerância a falhas, foram implementadas Dead Letter Queues (DLQ) no SQS. Se alguma função Lambda (como a `processa-pedidos.py`ou `validacao-s3-arquivos.py`) falhar repetidamente ao tentar processar um evento, a mensagem é movida para a DLQ. Isso evita a perda de dados e permite a análise e o reprocessamento manual posterior.

Objetivos:

* Evitar perda de mensagens.
* Permitir reprocessamento.
* Facilitar troubleshooting.

DLQ de Arquivos:

<p align="center">
  <img src="images/17-dlq-arquivos.png" width="900">
</p>

DLQ de Pedidos:

<p align="center">
  <img src="images/18-dlq-pedidos.png" width="900">
</p>

---

# 📈 Resultados Obtidos

Durante o projeto foram implementados com sucesso:

✅ API Gateway

✅ AWS Lambda

✅ Amazon SQS FIFO

✅ Amazon EventBridge

✅ Amazon DynamoDB

✅ Amazon S3

✅ Amazon SNS

✅ Dead Letter Queues

✅ CloudWatch Logs

---

# 💡 Competências Demonstradas

* Arquitetura Serverless
* Event Driven Architecture
* Processamento Assíncrono
* AWS Lambda
* Amazon API Gateway
* Amazon SQS FIFO e Standard
* Amazon EventBridge
* Amazon DynamoDB
* Amazon S3
* Amazon SNS
* CloudWatch
* Tratamento de Falhas
* Cloud Native Design
* Integração entre Serviços AWS

---

# 👨‍💻 Autor

Julio Leite

Analista de Infraestrutura | Cloud Computing | AWS

Projeto desenvolvido para consolidar conhecimentos práticos em arquitetura Serverless e processamento orientado a eventos utilizando serviços AWS.
