# Microservices Demo

Projeto com microsserviços Java/Spring, descoberta de serviços, configuração centralizada, verificação agendada de tarefas e envio de e-mail via Mailhog sob demanda (via Feign).

## Tecnologias

- Java 21
- Maven
- Spring Boot 3.5.6
- Spring Cloud 2025.0.0
  - Config Server
  - Netflix Eureka (Server/Client)
  - OpenFeign
- Spring Data JPA + H2 (in-memory)
- Spring Mail
- Docker e Docker Compose
- Lombok

## Arquitetura

- [service-main](service-main): Config Server + Eureka Server
  - Config remota: GitHub (Lucas-319/config-server), prefixo de config: /config
  - Descoberta de serviços (Eureka) para os demais módulos
- [service-task](service-task): API de tarefas
  - Persiste em H2; expõe POST /task
  - Scheduler verifica tarefas com vencimento por data e, quando aplicável, solicita envio de e-mail ao service-notification via Feign/Eureka
  - Nesta demo, a frequência do scheduler é de 60s para tornar o resultado visível rapidamente; em um cenário real ele seria executado 1 vez ao dia
- [service-notification](service-notification): Envio de e-mails
  - Recebe requisições e envia e-mail via JavaMail para o Mailhog
- [config-server](config-server): Arquivos de configuração externa
  - [service-task.properties](config-server/service-task.properties)
  - [service-notification.properties](config-server/service-notification.properties)

Fluxo:
service-task -> cria tarefa -> scheduler verifica vencimentos -> solicita envio ao service-notification (Feign + Eureka) -> e-mail entregue no Mailhog.

## Clonar o repositório (com submódulos)

Clone já trazendo os submódulos:
```sh
git clone --recurse-submodules https://github.com/Lucas-319/microservices-demo.git
cd microservices-demo
```

Se já clonou sem submódulos:
```sh
git submodule update --init --recursive
```

Para atualizar submódulos depois:
```sh
git submodule update --remote --merge
```

## Como executar com Docker Compose

Pré-requisitos: Docker e Docker Compose.

```sh
docker-compose up --build
```

Serviços e URLs:
- Mailhog (UI): http://localhost:8025
- service-main (Config/Eureka): http://localhost:8888
  - Config Server (exemplos):
    - http://localhost:8888/config/service-task/default
    - http://localhost:8888/config/service-notification/default
  - Eureka (dashboard): http://localhost:8888/
- service-notification: http://localhost:8082
- service-task: http://localhost:8081
  - H2 Console: http://localhost:8081/h2-console
    - JDBC: jdbc:h2:mem:task
    - User: admin
    - Password: (vazio)

 

## Teste rápido

Criar uma tarefa:
```sh
curl -X POST http://localhost:8081/task \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Pagar boleto",
    "email": "usuario@example.com",
    "dueDate": "15/12/2025"
  }'
```

- O scheduler roda a cada 60s nesta demo apenas para facilitar a visualização; em produção seria diário e a verificação é baseada na data (não no horário)
- Quando a tarefa estiver elegível pelo critério de vencimento por data, o service-task solicita o envio ao service-notification via Feign/Eureka
- Dica para testar rápido: use uma dueDate com a data de hoje; em até ~60s a notificação será disparada
- Veja o e-mail no Mailhog: http://localhost:8025

Endpoints úteis:
- service-task:
  - POST /task
  - GET /server-port
  - H2 Console: /h2-console
- service-notification:
  - POST /notification (uso interno via Feign)
- service-main:
  - / (dashboard do Eureka)
  - /eureka (REST API base)
  - /config/service-task/default (Config Server)
  - /config/service-notification/default (Config Server)

## Configuração

- Configuração externa dos serviços via Config Server (prefixo /config), apontando para o repositório remoto:
  - URL remota definida em [service-main/application.properties](service-main/src/main/resources/application.properties)
- Propriedades locais de exemplo:
  - [service-task/application.properties](service-task/src/main/resources/application.properties)
  - [service-notification/application.properties](service-notification/src/main/resources/application.properties)

## Estrutura

```
config-server/           # Config externa (submódulo)
service-main/            # Eureka Server + Config Server
service-notification/    # Serviço de notificação por e-mail
service-task/            # API de tarefas (H2, JPA, Feign, scheduler)
docker-compose.yaml      # Orquestração dos serviços e Mailhog
```

 
