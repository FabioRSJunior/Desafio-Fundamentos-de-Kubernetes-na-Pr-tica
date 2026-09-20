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

# Nível 2 — PostgreSQL com persistência

Subir o PostgreSQL de forma que os dados sobrevivam. Implante o PostgreSQL no cluster. Ele precisa de armazenamento que não desapareça quando o Pod for recriado — pesquise como reservar armazenamento e montá-lo no diretório de dados do Postgres. Também precisa de um Service para que outros recursos consigam encontrá-lo pelo nome.

Para este nível, foram criados três manifests YAML: um PersistentVolumeClaim para garantir que os dados do banco não se percam quando o Pod for recriado, um Deployment do PostgreSQL configurado com usuário e senha diretamente no YAML (etapa que será substituída por Secret no Nível 3) e um Service do tipo ClusterIP, que expõe o banco apenas dentro do cluster, permitindo que outros recursos o encontrem pelo nome em vez de por IP. No Deployment, vale destacar o uso do `subPath: postgres` no volumeMount, que evita um problema comum onde o Postgres reclama por já existir uma pasta `lost+found` na raiz do volume montado.

1. PersistentVolumeClaim (`02-postgres-pvc.yaml`)

```jsx
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: k8s-desafio
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

2. Deployment do Postgres (`02-postgres-deployment.yaml`)

```jsx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: k8s-desafio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              value: "admin"
            - name: POSTGRES_PASSWORD
              value: "admin123"
            - name: POSTGRES_DB
              value: "desafiodb"
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
              subPath: postgres
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

3. Service (`02-postgres-service.yaml`)

```jsx
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: k8s-desafio
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Depois dos artefatos criados, precisamos aplicação dos três manifests:

```jsx
kubectl apply -f 02-postgres-pvc.yaml
kubectl apply -f 02-postgres-deployment.yaml
kubectl apply -f 02-postgres-service.yaml
```

Verificar

```jsx
# confirma se o namespace existe 
kubectl get namespace k8s-desafio
```

![image.png](images/image%207.png)

```jsx
# Ver se o PVC foi criado e está "Bound" (vinculado a um volume):
kubectl get pvc -n k8s-desafio
```

![image.png](images/image%208.png)

```jsx
# Ver se o Pod do Postgres está rodando:
kubectl get pods -n k8s-desafio
```

![image.png](images/image%209.png)

tudo certo, está running , 5. Teste real de conexão (o mais importante — confirma que o Postgres está de pé e aceitando conexões). Entre no próprio pod e rode um comando psql

```jsx
kubectl exec -it -n k8s-desafio deploy/postgres -- psql -U admin -d desafiodb -c "SELECT 1;"
```

![image.png](images/image%2010.png)

Resultado

Namespace ativo, PVC `Bound`, Pod do Postgres em `Running`, e o comando `SELECT 1;` retornou com sucesso, confirmando que o banco estava de pé e aceitando conexões.

Reflita: qual a diferença entre montar um PVC e um emptyDir ? O que aconteceria com os
dados em cada caso ao deletar o Pod? (Você vai provar isso no Nível 5.)

**Um `emptyDir` é um volume que existe apenas enquanto o Pod estiver vivo naquele node: ele é criado junto com o Pod e apagado junto com ele. Se o Pod for deletado ou recriado, todos os dados armazenados nesse volume se perdem, já que o `emptyDir` não é um recurso independente do Pod ele é apenas um espaço temporário no disco do node atrelado ao ciclo de vida daquele Pod específico.. Já um PVC (PersistentVolumeClaim) é um recurso independente do Pod. Ele existe por conta própria no cluster e é apenas *montado* pelo Pod, não pertence a ele. Isso significa que, quando o Pod é deletado e um novo é recriado pelo Deployment, o novo Pod pode montar o mesmo PVC e continuar de onde os dados pararam — o armazenamento sobrevive independentemente do ciclo de vida do Pod. Na prática: se o Postgres estivesse usando um `emptyDir` e o Pod fosse deletado, todos os dados do banco seriam perdidos ao recriar o Pod. Como usamos um PVC, o mesmo dado inserido antes da recriação continua acessível depois.**

# Nível 3 — Configuração e segredos

Externalizar config e proteger as credenciais do banco, As credenciais do PostgreSQL (usuário e senha) não podem estar escritas dentro do YAML do Deployment. Mova-as para um Secret e injete no container do banco. Coloque também alguma configuração não sensível em um ConfigMap. Esse mesmo Secret será reutilizado pela API no próximo nível.

Para este nível, as credenciais do Postgres (usuário e senha) foram removidas do Deployment e movidas para um Secret, enquanto a configuração não sensível (o nome do banco) foi movida para um ConfigMap. O Deployment foi então atualizado para puxar essas variáveis via `valueFrom` (`secretKeyRef` e `configMapKeyRef`) em vez de valores fixos no YAML. Como o Kubernetes exige que os dados de um Secret estejam codificados em base64, os valores de usuário e senha foram convertidos antes de montar o arquivo.

```jsx
#Gerando os valores em base64:

echo -n "admin" | base64
echo -n "admin123" | base64

YWRtaW4=
YWRtaW4xMjM=
```

1. Secret (`03-postgres-secret.yaml`)

```jsx
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: k8s-desafio
type: Opaque
data:
  POSTGRES_USER: YWRtaW4=
  POSTGRES_PASSWORD: YWRtaW4xMjM=
```

2. ConfigMap (`03-postgres-configmap.yaml`)

```jsx
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: k8s-desafio
data:
  POSTGRES_DB: "desafiodb"
```

3. Deployment atualizado (`03-postgres-deployment.yaml`)

```jsx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: k8s-desafio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: POSTGRES_DB
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
              subPath: postgres
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

Aplicação dos três manifests:

```jsx
kubectl apply -f 03-postgres-secret.yaml
kubectl apply -f 03-postgres-configmap.yaml
kubectl apply -f 03-postgres-deployment.yaml
```

Verificação de que o Pod foi recriado e de que as variáveis de ambiente vieram corretamente do Secret e do ConfigMap:

```jsx
kubectl get pods -n k8s-desafio
kubectl exec -it -n k8s-desafio deploy/postgres -- env | grep POSTGRES
```

![image.png](images/image%2011.png)

Resultado

O comando `env` confirmou os valores corretos de `POSTGRES_USER`, `POSTGRES_PASSWORD` e `POSTGRES_DB`, comprovando que vieram do Secret e do ConfigMap em vez de estarem hardcoded no Deployment.

Um ponto de atenção observado: como a mudança de `value` fixo para `valueFrom` altera a especificação do Deployment, o Kubernetes recriou o Pod do Postgres automaticamente. Como o volume é persistente (PVC), os dados que já estavam lá continuaram intactos. Vale notar que, se usuário e senha fossem alterados para valores diferentes dos originais nessa troca, o Postgres não recriaria o usuário automaticamente — a inicialização do usuário só roda na primeira vez que o volume é criado. Como neste caso os valores permaneceram os mesmos (`admin`/`admin123`), não houve problema.

Reflita: ao inspecionar o Secret com -o yaml , o valor aparece "embaralhado". Isso é
criptografia de verdade ou apenas codificação? O que isso significa para a segurança real?

```jsx
kubectl get secret postgres-secret -n k8s-desafio -o yaml
```

![image.png](images/image%2012.png)

```jsx
echo "Usuário : $(echo "YWRtaW4=" | base64 -d)"
echo "Senha   : $(echo "YWRtaW4xMjM=" | base64 -d)"
```

![image.png](images/image%2013.png)

**Não é criptografia de verdade, é apenas codificação em base64. A diferença é fundamental: criptografia exige uma chave secreta para reverter o processo (sem a chave, o dado original é irrecuperável), enquanto base64 é uma codificação reversível por qualquer pessoa, sem senha nenhuma — como mostrado acima, basta rodar `base64 -d` para obter o valor original de volta.**

**Isso significa que um Secret do Kubernetes, por padrão, não protege as credenciais de quem tem acesso ao cluster. Qualquer pessoa com permissão de `get` sobre Secrets naquele namespace consegue ler as credenciais em texto puro com um comando trivial. Na prática, Secrets servem principalmente para *separar* dados sensíveis da definição da aplicação (evitando hardcode no YAML do Deployment) e para controlar o acesso via RBAC — mas não substituem uma solução real de segurança. Para proteção de verdade, seria necessário habilitar encryption at rest no etcd (armazenamento do cluster), usar uma ferramenta externa de gerenciamento de segredos (como HashiCorp Vault ou AWS Secrets Manager), e restringir rigorosamente via RBAC quem pode ler Secrets no namespace.**

