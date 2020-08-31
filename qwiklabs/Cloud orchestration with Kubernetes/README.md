## Visão Geral

Nessa seção iremos aprender:

* Como implantar e gerenciar contêineres Docker usando kubectl
* Provisionar um cluster completo do Kubernetes usando o Kuberntes Engine
* Dividir um aplicativo em micro-serviços usando as implantações e serviços do Kubernetes

O Kubernetes gira em torne de aplicativos. Para isso, usaremos um exemplo chamado **app**. O **app** está hospedado no GitHub e oferece um aplicativo de exemplo. Imagens Docker utilizadas:

* kelseyhightower/monolith: o monolith inclui serviços auth e hello.
* kelseyhightower/auth: um microsserviço auth. Ele gera tokens JWT para usuários autenticados.
* kelseyhightower/hello: um microsserviço hello. Ele saúda os usuários autenticados.
* ngnix: front-end para serviços auth e hello.

O projeto do Kubernetes é de código aberto que pode ser executado em diversos ambientes, de laptops a clusters com vários nós de alta disponibilidade, de nuvens públicas a implantações locais e de máquinas virtuais a bare metal.

Nesse laboratório iremos utilizar um ambiente gerenciado, como o Kubernetes Engine, para que você não perca tempo configurando toda a infraestrutura subjacente ao Kubernetes. 

## Receba o código de amostra

```bash
git clone https://github.com/googlecodelabs/orchestrate-with-kubernetes.git
```

## Demonstração rápida do Kubernetes

A maneira mais fácil de começar a usar o Kubernetes é usando o comando `kubectl create`. Exemplo:

```bash
kubectl create deployment nginx --image=nginx:1.10.0
```

Por de baixo dos panos o Kubernetes criará uma implantação. Tudo que você precisa saber é que as implantações mantêm os pods funcionando mesmo quando há falha nos nós em que eles são executados.

No kubernetes, todos os contêineres são executados em um pod. Use o comando `kubectl get pods` para ver o contêiner nginx em execução. Quando o contêiner nginx está em execução, você pode expô-lo fora do Kubernetes usando o `kubectl expose`:

```bash
kubectl expose deployment nginx --port 80 --type LoadBalancer
```

Ao executar isso, o que acontece? O Kubernetes criará, em segundo plano, um balanceador de carga externo com um endereço IP público anexado a ele. Qualquer cliente que alcançar esse endereço IP público será encaminhado para os pods que estão por trás do serviço. Nesse caso, seria o pod do nginx.

```bash
kubectl get services
```

>
> O preenchimento do campo ExternalIP do seu service pode demorar alguns segundos. Isso é normal. Basta executar novamente após alguns segundos.

Adicione o IP externo a esse comando para acessar o contêiner Nginx remotamente:

```
curl http://<External IP>:80
```

## Pods

No centro do Kubernetes estão os Pods. Eles representam e retêm uma coleção de um ou mais contêineres. Geralmente, quando há vários contêineres com forte dependência uns dos outros, você os empacota dentro de um único pod.

Os pods também possuem Volumes. Volumes são discos de dados que duram o tempo que durarem os pods e podem ser usados pelos contêineres desses pods. Os pods fornecem um namespace compartilhado do conteúdo deles, ou seja, os dois contêineres dentro de um pod podem se comunicar entre si e também compartilham os volumes anexados. 

Eles compartilham também uma namespace de rede. Isso significa que há um endereço IP por pod.

## Crie pods

Os Pods podem ser criados usando arquivos de configuração de Pod. Exemplo:

```yml
apiVersion: v1
kind: Pod
metadata:
  name: monolith
  labels:
    app: monolith
spec:
  containers:
    - name: monolith
      image: kelseyhightower/monolith:1.0.0
      args:
        - "-http=0.0.0.0:80"
        - "-health=0.0.0.0:81"
        - "-secret=secret"
      ports:
        - name: http
          containerPort: 80
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"
```

Há algumas coisas para serem observadas:

* Seu pod é composto por um contêiner chamado **monolith**.
* Alguns argumentos são passados para esse contêiner quando ele iniciar.
* A porta 80 será aberta para o tráfego http.

Crie o pod usando o comando:

```bash
kubectl create -f pods/monolith.yaml
```

Examine seus pods com o comando `kubectl get pods`, ele listará todos os pods em execução na namespace padrão.

>
> Pode levar alguns segundos para ativar o pod. A imagem precisa ser extraída do Docker Hub para que ele possa ser executado.
>

Quando o pod estiver funcionando, use o comando `kubectl describe` para receber mais informações sobre ele.

```bash
kubectl describe pods monolith
```

Você verá várias informações sobre ele, incluindo o endereço IP do pod e o log de eventos. Essas informações serão úteis na solução de problemas.

O Kubernetes facilita a criação de pods, basta descrevê-los nos arquivos de configuração, além de simplificar a visualização das informações sobre eles quando estão em execução.

## Interaja com os pods

Por padrão os pods recebem um endereço I´P privado e não podem ser alcançados fora do Cluster. Use o comando `kubectl port-forward` para mapear uma porta local para uma porta dentro de um pod.

```bash
kubectl port-forward monolith 10080:80
curl http://127.0.0.1:10080
curl http://127.0.0.1:10080/secure
curl -u user http://127.0.0.1:10080/login
TOKEN=$(curl http://127.0.0.1:10080/login -u user|jq -r '.token')
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:10080/secure
```

Use o comando `kubectl logs` para ver os registros do pod **monolith**

```bash
kubectl logs monolith
```

Use o parâmetro `-f` para receber um stream dos registros em tempo real

```bash
kubectl logs -f monolith
```

Use o comando `kubectl exec` para executar um shell interativo dentro do pod do **monolith**. Isso pode ser útil para solucionar problemas de dentro de um contêiner:

```bash
kubectl exec monolith --stdin --tty -c monolith /bin/sh
```

Como você pode ver a interação com pods é tão fácil quanto usar o comando `kubectl`. Seja para acessar remotamente um contêiner ou para receber um shell de login, o Kubernetes fornece tudo o que você precisa para iniciar e seguir em frente.

## Serviços

Os pods não foram criados para serem persistentes. Eles podem ser interrompidos ou iniciados po muitas razões, por exemplo, falha nas verficações de integridade ou de prontidão, o que gera um problema.

O que acontece quando você quer se comunicar com um conjunto de pods? Quando eles forem reiniciados, poderão ter um endereço IP diferente. É aqui que entram os Serviços. Eles fornecem pontos de extremidade estáveis para pods.

Os serviços aplicam rótulos para determinas os pods em que eles operam. Se os pods tiverem os rótulos corretos, serão automaticamente selecionados e expostos por nossos serviços. O nível de acesso de um serviço a um conjunto de pods depende do tipo de serviço. Atualmente existem três tipos:

* `ClusterIP` é o tipo padrão, significa que esse serviço só é visível dentro do cluster.
* `NodePort` concede a cada nó do cluster um IP externamente acessível.
* `LoadBalancer` adiciona um balanceador de carga do provedor de nuvem que encaminha o tráfego do serviço para os nós dentro dele.

## Crie um serviço

Antes de criar nossos serviços, vamos primeiro criar um pod seguro que possa lidar com tráfego HTTPS. Crie os pods de secure-monolith e os dados de configuração deles:

```bash
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-proxy-conf --from-file nginx/proxy.conf
kubectl create -f pods/secure-monolith.yaml
```

Agora que você tem um pod seguro, é hora de expor o pod de secure-monolith externamente.Para fazer isso, crie um serviço do Kubernetes.

```yml
kind: Service
apiVersion: v1
metadata:
  name: "monolith"
spec:
  selector:
    app: "monolith"
    secure: "enabled"
  ports:
    - protocol: "TCP"
      port: 443
      targetPort: 443
      nodePort: 31000
  type: NodePort
```

* Existe um seletor que é usado para localizar e expor automaticamente todos os pods com os rótulos **app=monolith** e **secure=enabled**.
* Agora é preciso expor um node port para definir como vamos encaminhar o tráfego externo da porta 310000 para nginx, na porta 443.

```bash
kubectl create -f services/monolith.yaml
```

Você está usando uma porta para expor o serviço. Isso significa que poderá haver colisões de porta se outro app tentar se vincular à porta 31000 em um de seus servidores. Normalmente, o Kubernetes lidaria com essa atribuição de porta. Neste laboratório, você escolheu uma porta para facilitar a configuração das verificações de integridade mais tarde.

Use o comando `gcloud compute firewall-rules` para permitir tráfego para o serviço do monolith no nodeport exposto:

```bash
gcloud compute firewall-rules create allow-monolith-nodeport \
  --allow=tcp:31000
```

Escolha um endereço IP externo para um dos nós.

```bash
gcloud compute instances list
```

Agora, tente acessar o serviço secure-monolith usando `curl`:

```bash
curl -k https://<EXTERNAL_IP>:31000
```

Ops, o tempo limite esgotou. O que houve de errado?

É hora de fazer uma verificação rápida de conhecimento.Use os seguintes comandos para responder às perguntas abaixo:

* `kubectl get services monolith`
* `kubectl describe services monolith`

Dica: isso está relacionado a rótulos. Você corrigirá o problema na próxima seção.

## Adicione rótulos a pods

Atualmente, o serviço do monolith não tem pontos de extremidade. Uma maneira de solucionar um problema como este é usar o comando `kubectl get pods` com uma consulta de rótulo.

Podemos ver que temos alguns pods em execução com o rótulo do monolith.

```bash
kubectl get pods -l "app=monolith"
```

Mas e o "app=monolith" e o "secure=enabled"?

```bash
kubectl get pods -l "app=monolith,secure=enabled"
```

Observe que essa consulta de rótulo não imprime nenhum resultado. Parece que precisamos adicionar o rótulo "secure=enabled" a eles.

Use o comando `kubectl label` para adicionar ao pod do `secure-monolith` o rótulo `secure=enabled` que falta. Depois, verifique se os rótulos foram atualizados.

```bash
kubectl label pods secure-monolith 'secure=enabled'
kubectl get pods secure-monolith --show-labels
```

Agora que seus pods estão rotulados corretamente, vamos ver a lista de pontos de extremidade no serviço do monolith:

```bash
kubectl describe services monolith | grep Endpoints
```

Vamos testá-lo acessando um de nossos nós novamente.

```bash
gcloud compute instances list
curl -k https://<EXTERNAL_IP>:31000
```

## Implante aplicativos com o Kubernetes

O objetivo desse laboratório é prepará-lo para escalonar e gerenciar contêineres na produção. É aqui que entram as implantações. As implantações são uma maneira declarativa de garantir que o número de pods em execução seja igual ao número desejado de pods especificado pelo usuário.

O principal benefício das implantações é a abstração dos detalhes a nível inferior do gerenciamento de pods. As implantações usam controladores de replicação ReplicaSet em segundo plano para gerenciar o início e a interrupção dos pods. Se for necessário atualizar ou escalonar os pods, a implantação cuidará disso. A implantação também trata do reinício dos pods, caso eles parem por algum motivo.

Os pods estão vinculados ao ciclo de vida do nó no qual foram criados. 
