# AWS Serverless Order Processing System 🚀☁️

## 📌 Sobre o Projeto
Este projeto prático implementa um sistema de processamento de pedidos altamente escalável e resiliente, construído 100% com serviços Serverless na AWS. A arquitetura é orientada a eventos (Event-Driven), garantindo o desacoplamento entre os microsserviços, processamento assíncrono e tolerância a falhas. 

## 🏗️ Arquitetura
![Arquitetura da Solução](arquitetura/01-arquitetura-completa.png.png)

## 🛠️ Tecnologias e Serviços AWS Utilizados
* **Amazon API Gateway:** Ponto de entrada RESTful para recebimento dos pedidos via requisições HTTP POST.
* **AWS Lambda:** Computação serverless para execução ágil das regras de negócio (pré-validação e validação profunda do pedido).
* **Amazon SQS (FIFO & DLQ):** Filas de mensagens garantindo a ordem de chegada (First-In, First-Out) e Dead-Letter Queues (DLQ) para reter e analisar mensagens com erro de processamento.
* **Amazon EventBridge:** Barramento de eventos customizado atuando como "roteador" para notificar o restante da infraestrutura sobre pedidos validados.
* **Amazon DynamoDB:** Banco de dados NoSQL utilizado para a persistência veloz e escalável dos pedidos processados.
* **Amazon S3:** Bucket para armazenamento histórico e processamento de arquivos relacionados aos pedidos.
* **Amazon SNS:** Tópico de publicações acionado para disparar notificações automáticas e alertas (ex: e-mails de erro de validação).

## ⚙️ Fluxo de Funcionamento e Evidências Práticas

### 1. Ingestão e Pré-validação
O fluxo inicia no **API Gateway** ([Ver API](./evidencias/02-api-gateway.png) | [Teste API](./evidencias/03-api-test.png)), que recebe o JSON do pedido e aciona uma função **Lambda** para checagem rápida de integridade do payload.
* [Log de Pré-validação no CloudWatch](./evidencias/04-pre-validacao-log.png)

### 2. Fila SQS e Processamento Desacoplado
Pedidos estruturalmente corretos são postados na fila **SQS FIFO** ([Ver Pedidos Pendentes](./evidencias/09-pedidos-pendentes.png)), que serve de gatilho para a Lambda principal de validação de negócios, garantindo que o Gateway não espere pelo processamento pesado.
* [Log de Validação](./evidencias/06-validacao-log.png)
* [Log de Processamento Geral](./evidencias/10-processamento-log.png)

### 3. Roteamento de Eventos e Persistência
Ao ser validado, um evento é emitido para o **EventBridge** ([Regras do Event Bus](./evidencias/07-event-bus-rules.png)). A partir daqui, consumidores independentes agem: os dados do pedido são gravados no **DynamoDB** ([Ver Tabela](./evidencias/11-dynamodb-pedidos.png)) e documentos/arquivos de relatório são versionados no **S3** ([Bucket S3](./evidencias/12-s3-bucket.png) | [Validação](./evidencias/13-validacao-arquivos.png) | [Histórico de Arquivos](./evidencias/14-historico-arquivos.png)).

### 4. Resiliência, Notificações e Tratamento de Exceções
A arquitetura foi pensada para "falhar com segurança". Erros críticos de validação acionam o **SNS** ([Ver Tópico](./evidencias/15-sns-topico.png)), que envia um e-mail de alerta para o administrador ([E-mail de Erro](./evidencias/16-email-erro-validacao.png)). Mensagens corrompidas ou falhas exaustivas são roteadas para as **DLQs**, impedindo travamentos sistêmicos.
* [Dead-Letter Queue de Arquivos](./evidencias/17-dlq-arquivos.png)
* [Dead-Letter Queue de Pedidos](./evidencias/18-dlq-pedidos.png)

## 💡 Habilidades Demonstradas
* **Escalabilidade Global e Modelo Financeiro:** Infraestrutura que explora a elasticidade nativa das regiões AWS e alavanca o modelo financeiro *pay-as-you-go*, gerando processamento massivo sem custos de ociosidade.
* **Governança e Protocolos de Segurança:** Aplicação estrita de roles do AWS IAM, concedendo permissões baseadas no princípio do menor privilégio entre os recursos (ex: Lambda apenas com acesso de escrita a filas específicas).
* **Resistência a Falhas:** Prevenção de perda de dados e isolamento de falhas utilizando microsserviços desacoplados e gerenciamento de DLQs.
