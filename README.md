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


![image.png](images/image40.png)


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

# Nível 4 — A API conectada ao banco (a integração)

Subir o PostgREST apontando para o PostgreSQL, Implante a API (PostgREST) como um Deployment. Ela se configura por variáveis de ambiente — precisa da string de conexão com o banco, que deve apontar para o nome do Service do PostgreSQL (não um IP). Reaproveite o usuário e a senha do Secret do nível anterior. Crie uma tabela no banco e confirme que a API expõe essa tabela via HTTP.

Primeiro foi criada uma tabela de exemplo no Postgres, já que o PostgREST expõe automaticamente qualquer tabela do banco como endpoint REST. Em seguida, foi criado o Deployment do PostgREST, configurado via variável de ambiente `PGRST_DB_URI` (a string de conexão), montada combinando as variáveis já existentes do Secret e do ConfigMap através da sintaxe `$(VAR)` do Kubernetes, o que evita repetir usuário e senha em texto puro. Essa string aponta para `postgres-service`, o nome do Service do Postgres, permitindo que o PostgREST encontre o banco via DNS interno do cluster, sem depender de IP. Por fim, foi criado um Service para o PostgREST, que já deixa a API pronta para ser exposta no Nível 5.

**Criando a tabela de exemplo:**

```jsx
kubectl exec -it -n k8s-desafio deploy/postgres -- psql -U admin -d desafiodb
```

Dentro do `psql:`

```jsx
CREATE TABLE todos (
  id SERIAL PRIMARY KEY,
  tarefa TEXT NOT NULL,
  feito BOOLEAN DEFAULT false
);
GRANT ALL ON todos TO admin;
GRANT ALL ON SEQUENCE todos_id_seq TO admin;
\q
```

![image.png](images/image%2014.png)

1. Deployment do PostgREST (`04-postgrest-deployment.yaml`)

```jsx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgrest
  namespace: k8s-desafio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgrest
  template:
    metadata:
      labels:
        app: postgrest
    spec:
      containers:
        - name: postgrest
          image: postgrest/postgrest:latest
          ports:
            - containerPort: 3000
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
            - name: PGRST_DB_URI
              value: "postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres-service:5432/$(POSTGRES_DB)"
            - name: PGRST_DB_SCHEMA
              value: "public"
            - name: PGRST_DB_ANON_ROLE
              value: "admin"
```

Dois pontos importantes nesse Deployment: `postgres-service` no `PGRST_DB_URI` é o nome do Service do Postgres, é assim que o PostgREST encontra o banco pelo DNS interno do cluster; e `PGRST_DB_ANON_ROLE` precisa ser um role que já tenha permissões no banco como foi dado `GRANT ALL` para `admin`, esse role foi reutilizado aqui (em produção o ideal seria criar um role dedicado, só de leitura/gravação restrita).

**2. Service do PostgREST (`04-postgrest-service.yaml`)**

```jsx
apiVersion: v1
kind: Service
metadata:
  name: postgrest-service
  namespace: k8s-desafio
spec:
  selector:
    app: postgrest
  ports:
    - port: 80
      targetPort: 3000
```

Aplicação dos manifests:

```jsx
kubectl apply -f 04-postgrest-deployment.yaml
kubectl apply -f 04-postgrest-service.yaml
```

Verificação de que a API subiu corretamente:

```jsx
**kubectl get pods -n k8s-desafio
kubectl logs -n k8s-desafio -l app=postgrest**
```

![image.png](images/image%2015.png)

Teste da integração, ainda de dentro do cluster

```jsx
kubectl run curl-test --image=curlimages/curl -it --rm --restart=Never -n k8s-desafio -- curl http://postgrest-service/todos
```

**Testar a integração**

```bash
kubectl run curl-test --image=curlimages/curl -it --rm --restart=Never -n k8s-desafio -- curl http://postgrest-service/todos
```

Isso deve retornar `[]` (array vazio, já que a tabela `todos` ainda não tem dados) — se retornar isso, prova que a API achou o Postgres e leu a tabela com sucesso.

![image.png](images/image%2016.png)

Resultado

 Os logs mostraram `Listening on port 3000`, confirmando que a API estava de pé e conectada ao banco. O teste de integração retornou `[]` (array vazio, já que a tabela `todos` ainda não tinha dados), o que prova três coisas ao mesmo tempo: a API PostgREST está no ar, ela conseguiu se conectar ao Postgres pelo nome do Service (DNS interno do cluster), e a tabela `todos` existe e está acessível.

Reflita: por que usamos o nome do Service do Postgres na string de conexão, em vez do IP
do Pod? O que aconteceria com a conexão se você usasse o IP e o Pod do banco fosse
recriado?

Usamos o nome do Service porque ele é um endereço estável, resolvido via DNS interno do cluster, enquanto o IP de um Pod é dinâmico e pode mudar sempre que ele é recriado (por crash, atualização ou remoção manual). Se a conexão apontasse diretamente para o IP do Pod, ela funcionaria até o Postgres ser recriado com um novo IP, e a partir daí o PostgREST continuaria tentando acessar o endereço antigo e falharia; o Service evita isso ao atuar como uma camada estável que redireciona o tráfego automaticamente para o Pod correto, não importa quantas vezes seu IP interno mude.

# Nível 5 — Expor a API e provar a persistência (o coração do desafio)

Acessar a API de fora e garantir que os dados não somem Exponha a API para você conseguir acessá-la da sua máquina. Faça uma requisição que insira um dado através da API e outra que leia esse dado de volta. Em seguida, delete o Pod do PostgreSQL, espere o cluster recriá-lo, e consulte a API novamente.

O critério de sucesso deste nível: o dado inserido antes de deletar o Pod do banco
ainda deve estar acessível pela API depois da recriação. Se sumiu, seu volume não
está persistindo — revise o Nível 2.

Para expor a API, foi usado `kubectl port-forward` diretamente no Service, redirecionando a porta local 8080 para a porta 80 do `postgrest-service`. Com o `port-forward` ativo em um terminal, um dado foi inserido via `POST` e lido de volta via `GET` em outro terminal, confirmando que a API estava acessível de fora do cluster. Em seguida veio o teste central do desafio: o Pod do Postgres foi deletado manualmente, e como ele é gerenciado por um Deployment, o Kubernetes o recriou automaticamente. Depois da recriação, a API foi consultada novamente e o dado inserido anteriormente continuava lá, comprovando que o PVC realmente persiste os dados independente do ciclo de vida do Pod.

```jsx
kubectl port-forward -n k8s-desafio svc/postgrest-service 8080:80
```

Confirmando que está exposta e funcionando:

```jsx
curl http://localhost:8080/todos
```

![image.png](images/image%2017.png)

**1. Inserindo um dado via POST:**

```jsx
curl -X POST http://localhost:8080/todos \
  -H "Content-Type: application/json" \
  -d '{"tarefa": "Aprender Kubernetes", "feito": false}'
```

![image.png](images/image%2018.png)

2. Lendo o dado de volta via GET:

```jsx
curl http://localhost:8080/todos
```

![image.png](images/image%2019.png)

3. Provando a persistência, deletando o Pod do Postgres:

```jsx
kubectl delete pod -n k8s-desafio -l app=postgres
```

![image.png](images/image%2020.png)

**Aguardando o Kubernetes recriar o Pod automaticamente:**

```jsx
kubectl get pods -n k8s-desafio -w
```

![image.png](images/image%2021.png)

Consultando a API novamente após a recriação:

```jsx
curl http://localhost:8080/todos
```

Se o dado que você inseriu no passo 1 **ainda estiver lá**, você provou que:

- O volume (PVC) realmente persiste os dados, independente do Pod
- Toda a cadeia (PVC → Deployment → Service → API) está coordenada corretamente

![image.png](images/image%2022.png)

Resultado:  O dado inserido no passo 1 (`"Aprender Kubernetes"`) continuou acessível pela API mesmo depois da recriação do Pod do Postgres, com o `id` gerado automaticamente preservado. Isso comprova que o volume (PVC) realmente persiste os dados de forma independente do Pod, e que toda a cadeia PVC → Deployment → Service → API está coordenada corretamente. Isso fecha o Nível 5.

Reflita: qual a diferença prática entre liveness e readiness? Por que escalar a API para várias
réplicas é seguro, mas escalar o banco desse jeito (com o mesmo PVC) não seria?

A liveness probe detecta se o container travou e precisa ser reiniciado, enquanto a readiness probe detecta se o container está pronto para receber tráfego, removendo-o temporariamente dos Endpoints do Service caso não esteja, sem reiniciá-lo; escalar o PostgREST é seguro porque a API é stateless (todas as réplicas apenas consultam o mesmo Postgres, sem guardar dados próprios), mas escalar o Postgres com o mesmo PVC seria perigoso porque o banco é stateful e o volume (`ReadWriteOnce`) só pode ser escrito por um Pod por vez, então múltiplas réplicas escrevendo no mesmo diretório de dados corromperiam o banco.

# Nível 6 — Health Checks e escala

Ensinar o cluster a monitorar e escalar a API Adicione liveness e readiness probes à API, para que o Kubernetes saiba quando reiniciá-la e quando ela está pronta para receber tráfego. Defina também requests e limits de CPU e memória. Aumente o número de réplicas da API e observe o Service balancear a carga entre elas.

**Ver as 3 réplicas rodando:**

```bash
kubectl get pods -n k8s-desafio -l app=postgrest
```

Espera-se 3 Pods, todos `1/1 Running`.

![image.png](images/image%2023.png)

**Confirmar que as probes estão configuradas:**

```bash
kubectl describe pod -n k8s-desafio -l app=postgrest | grep -A 3 "Liveness\|Readiness"
```

![image.png](images/image%2024.png)

**Confirmar os requests/limits:**

```bash
kubectl describe pod -n k8s-desafio -l app=postgrest | grep -A 4 "Limits\|Requests"
```

**Testar se a API continua respondendo (com o `port-forward` ainda ativo):**

```jsx
kubectl port-forward -n k8s-desafio svc/postgrest-service 8080:80
curl http://localhost:8080/todos
```

![image.png](images/image%2025.png)

Confirme que o `port-forward` está ativo em um terminal separado:

```bash
kubectl port-forward -n k8s-desafio svc/postgrest-service 8080:80
```

No outro terminal, dispare várias requisições seguidas

```bash
for i in {1..10}; do curl -s http://localhost:8080/todos > /dev/null && echo "req$i ok"; done
```

![image.png](images/image%2026.png)

Para *ver de fato* o balanceamento acontecendo, olhe os logs de cada um dos 3 Pods depois de rodar o teste acima:

```bash
kubectl get pods -n k8s-desafio -l app=postgrest
```

Pegue os 3 nomes que aparecerem e rode para cada um:

![image.png](images/image%2027.png)

```bash
kubectl logs -n k8s-desafio <nome-do-pod-1>
kubectl logs -n k8s-desafio <nome-do-pod-2>
kubectl logs -n k8s-desafio <nome-do-pod-3>
```

Se as requisições aparecerem espalhadas entre os 3 Pods (não todas concentradas em um só), isso comprova visualmente que o Service está balanceando a carga. Os 3 Pods estão rodando normalmente. Agora rode os `logs` com os nomes reais:

```bash
kubectl logs -n k8s-desafio postgrest-8699cfb64d-7n42h
kubectl logs -n k8s-desafio postgrest-8699cfb64d-pvgnc
kubectl logs -n k8s-desafio postgrest-8699cfb64d-sct66
```

gere um pouco de tráfego primeiro (com o `port-forward` ativo em outro terminal):

![image.png](images/image%2028.png)

depois dos testes de logs 

![image.png](images/image%2029.png)

As 15 requisições passaram (`req 1 ok` até `req 15 ok`) e o `port-forward` mostrou `Handling connection for 8080` 15 vezes, confirmando que o tráfego chegou até a API.

Isso já prova que o Service está roteando as requisições normalmente. Como o kube-proxy faz round-robin por padrão entre os Endpoints (os 3 Pods do PostgREST), o balanceamento está acontecendo mesmo que os logs individuais do PostgREST não mostrem cada request (ele só loga eventos de sistema, não cada chamada HTTP). Para fechar a evidência do balanceamento de forma mais concreta:

```bash
kubectl get endpoints -n k8s-desafio postgrest-service
```

![image.png](images/image%2030.png)

Listar os 3 IPs dos Pods por trás do Service, essa é a prova de que existem 3 destinos possíveis recebendo o tráfego balanceado.

qual a diferença prática entre liveness e readiness? Por que escalar a API para várias
réplicas é seguro, mas escalar o banco desse jeito (com o mesmo PVC) não seria?

A liveness probe detecta se o container travou e precisa ser reiniciado, enquanto a readiness probe detecta se o container está pronto para receber tráfego, removendo-o temporariamente dos Endpoints do Service caso não esteja, sem reiniciá-lo; escalar o PostgREST é seguro porque a API é stateless (todas as réplicas apenas consultam o mesmo Postgres, sem guardar dados próprios), mas escalar o Postgres com o mesmo PVC seria perigoso porque o banco é stateful e o volume (`ReadWriteOnce`) só pode ser escrito por um Pod por vez, então múltiplas réplicas escrevendo no mesmo diretório de dados corromperiam o banco.

# Nível 7 — Escalonamento automático (bônus)

Deixar a API escalar sozinha sob carga, Configure um Horizontal Pod Autoscaler (HPA) para a API, escalando conforme o uso de CPU. Gere carga com uma ferramenta de sua escolha e observe o cluster criar novos Pods automaticamente — e removê-los quando a carga cair.

**Nível 7 — Horizontal Pod Autoscaler (HPA)**. O objetivo é fazer a API escalar sozinha conforme o uso de CPU sobe.

**1. Habilitar o metrics-server no minikube**

O HPA depende dele para saber o uso de CPU dos Pods. No minikube, é um addon:

bash

```bash
minikube addons enable metrics-server
```

![image.png](images/image%2031.png)

Confirme que ele subiu:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

![image.png](images/image%2032.png)

Espere até aparecer `Running`. 

```bash
kubectl top pods -n k8s-desafio
```

![image.png](images/image%2033.png)

```bash
kubectl autoscale deployment postgrest -n k8s-desafio --cpu-percent=50 --min=1 --max=5
```

![image.png](images/image%2034.png)

Confirme que foi criado e que já está lendo métricas:

```bash
kubectl get hpa -n k8s-desafio
```

![image.png](images/image%2035.png)

**Importante:** o HPA calcula a porcentagem em cima do `requests.cpu` que você já definiu no Deployment do Nível 6. Se esse valor estiver alto (ex: `100m` ou mais), pode ser difícil gerar carga suficiente para estourar 50%. Confira com:

```bash
kubectl describe pod -n k8s-desafio -l app=postgrest | grep -A 2 Requests
```

![image.png](images/image%2036.png)

O problema mais comum aqui é que um loop simples de `curl` de fora do cluster não gera carga suficiente, porque cada requisição HTTP é rápida demais e o gargalo é a rede, não a CPU do Pod. O jeito mais confiável é rodar um Pod **dentro do cluster**, batendo direto no Service em loop apertado, sem pausa:

```bash
kubectl run carga -n k8s-desafio --image=busybox --restart=Never -- sh -c '
for i in $(seq 1 20); do
  (while true; do wget -q -O- http://postgrest-service/todos; done) &
done
wait'
```

Deixe rodando. Se depois de 1-2 minutos o CPU não subir o suficiente, suba a intensidade rodando esse mesmo comando em 2-3 terminais diferentes ao mesmo tempo, multiplicando a carga.Observar o escalonamento em tempo real

```bash
kubectl get hpa -n k8s-desafio -w
```

![image.png](images/image%2037.png)

Você deve ver o `TARGETS` subir (ex: `65%/50%`) e o número em `REPLICAS` aumentar sozinho, de 1 para 2, 3, etc. Em paralelo, confira os Pods sendo criados:

```bash
kubectl get pods -n k8s-desafio -l app=postgrest -w
```

Ver ele escalar de volta

Pare o Pod de carga (`Ctrl+C`, ele se autodeleta por causa do `--rm`). O HPA tem um período de estabilização (padrão 5 minutos) antes de reduzir réplicas, para evitar oscilação. Espere alguns minutos e rode de novo:

bash

```bash
kubectl get hpa -n k8s-desafio -w
```

![image.png](images/image%2038.png)

Você deve ver `REPLICAS` voltar gradualmente para `1`.

![image.png](images/image%2039.png)

kubectl get all -n k8s-desafio

Com o HPA configurado e uma carga concorrente suficiente gerada (múltiplos loops de requisições simultâneas dentro de um Pod), o uso de CPU do PostgREST ultrapassou amplamente o limiar configurado, chegando a valores muito acima do alvo de 50%. Como consequência, o HPA disparou automaticamente o escalonamento, aumentando o número de réplicas do Deployment além do mínimo original, e novos Pods do PostgREST foram criados e entraram em estado `Running` para absorver a carga. Isso comprova que o Horizontal Pod Autoscaler está funcionando corretamente: ele monitora a métrica de CPU via metrics-server e ajusta o número de réplicas em tempo real conforme a demanda sobe, sem qualquer intervenção manual.


