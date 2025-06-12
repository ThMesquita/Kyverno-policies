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

# Tratar Exceções 🔓
Aqui alguns exemplos de como gerar exceções às suas regras

### Através de Namespace (direto no ClusterPolicy)
```yaml
rules:
  - name: namespace-exception
    match:
      resources:
        kinds:
          - Deployment
        namespaces:
          - "!excluded-namespace"  # Retira namespace da política
```

### Adicionando condição (direto no ClusterPolicy)
```yaml
preconditions:
  all:
    - key: "{{ request.object.metadata.annotations.\"kyverno.io/ignore\" }}"
      operator: NotEquals
      value: "true"
```
A política não será aplicada:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep
  annotations:
    kyverno.io/ignore: "true"  # Política vai ser ignorada
```

### [PolicyException](https://release-1-11-0.kyverno.io/docs/writing-policies/exceptions/)
```yaml
apiVersion: kyverno.io/v2alpha1
kind: PolicyException
metadata:
  name: allow-legacy-apis
spec:
  policies: ["politica-block"]
  rules: ["regra-politica-block"]
  exceptions:
    namespaces: ["prd"]     # Namespace não participará da regra acima 
    resourceNames: ["nginx-pod"]  # Nome dos recursos que não participaram da regra

```

# Exemplos de uso

### Check deprecated APIs
Avisa o usuário sobre APIs descontinuadas que estão sendo usadas
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-dep-api
spec:
  validationFailureAction: Audit
  background: true # Verifica recursos existentes no cluster
  rules:
    - name: deprecated-api
      match:
        resources:
          kinds:
            - batch/*/CronJob
            - discovery.k8s.io/*/EndpointSlice
            - events.k8s.io/*/Event
            - policy/*/PodDisruptionBudget
            - policy/*/PodSecurityPolicy
            - node.k8s.io/*/RuntimeClass
            - networking.k8s.io/*/Ingress
          namespaces:
          - "!excluded-namespace" # Com exceção desse namespace
      preconditions:
        all:
        - key: "{{ request.operation || 'BACKGROUND' }}"
          operator: NotEquals
          value: DELETE
        - key: "{{request.object.apiVersion}}"
          operator: AnyIn
          value:
          - batch/v1beta1
          - discovery.k8s.io/v1beta1
          - events.k8s.io/v1beta1
          - policy/v1beta1
          - node.k8s.io/v1beta1
      validate:
        message: >-
          {{ request.object.apiVersion }}/{{ request.object.kind }} is deprecated and will be removed in v1.25. 
          See: https://kubernetes.io/docs/reference/using-api/deprecation-guide/
    - name: validate-v1-26-removals
      match:
        any:
        - resources:
            kinds:
            - flowcontrol.apiserver.k8s.io/*/FlowSchema
            - flowcontrol.apiserver.k8s.io/*/PriorityLevelConfiguration
            - autoscaling/*/HorizontalPodAutoscaler
      preconditions:
        all:
        - key: "{{ request.operation || 'BACKGROUND' }}"
          operator: NotEquals
          value: DELETE
        - key: "{{request.object.apiVersion}}"
          operator: AnyIn
          value:
          - flowcontrol.apiserver.k8s.io/v1beta1
          - autoscaling/v2beta2
      validate:
        message: >-
          {{ request.object.apiVersion }}/{{ request.object.kind }} is deprecated and will be removed in v1.26.
          See: https://kubernetes.io/docs/reference/using-api/deprecation-guide/
```

### Require Limits and Requests
Alerta que Memória e CPU não está preenchido
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-limits-request
spec:
  validationFailureAction: Audit
  background: true
  rules:
  - name: validate-resources
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Limites de CPU e Memória não definidos"
      pattern:
        spec:
          containers:
            - resources:
                requests:
                  memory: "?*"
                  cpu: "?*"
                limits:
                  memory: "?*"
                  cpu: "?*"
```

### Require Pod Probes
Verifica se os Pods foram preenchidos com Probes
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: probes-policy
spec:
  validationFailureAction: Audit
  background: true
  rules:
  - name: validate-probes
    match:
      any:
      - resources:
          kinds:
            - Pod
    validate:
      message: "Liveness, readiness, ou startup probes precisam ser preenchidos"
      foreach:
      - list: request.object.spec.containers[]
        deny: # Vai negar se os 3 forem verdadeiros
          conditions:
            all:
            - key: livenessProbe
              operator: AllNotIn # Retorna verdadeiro se não encontrar livenessProbe nos elementos do container
              value: "{{ element.keys(@)[] }}"
            - key: startupProbe
              operator: AllNotIn
              value: "{{ element.keys(@)[] }}"
            - key: readinessProbe
              operator: AllNotIn
              value: "{{ element.keys(@)[] }}"
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
