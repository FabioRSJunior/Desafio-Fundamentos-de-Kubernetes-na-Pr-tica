# DESAFIO 2

Implantar, do zero, uma aplicação integrada a um banco de dados em um cluster Kubernetes local, aplicando os principais conceitos da orquestração de contêineres: Pods, Deployments, Services, ConfigMaps, Secrets, Volumes persistentes, Namespaces, probes de saúde e escalonamento. Ao final, você terá uma API que lê e grava dados em um PostgreSQL, com os dados persistentes, configuração externalizada e a aplicação exposta para acesso — tudo declarado em manifests YAML versionáveis. O ponto central é ver as duas camadas conversando entre si dentro do cluster.

> ⚠️ **Aviso**
As credenciais, senhas e Secrets utilizados neste projeto são apenas para fins didáticos e não refletem boas práticas de segurança para ambientes reais. Valores como `admin`/`admin123` estão em texto simples propositalmente, apenas para facilitar a reprodução e o entendimento do funcionamento do Kubernetes. Em um cenário de produção, seriam necessárias medidas adicionais como encryption at rest, gerenciamento externo de segredos (Vault, AWS Secrets Manager, etc.) e políticas de RBAC restritivas.
> 

# Nível 0 -  Como aplicar o projeto

Estrutura do repositório:

Os manifests estão organizados na pasta `manifests/`, com nomes numerados que indicam a ordem de aplicação:

```jsx
manifests/
├── 02-postgres-pvc.yaml
├── 02-postgres-deployment.yaml
├── 02-postgres-service.yaml
├── 03-postgres-secret.yaml
├── 03-postgres-configmap.yaml
├── 03-postgres-deployment.yaml
├── 04-postgrest-deployment.yaml
└── 04-postgrest-service.yaml
```

O prefixo numérico indica o nível do desafio em que cada arquivo foi criado ou atualizado, permitindo aplicar o projeto inteiro na ordem correta.

Ferramenta usada: Minikube (driver Docker)

```jsx
minikube start --driver=docker
kubectl create namespace k8s-desafio
kubectl apply -f manifests/02-postgres-pvc.yaml
kubectl apply -f manifests/02-postgres-deployment.yaml
kubectl apply -f manifests/02-postgres-service.yaml
kubectl apply -f manifests/03-postgres-secret.yaml
kubectl apply -f manifests/03-postgres-configmap.yaml
kubectl apply -f manifests/03-postgres-deployment.yaml
kubectl apply -f manifests/04-postgrest-deployment.yaml
kubectl apply -f manifests/04-postgrest-service.yaml
```

Verificação final:

```jsx
kubectl get all -n k8s-desafio
```

![image.png](images/image.png)

# Nível 1 - Namespace e primeiro contato

Organizar os recursos e subir um contêiner avulso; Crie um Namespace próprio para o desafio e suba um Pod avulso de teste (pode ser qualquer imagem simples). Inspecione o Pod: seus detalhes, eventos e logs. Depois, delete-o. Namespace Pod kubectl get / describe / logs

criando um nomespace: 

```jsx
kubectl create namespace k8s-desafio
```

Confirmando que o namespace existe: 

```jsx
# 1. Confirmar que o namespace existe (evidência)
kubectl get namespace k8s-desafio
```

![image.png](images/image%201.png)

```jsx
# 2. Subir um Pod avulso de teste
kubectl run pod-teste --image=nginx -n k8s-desafio
```

![image.png](images/image%202.png)

```jsx
# 3. Inspecionar o Pod
kubectl get pods -n k8s-desafio
```

![image.png](images/image%203.png)

```jsx
# 3.1 Inspecionar o Pod
kubectl describe pod pod-teste -n k8s-desafio
```

![image.png](images/image%204.png)

```jsx
# 3.2 Inspecionar o Pod
kubectl logs pod-teste -n k8s-desafio
```

![image.png](images/image%205.png)

```jsx
# 4. Deletar o Pod
kubectl delete pod pod-teste -n k8s-desafio
```

![image.png](images/image%206.png)

```jsx
# 5. Confirmar que ele NÃO voltou sozinho
kubectl get pods -n k8s-desafio
```

Primeiro criamos um Namespace próprio (`k8s-desafio`) para isolar todos os recursos do desafio, e confirmamos sua existência com `kubectl get namespace`. Em seguida, subimos um Pod avulso de teste usando a imagem simples do Nginx, sem nenhum Deployment ou controller por trás dele. Inspecionamos esse Pod de três formas: `kubectl get pods` para confirmar que estava com status `Running`; `kubectl describe pod` para ver detalhes como IP interno, imagem usada e o histórico de eventos (agendamento, pull da imagem, criação e início do container); e `kubectl logs` para conferir a saída de inicialização da aplicação, mostrando o Nginx subindo seus worker processes sem erros. Por fim, deletamos o Pod com `kubectl delete pod` e rodamos `kubectl get pods` novamente para confirmar que ele não foi recriado, esperando que ele simplesmente desaparecesse da lista.

Reflita: ao deletar esse Pod avulso, ele volta sozinho? Por quê? Isso te diz algo sobre por
que raramente criamos Pods diretamente.

**Não, o Pod avulso não voltou sozinho. Isso acontece porque um Pod criado diretamente com `kubectl run` não tem nenhum controller (Deployment, ReplicaSet, StatefulSet) vigiando o seu estado. Ele existe de forma isolada, sem nada que o reconcilie caso seja removido.**

**Diferente disso, os Pods do Postgres e do PostgREST (criados via Deployment) são gerenciados por um ReplicaSet que garante continuamente que o número de réplicas desejado esteja rodando. Se um desses Pods for deletado, o Kubernetes recria automaticamente para manter o estado declarado.**

**É exatamente por isso que raramente criamos Pods "nus" em produção: eles não se recuperam de falhas, reinícios de node ou remoções acidentais. Usar Deployments (ou outros controllers) é o que torna a aplicação resiliente e alinhada ao modelo declarativo do Kubernetes.**

