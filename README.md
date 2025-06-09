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
          containers:
          - image: "!*latest" # Bloqueia imagens que terminam com 'latest'
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
