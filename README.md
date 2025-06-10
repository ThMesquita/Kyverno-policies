# Políticas Kyverno

Kyverno é uma ferramenta que permite validar, modificar e gerar recursos com os arquivos YAML.
O uso de políticas no Kyverno é essencial para impedir a criação de regras que possam comprometer a segurança ou a integridade do seu código.

## Requisitos 🧰
- Ter o [kubectl](https://kubernetes.io/docs/reference/kubectl/) instalado e conectado a um cluster  
- Ter o [Helm](https://helm.sh/docs/intro/install/) instalado  
- [kubectx](https://github.com/ahmetb/kubectx) (opcional)

## Como instalar o Kyverno 🏗️
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```



# Validação Kyverno
Com um simples código YAML, podemos definir uma política de validação:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-latest-version
spec:
  validationFailureAction: Enforce # A política Enforce obriga o usuário a cumprir a regra, enquanto a política Audit apenas gera um aviso informativo. 
  rules:
  - name: validate-version
    match:
      resources:
        kinds:
        - Deployment
        - StatefulSet
        - DaemonSet
    validate:
      message: "NÃO usar LATEST na versão, para evitar bugs futuros"
      pattern:
        spec:
          template:
            spec:
              containers:
              - image: "!*:latest" # Bloqueia imagens que terminam com 'latest'
```

Aqui temos um código simples que será implementado para o teste de validação:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-latest
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-latest
  template:
    metadata:
      labels:
        app: nginx-latest
    spec:
      containers:
        - name: nginx
          image: nginx:latest # Imagem proibida anteriormente na nossa política
```
Resultado após o kubectl apply -f nginx-latest.yaml:
```bash
Error from server: error when creating "nginx-latest.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Deployment/default/nginx-latest was blocked due to the following policies

block-latest-version:
  validate-version: 'validation error: NÃO usar LATEST na versão, para evitar bugs
    futuros. rule validate-version failed at path /spec/containers/'
```



# Mutação Kyverno
A mutação no Kyverno possibilita a edição de conteúdo de algo criado
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: label-require
spec:
  validationFailureAction: Enforce
  rules:
  - name: add-stg-label
    match:
      resources:
        kinds:
        - Deployment
    mutate:  # Mutação
      patchStrategicMerge:
        metadata:
          labels:
            team: stg # adiciona label no Deployment
        spec:
          selector:
            matchLabels:
              team: stg # adiciona label de conexão entre pod e deployment
          template:
            metadata:
              labels:
                team: stg # adiciona label no pod do deployment 
```
Exemplo de como ficaria um deployment com a mutação:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-mutacao
  # a policy vai adicionar: team: stg
spec:
  replicas: 1
  selector:
    matchLabels:
      # a policy vai adicionar: team: stg
  template:
    metadata:
      labels:
        # a policy vai adicionar: team: stg
    spec:
      containers:
        - image: nginx:1.25
          name: nginx
```



# Geração Kyverno
A geração no Kyverno possibilita gerar novos recursos quando algo é criado
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: hpa-generate
spec:
  rules:
  - name: generate-hpa
    match: # Onde a política é aplicada
      resources:
        kinds:
          - Deployment
    preconditions: # A regra só é acionada se o Deployment tiver a label hpa_enabled: "true"
      all:
      - key: "{{ request.object.metadata.labels.hpa_enabled }}"
        operator: Equals
        value: "true"
    generate: # Criação do HPA
      kind: HorizontalPodAutoscaler
      apiVersion: autoscaling/v2
      name: "{{request.object.metadata.name}}-hpa"
      namespace: "{{request.object.metadata.namespace}}"
      synchronize: true # Atualiza o HPA caso o deployment seja modificado
      data:
        spec:
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: "{{request.object.metadata.name}}"
          minReplicas: 1
          maxReplicas: 5
          metrics: # Escala com base na CPU (alvo de 60% de utilização)
          - type: Resource
            resource:
              name: cpu
              target:
                type: Utilization
                averageUtilization: 60
```
Código para ativar a política para criar o HPA
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hpa
  labels:
    hpa_enabled: "true"  # Ativa a política
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```
### HPA gerado:
```bash
NAME            REFERENCE              TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa-hpa   Deployment/nginx-hpa   cpu: <unknown>/60%   1         5         0          4s
```


# Vantagens de usar Kyverno ✅
  ## Validação:
  -  Definir uma tag padrão ou até mesmo proibir o latest como visto anteriormente
  -  Evitar containers rodando como root
  -  Bloquear uso de portas não autorizadas
  - Garantir Labels
  - Obrigar convenção de nomes (dev-*, prd-*)
  - Definir limites de CPU/memória
  - Impedir que usuários usem outros namespaces

## Mutação:
  - Adicionar labels automaticamente
  - Renomear ou adicionar prefixos nos recursos criados (dev-)
  - Injetar PodDisruptionBudget (evitar que todos os pods sejam reiniciados de uma só vez)
  - Evita erros por omissão (storageClassName no PVC)
  - Aplicar mudanças em geral

## Geração
  - Gerar recursos de forma automática como:
      - HPA
      - ConfigMap
      - Secret
      - ResourceQuota
      - LimitRange
      - Service
      - Ingress
      - PVC

# Desvantagens ❌
- Performance ruim em clusters com muitas políticas e recursos
- Limitações que com a linguagem Go você não teria
- Pode ter inconsistência com a política de Geração caso recursos sejam apagados manualmente
- Mensagens de erro às vezes não são tão claras
