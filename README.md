# Eureka Server - Serviço de Descoberta e Registro

Servidor de descoberta de serviços do ecossistema Bytecooks, responsável pelo registro e localização dinâmica de microsserviços.

## Visão Geral

O Eureka Server atua como um registro central onde todos os microsserviços do ecossistema Bytecooks se registram automaticamente. Ele permite a descoberta dinâmica de serviços, eliminando a necessidade de configurações estáticas de endereços IP e portas, além de possibilitar balanceamento de carga e tolerância a falhas.

## Tecnologias Utilizadas

- **Java 17**
- **Spring Boot 3.5.7**
- **Spring Cloud 2025.0.0**
- **Netflix Eureka Server** - Service Discovery

## Conceitos Importantes

### O que é Service Discovery?

Service Discovery é um padrão de arquitetura de microsserviços que permite que serviços se encontrem automaticamente na rede sem configuração manual de endereços. Em vez de um microsserviço precisar saber o endereço exato de outro, ele consulta o Eureka Server que mantém um registro atualizado de todos os serviços disponíveis.

### Como Funciona?

1. **Registro (Registration)**: Quando um microsserviço inicia, ele se registra automaticamente no Eureka Server, informando seu nome, endereço IP e porta
2. **Heartbeat**: Periodicamente, cada serviço envia sinais vitais para o Eureka indicando que está ativo
3. **Descoberta (Discovery)**: Quando um serviço precisa comunicar com outro, consulta o Eureka para obter a localização atual
4. **Cache Local**: Cada cliente mantém cache das localizações para melhorar performance e resiliência

### Benefícios

- **Desacoplamento**: Serviços não precisam conhecer a localização física uns dos outros
- **Escalabilidade**: Múltiplas instâncias do mesmo serviço podem ser registradas
- **Tolerância a Falhas**: Se uma instância cai, o Eureka automaticamente remove do registro
- **Balanceamento de Carga**: Distribui requisições entre múltiplas instâncias
- **Deploy Sem Downtime**: Novas instâncias são registradas sem afetar as existentes

## Pré-requisitos

- JDK 17 ou superior
- Maven 3.6+
- IDE de sua preferência (IntelliJ IDEA, Eclipse, VS Code)

## Instalação e Execução

### Clonar o Repositório

```bash
git clone git@github.com:WaldirJuniorGPN/cybercooks-eureka-server.git
cd eureka-server
```

### Compilar o Projeto

```bash
mvn clean install
```

### Executar a Aplicação

```bash
mvn spring-boot:run
```

O Eureka Server estará disponível em `http://localhost:8761`

## Configuração

### application.properties

```properties
spring.application.name=eureka-server
server.port=8761

# Configurações do Eureka Server
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.server.enable-self-preservation=true
```

**Explicação das Configurações:**

- `register-with-eureka=false`: O próprio Eureka Server não se registra como cliente
- `fetch-registry=false`: Não busca informações de registro de outros Eureka Servers (modo standalone)
- `enable-self-preservation=true`: Protege contra remoção em massa de serviços em caso de problemas de rede

### Classe Principal

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

A anotação `@EnableEurekaServer` ativa todas as funcionalidades necessárias para o servidor funcionar.

## Dashboard Web

Após iniciar a aplicação, acesse `http://localhost:8761` no navegador para visualizar:

- **Instances currently registered**: Todos os serviços atualmente registrados
- **General Info**: Informações gerais do servidor
- **Instance Info**: Detalhes de cada instância registrada
- **Health Status**: Status de saúde de cada serviço

## Integrando Microsserviços

### No microsserviço cliente (exemplo: pagamentos)

**1. Adicionar dependência no pom.xml:**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

**2. Configurar application.properties:**

```properties
spring.application.name=pagamentos
server.port=8080

eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=true
eureka.instance.prefer-ip-address=true
```

**3. Habilitar o cliente na aplicação:**

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PagamentosApplication {
    public static void main(String[] args) {
        SpringApplication.run(PagamentosApplication.class, args);
    }
}
```

## Comunicação Entre Serviços

### Usando RestTemplate

```java
@Configuration
public class RestTemplateConfig {
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```java
@Service
public class PagamentoService {
    
    private final RestTemplate restTemplate;
    
    public PagamentoService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    public Pedido buscarPedido(Long pedidoId) {
        // Usa o nome do serviço registrado no Eureka, não URL fixa
        return restTemplate.getForObject(
            "http://pedidos-service/api/v1/pedidos/" + pedidoId,
            Pedido.class
        );
    }
}
```

A anotação `@LoadBalanced` permite usar o nome do serviço no lugar da URL, e automaticamente distribui carga entre instâncias.

### Usando OpenFeign (Recomendado)

**1. Adicionar dependência:**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**2. Habilitar Feign:**

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class PagamentosApplication {
    // ...
}
```

**3. Criar interface de cliente:**

```java
@FeignClient(name = "pedidos-service")
public interface PedidosClient {
    
    @GetMapping("/api/v1/pedidos/{id}")
    Pedido buscarPorId(@PathVariable Long id);
}
```

Esta abordagem é superior porque:
- Código mais limpo e declarativo
- Tratamento automático de erros
- Suporte a circuit breaker integrado
- Menos código boilerplate

## Arquitetura do Ecossistema

```
┌─────────────────────────────────────────┐
│         Eureka Server (8761)            │
│     Service Registry & Discovery        │
└────────────┬────────────────────────────┘
             │
             │ Registration & Heartbeat
             │
    ┌────────┴────────┬─────────────┐
    │                 │             │
┌───▼────┐      ┌────▼─────┐  ┌───▼──────┐
│Pagamen-│      │ Pedidos  │  │  Outros  │
│tos     │◄────►│ Service  │◄─┤ Services │
│(8080)  │      │  (8081)  │  │          │
└────────┘      └──────────┘  └──────────┘
```

## Monitoramento e Saúde

### Health Check

O Eureka Server expõe endpoints de saúde via Spring Boot Actuator:

```
GET http://localhost:8761/actuator/health
```

### Métricas

Para habilitar métricas detalhadas, adicione:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## Ambientes de Produção

### Alta Disponibilidade

Para produção, recomenda-se executar múltiplas instâncias do Eureka Server:

```properties
# Eureka Server 1 (porta 8761)
eureka.client.service-url.defaultZone=http://eureka-server-2:8762/eureka/

# Eureka Server 2 (porta 8762)
eureka.client.service-url.defaultZone=http://eureka-server-1:8761/eureka/
```

### Segurança

Adicione autenticação ao Eureka Dashboard:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

```properties
spring.security.user.name=admin
spring.security.user.password=${EUREKA_PASSWORD}
```

## Troubleshooting

### Serviço não aparece no Dashboard

- Verifique se o microsserviço está rodando
- Confirme a URL do Eureka no application.properties do cliente
- Aguarde até 30 segundos (tempo de registro inicial)
- Verifique logs por erros de conexão

### Self Preservation Mode

Mensagem vermelha no dashboard indicando que o Eureka entrou em modo de auto-preservação:
- É um mecanismo de proteção contra falhas de rede
- O Eureka não remove serviços automaticamente neste modo
- Em desenvolvimento, pode desativar com `eureka.server.enable-self-preservation=false`
- Em produção, mantenha ativado

## Boas Práticas

- **Nomes de Serviços**: Use nomes descritivos e consistentes (kebab-case)
- **Health Checks**: Implemente endpoints `/actuator/health` robustos em cada serviço
- **Timeouts**: Configure timeouts adequados para evitar serviços travados no registro
- **Redundância**: Execute múltiplas instâncias do Eureka em produção
- **Segurança**: Sempre proteja o Eureka Server em produção
- **Monitoramento**: Integre com ferramentas de APM para rastrear descoberta de serviços

## Recursos Adicionais

- [Documentação Oficial Spring Cloud Netflix](https://spring.io/projects/spring-cloud-netflix)
- [Netflix Eureka Wiki](https://github.com/Netflix/eureka/wiki)
- [Padrões de Microsserviços](https://microservices.io/patterns/service-registry.html)

## Próximos Passos

Após configurar o Eureka Server, considere implementar:

1. **API Gateway** - Ponto único de entrada para todos os microsserviços
2. **Config Server** - Configuração centralizada
3. **Circuit Breaker** - Resiliência em comunicação entre serviços
4. **Distributed Tracing** - Rastreamento de requisições entre serviços

## Autores

- **Bytecooks Team**

## Licença

MIT License

---

**Status do Projeto**: Operacional ✅