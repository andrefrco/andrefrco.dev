---
date: "2025-02-27T15:29:20-03:00"
title: "OpenTelemetry Operator: Instrumentação Rápida sem modificações no código"
description: "Descubra como o OpenTelemetry Operator permite instrumentar aplicações automaticamente no Kubernetes, sem alterar o código."
authors:
- André Luís Francisco
tags:
- opentelemetry
- opentelemetry-collector
- opentelemetry-operator
- instrumentation
categories:
- observability
---

Adotar observabilidade pode ser desafiador para muitas equipes de engenharia. Em ambientes com dezenas ou até centenas de microsserviços, convencer equipes de produto e engenharia a instrumentar manualmente suas aplicações pode ser um processo lento e exaustivo. A auto-instrumentação do OpenTelemetry, também conhecida como instrumentação de código zero, surge como uma solução para acelerar essa adoção, proporcionando visibilidade imediata sem exigir modificações no código.

### Tipos de Instrumentação e Seus Papéis

Podemos dividir a instrumentação do OpenTelemetry em duas categorias principais:

- **Manual**: Requer mudanças no código da aplicação para coletar dados de observabilidade — como spans, métricas ou logs. Oferece controle total e permite a inclusão de informações específicas de negócio.
- **Automática (auto-instrumentação)**: Utiliza agentes que injetam bibliotecas de instrumentação em tempo de execução para capturar automaticamente telemetria de frameworks populares, sem alterar o código. É ideal para adoção rápida, embora forneça dados mais genéricos.

### Benefícios da Auto-Instrumentação

- **Adoção Rápida**: Equipes podem começar a coletar dados de observabilidade sem modificar código.
- **Padronização**: A instrumentação é aplicada de forma consistente em todos os serviços.
- **Esforço Reduzido** – Desenvolvedores podem focar na entrega de funcionalidades em vez de integrações complexas.

### Desafios da Auto-Instrumentação

- **Informações Contextuais Limitadas**: Sinais coletados automaticamente podem carecer de atributos específicos de negócio, dificultando a extração de insights significativos.
- **Sobrecarga de Desempenho**: Embora as bibliotecas sejam projetadas para minimizar o impacto, a instrumentação em tempo de execução pode introduzir alguma latência, especialmente em aplicações de alto throughput.
- **Controle Fino Limitado**: Desenvolvedores têm menos controle sobre o que é instrumentado e como os sinais são estruturados.
- **Potenciais Problemas de Compatibilidade**: Alguns frameworks ou bibliotecas podem não ser totalmente suportados pela auto-instrumentação, resultando em lacunas na observabilidade. Além disso, nem todas as linguagens são suportadas — veja a lista completa nesta [documentação](https://opentelemetry.io/docs/zero-code/).
- **Maior Complexidade de Depuração**: Como a instrumentação é aplicada dinamicamente, a solução de problemas inesperados pode ser mais difícil em comparação com aplicações instrumentadas manualmente, especialmente para linguagens interpretadas.

### O OpenTelemetry Operator

Em ambientes Kubernetes, o OpenTelemetry Operator é um componente essencial para escalar a auto-instrumentação em todo o cluster. Ele gerencia a instrumentação automática injetando agentes nos contêineres sem exigir alterações nos manifestos das aplicações.

Embora seja possível habilitar a instrumentação OpenTelemetry modificando imagens Docker ou usando entrypoints personalizados — seja alterando cada imagem de aplicação individualmente ou usando uma imagem base compartilhada — essa abordagem aumenta a complexidade operacional. Cada atualização de imagem deve incluir as dependências necessárias, tornando a manutenção mais difícil.

O Operador utiliza webhooks de mutação do Kubernetes para interceptar solicitações de criação de pods. Quando um novo pod é criado, esses webhooks modificam automaticamente a especificação do pod para incluir o agente de instrumentação apropriado com base nas anotações e na linguagem detectada. O agente então carrega as bibliotecas de instrumentação durante o tempo de execução da aplicação para coletar dados de telemetria.

Essa arquitetura permite que o Operador detecte automaticamente aplicações e injete os agentes necessários, que então carregam as bibliotecas de instrumentação para coletar traces e métricas em tempo de execução. Além disso, permite a configuração centralizada das configurações do OpenTelemetry, como estratégias personalizadas de amostragem, enriquecimento de atributos e configurações de exportação. Isso não apenas reduz a barreira de adoção para equipes de desenvolvimento, mas também fornece uma abordagem escalável e sustentável para observabilidade.

O diagrama abaixo mostra como o Operador funciona, ilustrando o fluxo desde o webhook de mutação interceptando solicitações de deployment até os agentes de instrumentação coletando e enviando dados de telemetria para os sistemas de backend.

![Como o Operador Funciona](assets/how-otel-operator-works.png)  
_Imagem 1: Como a Instrumentação Automática Funciona com o OpenTelemetry Operator_

### Configurando Auto-Instrumentação com o OpenTelemetry Operator

Para habilitar a auto-instrumentação usando o [OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator), siga estes passos:

1. **Instale o OpenTelemetry Operator**
   ```sh
   kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
   ```

2. **Crie um OpenTelemetry Collector**
   Se necessário, crie seu próprio Coletor Otel. No exemplo abaixo, estamos usando um modo de Deployment — certifique-se de que isso se adequa à sua arquitetura. Bem, também podemos usá-lo como Sidecar, StatefulSet ou Daemonset.
   ```yaml
    apiVersion: opentelemetry.io/v1beta1
    kind: OpenTelemetryCollector
    metadata:
      name: simplest
    spec:
      config:
        receivers:
          otlp:
            protocols:
              grpc:
                endpoint: 0.0.0.0:4317
              http:
                endpoint: 0.0.0.0:4318
        processors:
          memory_limiter:
            check_interval: 1s
            limit_percentage: 75
            spike_limit_percentage: 15
          batch:
            send_batch_size: 10000
            timeout: 10s  
        exporters:
          debug: {} 
        service:
          pipelines:
            traces:
              receivers: [otlp]
              processors: [memory_limiter, batch]
              exporters: [debug]
   ```

3. **Crie uma Configuração de Auto-Instrumentação**
   Defina um recurso `Instrumentation` para aplicar auto-instrumentação às suas cargas de trabalho:
   ```yaml
    apiVersion: opentelemetry.io/v1alpha1
    kind: Instrumentation
    metadata:
      name: my-instrumentation
    spec:
      exporter:
        endpoint: http://otel-collector:4317
      propagators:
        - tracecontext
        - baggage
        - b3
      sampler:
        type: parentbased_traceidratio
        argument: "0.25"
      python:
        env:
          - name: OTEL_EXPORTER_OTLP_ENDPOINT
            value: http://otel-collector:4318
      dotnet:
        env:
          - name: OTEL_EXPORTER_OTLP_ENDPOINT
            value: http://otel-collector:4318
      go:
        env:
          - name: OTEL_EXPORTER_OTLP_ENDPOINT
            value: http://otel-collector:4318
   ```

4. **Anote Deployments para Habilitar Auto-Instrumentação**
   Escolha a anotação de acordo com sua linguagem. O exemplo abaixo usa Java — veja a [documentação completa](https://github.com/open-telemetry/opentelemetry-operator?tab=readme-ov-file#opentelemetry-auto-instrumentation-injection) para encontrar sua linguagem. Adicione a seguinte anotação aos seus Deployments do Kubernetes:
   ```yaml
   metadata:
     annotations:
       instrumentation.opentelemetry.io/inject-java: "true"
   ```

5. **Verifique a Instrumentação**
   Verifique se os sinais de telemetria estão sendo coletados em seu backend de observabilidade (Jaeger, Tempo, Loki, Prometheus, etc.) e ajuste as configurações, se necessário.

### Dicas

- **Aprenda sobre instrumentação manual**: Apesar de seus benefícios, a auto-instrumentação não substitui completamente a instrumentação manual. Para insights mais profundos e atributos personalizados, a instrumentação manual ainda é necessária. Uma abordagem híbrida — onde a auto-instrumentação fornece uma base e a instrumentação manual é adicionada a serviços críticos — pode ser mais eficaz.

- **Teste a auto-instrumentação em um sandbox**: Linguagens como Python, JavaScript e Go podem ter peculiaridades quando os agentes injetam suas respectivas bibliotecas de instrumentação em tempo de execução.

- **Monitore a sobrecarga com métricas**: Acompanhe o impacto da auto-instrumentação em sua aplicação monitorando latência, uso de CPU e alocação de memória.

- **Abstraia a implantação da instrumentação**: Automatize a implantação da instrumentação, seja por meio de templates de seus charts Helm com anotações dinâmicas ou usando ferramentas como Kyverno para aplicá-la com base em labels, namespaces ou propriedade da equipe.

- **Enriqueça logs com contexto de trace**: Mesmo com auto-instrumentação, você pode incluir `trace_id` e `span_id` em seus logs, o que ajuda muito durante a depuração. Isso pode ser feito via enriquecimento automático de logs ou diretamente na configuração do logger.

### Conclusão

A auto-instrumentação com o OpenTelemetry Operator é uma estratégia poderosa para escalar a observabilidade sem exigir esforço significativo das equipes de desenvolvimento. No entanto, para visibilidade completa e otimizada, combiná-la com instrumentação manual continua sendo essencial.

### Saiba Mais

- [Documentação do OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator)
- [Instrumentação de código zero por linguagem](https://opentelemetry.io/docs/zero-code/)
- [Instrumentação manual com OpenTelemetry](https://opentelemetry.io/docs/instrumentation/)