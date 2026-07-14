# 02 — Kubernetes: conceitos essenciais para dev .NET

## Por que Kubernetes aparece em vagas pleno/sênior

Kubernetes deixou de ser "coisa de DevOps" e virou expectativa para dev backend pleno/sênior. Você não precisa administrar um cluster, mas precisa saber:
1. Empacotar sua aplicação para rodar em K8s (Deployment, Service, ConfigMap)
2. Entender o ciclo de vida de um pod (liveness/readiness probes)
3. Debuggar problemas básicos (`kubectl logs`, `kubectl describe`)
4. Conversar com o time de infraestrutura sem parecer perdido

---

## Conceitos fundamentais (traduzidos para .NET)

### Pod: a menor unidade

Um pod é um grupo de 1+ containers que compartilham rede e storage. No caso .NET, 1 pod = 1 container da API.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eventhub-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: eventhub-api
  template:
    metadata:
      labels:
        app: eventhub-api
    spec:
      containers:
        - name: api
          image: eventhub-api:latest
          ports:
            - containerPort: 8080
          env:
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef:
                  name: eventhub-secrets
                  key: connection-string
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

### Service: expondo pods

```yaml
apiVersion: v1
kind: Service
metadata:
  name: eventhub-api
spec:
  selector:
    app: eventhub-api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP # interno ao cluster (padrão)
  # type: LoadBalancer # expõe externamente (cloud)
```

Um Service fornece um IP estável e DNS interno (`eventhub-api.default.svc.cluster.local`) que balanceia entre os pods do Deployment. Se um pod morre e outro nasce, o Service continua roteando sem mudança de configuração.

### ConfigMap + Secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eventhub-config
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Logging__LogLevel__Default: "Warning"

---
apiVersion: v1
kind: Secret
metadata:
  name: eventhub-secrets
type: Opaque
stringData:
  connection-string: "Host=postgres;Database=eventhub;Username=postgres;Password=SuperSecret123"
```

ConfigMap para configuração não-sensível. Secret para senhas, connection strings, chaves de API. Em produção, use **Sealed Secrets** ou **External Secrets Operator** (integra com Azure Key Vault, AWS Secrets Manager).

### Probes: mantendo a aplicação saudável

| Probe | O que faz | Exemplo .NET |
|---|---|---|
| **Liveness** | "O pod deve ser reiniciado?" | `/health` retorna 200 → vivo. 500 → K8s reinicia. |
| **Readiness** | "O pod pode receber tráfego?" | `/health` retorna 200 com banco OK → pronto. Banco fora → 503, K8s remove do Service. |
| **Startup** | "O pod já terminou de iniciar?" | Útil para apps com cold start lento. Delay maior antes de começar liveness. |

O ASP.NET Core tem health checks nativos que se integram com probes:

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("Default")!);

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

---

## Minikube/Kind: K8s local para desenvolvimento

```bash
# Minikube
minikube start
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl port-forward svc/eventhub-api 8080:80
curl http://localhost:8080/health

# Kind (Kubernetes in Docker — mais leve, roda em qualquer lugar)
kind create cluster
kubectl apply -f deployment.yaml
```

---

## Trade-offs: quando NÃO usar Kubernetes

- **Aplicação monolítica com 1-2 réplicas**: Docker Compose ou Azure App Service resolvem com menos complexidade operacional
- **Equipe sem experiência em K8s**: o custo de aprendizado e manutenção supera os benefícios
- **Orçamento apertado**: clusters gerenciados (AKS, EKS, GKE) têm custo fixo de ~$70/mês só pelo control plane

**Regra prática**: se você está na Fase 5-6 do roadmap e o projeto é de portfólio, ter um `deployment.yaml` funcional e entender os conceitos já é suficiente. O aprofundamento em operação de cluster fica para o dia a dia do trabalho.

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Explicar a diferença entre Deployment (pods), Service (rede) e ConfigMap (config)
- [ ] Configurar liveness e readiness probes apontando para `/health`
- [ ] Diferenciar `requests` (mínimo garantido) de `limits` (máximo permitido)
- [ ] Usar `kubectl logs`, `kubectl describe` e `kubectl port-forward` para debug
- [ ] Saber que Secret em YAML puro não é seguro para produção (use Sealed Secrets ou External Secrets)
