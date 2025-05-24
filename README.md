# FarmTech Solutions - Serviço de Alerta AWS (Fase 7)

Este documento descreve a implementação do serviço de alerta utilizando a AWS, conforme os requisitos da Fase 7 do projeto FarmTech Solutions. O objetivo deste serviço é notificar os responsáveis sobre condições críticas na fazenda, permitindo ações corretivas em tempo hábil.

O serviço foi construído utilizando os seguintes componentes da AWS:
* **Amazon SNS (Simple Notification Service):** Para o envio das notificações (e-mail).
* **AWS Lambda:** Para executar a lógica de verificação dos dados e acionar os alertas.
* **Amazon EventBridge (CloudWatch Events):** Para agendar a execução periódica da função Lambda.
* **AWS IAM (Identity and Access Management):** Para gerenciar as permissões necessárias para os serviços interagirem.

---

##  Índice

1.  [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2.  [Configuração do Amazon SNS](#2-configuração-do-amazon-sns-simple-notification-service)
    * [Criação do Tópico SNS](#criação-do-tópico-sns)
    * [Criação da Assinatura (E-mail)](#criação-da-assinatura-e-mail)
3.  [Configuração da Permissão (IAM Role para Lambda)](#3-configuração-da-permissão-iam-role-para-lambda)
    * [Criação da IAM Role](#criação-da-iam-role)
4.  [Desenvolvimento da Função AWS Lambda](#4-desenvolvimento-da-função-aws-lambda)
    * [Criação da Função Lambda](#criação-da-função-lambda)
    * [Código da Função Lambda](#código-da-função-lambda)
    * [Teste da Função Lambda](#teste-da-função-lambda)
5.  [Agendamento da Função Lambda com Amazon EventBridge](#5-agendamento-da-função-lambda-com-amazon-eventbridge)
    * [Criação da Regra no EventBridge](#criação-da-regra-no-eventbridge)
6.  [Fluxo do Serviço de Alerta](#6-fluxo-do-serviço-de-alerta)
7.  [Considerações Finais](#7-considerações-finais)

---

## 1. Visão Geral da Arquitetura

O serviço de alerta opera da seguinte forma:

1.  O **Amazon EventBridge** aciona periodicamente (conforme agendamento) a função **AWS Lambda**.
2.  A função **AWS Lambda** executa um código Python que simula a leitura de sensores (ou processa dados reais da fazenda).
3.  Se uma condição de alerta é identificada (ex: pH baixo, umidade baixa), a função Lambda publica uma mensagem no tópico **Amazon SNS**.
4.  O **Amazon SNS** envia a mensagem de alerta para os endpoints inscritos (neste caso, um endereço de e-mail).

[Amazon EventBridge (Scheduler)] ----aciona----> [AWS Lambda (Lógica de Verificação)] ----publica (se alerta)----> [Amazon SNS (Tópico)] ----envia----> [E-mail do Usuário]


---

## 2. Configuração do Amazon SNS (Simple Notification Service)

O Amazon SNS foi utilizado para criar um canal de comunicação (tópico) e para gerenciar o envio das notificações por e-mail.

### Criação do Tópico SNS

Um tópico SNS chamado `AlertasFarmTech` foi criado para centralizar as mensagens de alerta.

* **Tipo:** Standard
* **Nome:** `AlertasFarmTech`

![Criação do Tópico SNS - Configurações Iniciais](img/aws_alerts/sns_topic_creation_settings.png)
*Legenda: Tela de configuração inicial para a criação do tópico SNS.*

Após a criação, o ARN (Amazon Resource Name) do tópico é gerado e será utilizado pela função Lambda para publicar mensagens.

![ARN do Tópico SNS Criado](img/aws_alerts/sns_topic_arn_detail.png)
*Legenda: Detalhes do tópico SNS `AlertasFarmTech` exibindo seu ARN.*

### Criação da Assinatura (E-mail)

Uma assinatura do tipo e-mail foi criada para o tópico `AlertasFarmTech`, direcionando os alertas para o endereço de e-mail do responsável.

* **Protocolo:** E-mail
* **Endpoint:** `seu-email@exemplo.com`

![Criação da Assinatura de E-mail no SNS](img/aws_alerts/sns_subscription_creation.png)
*Legenda: Formulário de criação da assinatura de e-mail para o tópico SNS.*

Após a solicitação, um e-mail de confirmação é enviado para o endereço fornecido.

![E-mail de Confirmação da Assinatura SNS](img/aws_alerts/sns_subscription_confirmation_email.png)
*Legenda: E-mail recebido para confirmação da assinatura no tópico SNS.*

Com a confirmação, a assinatura fica ativa e pronta para receber notificações.

![Status da Assinatura Confirmada no SNS](img/aws_alerts/sns_subscription_confirmed_status.png)
*Legenda: Assinatura de e-mail confirmada e ativa no console do SNS.*

---

## 3. Configuração da Permissão (IAM Role para Lambda)

Foi criada uma IAM Role para conceder à função Lambda as permissões necessárias para interagir com outros serviços AWS, como o CloudWatch Logs (para logging) e o SNS (para publicar mensagens).

### Criação da IAM Role

* **Nome da Role:** `LambdaExecutaAlertasFarmTechRole`
* **Entidade Confiável:** AWS Service - Lambda
* **Políticas de Permissão Anexadas:**
    * `AWSLambdaBasicExecutionRole` (para logs no CloudWatch)
    * `AmazonSNSFullAccess` (para publicar no SNS - em produção, idealmente seria mais restrita)

![Seleção de Entidade Confiável para IAM Role (Lambda)](img/aws_alerts/iam_role_trusted_entity_lambda.png)
*Legenda: Seleção do serviço Lambda como entidade confiável durante a criação da IAM Role.*

![Adição de Políticas de Permissão à IAM Role](img/aws_alerts/iam_role_permissions_policies.png)
*Legenda: Anexando as políticas `AWSLambdaBasicExecutionRole` e `AmazonSNSFullAccess` à IAM Role.*

![Revisão e Nomeação da IAM Role](img/aws_alerts/iam_role_naming_review.png)
*Legenda: Tela de revisão final e nomeação da IAM Role antes da criação.*

![IAM Role Criada com Sucesso](img/aws_alerts/iam_role_creation_success.png)
*Legenda: Confirmação da criação bem-sucedida da IAM Role `LambdaExecutaAlertasFarmTechRole`.*

---

## 4. Desenvolvimento da Função AWS Lambda

A função Lambda `VerificaSensoresFarmTech` contém a lógica principal do serviço de alerta. Ela é responsável por verificar (atualmente de forma simulada) os dados dos sensores e, caso necessário, enviar um alerta.

### Criação da Função Lambda

* **Nome da Função:** `VerificaSensoresFarmTech`
* **Runtime:** Python 3.9 (ou a versão utilizada)
* **Arquitetura:** x86_64 (ou a utilizada)
* **Função de Execução (Role):** Utilizada a IAM Role `LambdaExecutaAlertasFarmTechRole` criada anteriormente.

![Configurações Iniciais para Criação da Função Lambda](img/aws_alerts/lambda_creation_initial_settings.png)
*Legenda: Tela de configuração para criar uma nova função Lambda "do zero".*

![Seleção da IAM Role Existente para a Função Lambda](img/aws_alerts/lambda_execution_role_selection.png)
*Legenda: Selecionando a IAM Role `LambdaExecutaAlertasFarmTechRole` para a função.*

### Código da Função Lambda

O código Python implementado na função realiza uma simulação de leitura de sensores (pH e umidade) e envia uma mensagem para o tópico SNS caso os valores estejam fora dos limites pré-definidos. O ARN do tópico SNS é uma variável crucial no código.

```python
# Exemplo do trecho principal do código (substitua pelo seu código ou um resumo)
import json
import boto3
import os

SNS_TOPIC_ARN = 'arn:aws:sns:SUA-REGIAO:SEU-ID-CONTA:AlertasFarmTech' # SUBSTITUIR!

def lambda_handler(event, context):
    # ... (lógica de simulação e verificação) ...
    if alerta_necessario:
        sns_client = boto3.client('sns')
        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Message=mensagem_alerta,
            Subject="Alerta Urgente - Fazenda FarmTech"
        )
    # ... (retorno) ...
```
Legenda: Código Python da função VerificaSensoresFarmTech no editor do console Lambda.

Após inserir o código, ele é implantado (Deploy) para estar ativo.

Legenda: Realizando o deploy das alterações no código da função Lambda.

### Teste da Função Lambda
Um evento de teste foi configurado para simular a execução da função Lambda e validar seu comportamento, incluindo o envio do e-mail de alerta.

Legenda: Configurando um novo evento de teste no console Lambda.

O teste bem-sucedido confirma que a função executa corretamente e que a integração com o SNS está funcionando.

Legenda: Log de execução indicando sucesso no teste da função Lambda.

Como resultado do teste, um e-mail de alerta é recebido.

Legenda: Exemplo do e-mail de alerta recebido, originado pelo teste da função Lambda.

Os logs detalhados da execução podem ser visualizados no Amazon CloudWatch.

Legenda: Visualização dos logs de execução da função Lambda no CloudWatch.

## 5. Agendamento da Função Lambda com Amazon EventBridge
Para automatizar a verificação dos sensores, uma regra foi criada no Amazon EventBridge para acionar a função Lambda VerificaSensoresFarmTech em intervalos regulares.

### Criação da Regra no EventBridge
Nome da Regra: AgendaVerificaSensoresFarmTech
Tipo de Regra: Agendamento (Schedule)
Padrão de Agendamento: Taxa fixa (ex: a cada 5 minutos para teste).
Alvo (Target): A função Lambda VerificaSensoresFarmTech.
Legenda: Definindo o nome e selecionando o tipo "Regra de agendamento" no EventBridge.

Legenda: Configurando a regra para executar a uma taxa fixa (ex: a cada 5 minutos).

Legenda: Selecionando a função VerificaSensoresFarmTech como o alvo da regra agendada.

Legenda: Tela de revisão das configurações da regra antes de sua criação.

Legenda: Regra AgendaVerificaSensoresFarmTech criada com sucesso e ativa.

## 6. Fluxo do Serviço de Alerta
O fluxo de operação do sistema de alerta pode ser resumido da seguinte forma:

Agendamento (EventBridge): A regra AgendaVerificaSensoresFarmTech é disparada no intervalo configurado.
Execução da Lógica (Lambda): O EventBridge invoca a função VerificaSensoresFarmTech.
Processamento de Dados: A função Lambda executa sua lógica interna para verificar as condições dos "sensores" (atualmente simulados).
Decisão de Alerta: Se uma condição de alerta é detectada (ex: pH &lt; 5.0 ou umidade &lt; 30%), uma mensagem de alerta é preparada.
Publicação no Tópico (SNS): A função Lambda publica a mensagem de alerta no tópico SNS AlertasFarmTech.
Notificação (SNS para E-mail): O SNS envia a mensagem do tópico para todos os endpoints inscritos e confirmados (o e-mail configurado).
Ação Corretiva: O responsável recebe o e-mail e pode tomar as ações corretivas indicadas na mensagem.
## 7. Considerações Finais
O serviço de alerta implementado representa um componente crucial para o sistema de gestão da FarmTech. Ele permite o monitoramento proativo e a notificação de eventos críticos, contribuindo para a eficiência e sustentabilidade da produção agrícola.

Próximos Passos Sugeridos (para evolução do projeto):

Integrar a função Lambda com as fontes de dados reais das Fases 1, 3 ou 6 (banco de dados, AWS IoT Core, resultados de análise de imagem).
Refinar as permissões da IAM Role para seguir o princípio do menor privilégio (ex: permitir publicação apenas no tópico SNS específico).
Implementar diferentes tipos de alertas (ex: SMS, push notifications) através de mais assinaturas no SNS.
Adicionar mais inteligência à lógica de alerta na função Lambda, considerando históricos ou combinações de fatores.
