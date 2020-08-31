## Visão Geral

O Google Kubernetes Engine (GKE) oferece um ambiente gerenciado para implantação, gerenciamento e escalonamento de aplicativos em contêineres usando a infraestrutura do Google. O ambiente do Kubernetes Engine consiste em várias máquinas (especificamente, instâncias do Google Compute Engine) agrupadas para formar um cluster de contêiner. Nesse laboratório iremos criar contêineres e implantar aplicativos com o GKE.

## Orquestração de cluster com o Kubernetes Engine

Os clusters do Kubernetes Engine são fornecidos pelo sistema de gerenciamento de cluster de código aberto do Kubernetes. O Kubernetes oferece os mecanismos para você interagir com o cluster de contêiner. Você usa os comandos e recursos do Kubernetes para implantar e gerenciar os aplicativos, executar tarefas administrativas e definir políticas, além de monitorar a integridade das cargas de trabalho implantadas. 

O Kubernetes se baseia nos mesmos princípios de design que guiam os serviços do Google mais conhecidos e oferece os mesmos benefícios: gerenciamento automático, sondagem de ativação e monitoramento para contêiners de aplicativos, escalonamento automático, atualizações contínuas e muito mais. Ao executar seus aplicativos em um cluster de contêiner, você está usando tecnologia baseada nos mais de 10 anos de experiência do Google em execução de cargas de trabalho de produção em contêineres.

## Kubernetes no Google Cloud Platform

Quando você executa um Cluster do Kubernetes Engine, também tem acesso aos recursos avançados de gerenciamento de cluster que o GCP oferece:

* Balanceamento de carga para instâncias do Compute Engine
* Pools de nós para designar subconjuntos de nós em um cluster, o que aumenta a flexibilidade.
* Escalonamento automático da contagem de instâncias de nós do cluster.
* Atualizações automáticas do software de nós do cluster.
* Reparação automática de nós para manter a disponibilidade e a integridade dos nós.
* Geração de registros e monitoramento com o Cloud Monitoring para ter uma visão do cluster.

Agora que tem um conhecimento básico do Kubernetes, você aprenderá a implantar um aplicativo em contêiner com o Kubernetes Engine em menos de 30 minutos.

## Como definir uma zona do Compute padrão

A zona do Compute é um local regional próximo que armazena seus clusters e os recursos correspondentes. Por exemplo, `us-central1-a` é uma zona na região `us-central1`.

```bash
gcloud config set compute/zone us-central1-a
```

## Como criar um cluster do Kubernetes Engine

Um cluster consiste em pelo menos uma máquina mestre do cluster e diversas máquinas worker chamadas de nós. Os nós são instâncias de máquina virtual (VM) do Compute Engine que executam os processos do Kubernetes necessários para integrá-los ao cluster.

Para criar um cluster, execute o comando abaixo, substituindo `[CLUSTER-NAME]` pelo nome que você escolher (por exemplo, `my-cluster`). Os nomes de cluster precisam começar com uma letra, terminar com um caractere alfanumérico e não podem ter mais de 40 caracteres.

```bash
gcloud container clusters create [CLUSTER-NAME]
```

## Receba as credenciais de autenticação para o cluster

Depois de criar o cluster, você precisará de credenciais de autenticação para interagir com ele. Para autenticar o cluster, execute o comando abaixo, substituindo `[CLUSTER-NAME]` pelo nome do seu cluster:

```bash
gcloud container clusters get-credentials [CLUSTER-NAME]
```

## Como implantar um aplicativo no cluster

Agora que criou um cluster, você pode implantar um aplicativo em contêiner nele. Neste laboratório, você executará um `hello-app` no seu cluster. 

O Kubernetes Engine usa objetos do Kubernetes para criar e gerenciar os recursos do seu cluster. O Kubernetes fornece o objeto de `implantação` para implantar aplicativos sem estado como servidores da Web. Os objetos de `serviço` definem as regras e o balanceamento de carga para acessar seu aplicativo pela Internet.

Execute o seguinte comando `kubectl create` no Cloud Shell para criar um novo `hello-server` de implantação com a imagem de contêiner do `hello-app`:

```bash
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
```

O comando acima cria um objeto d eimplantação que representa `hello-server`. Nesse caso, o parâmetro `--image` especifica uma imagem de contêiner a ser implantada. Nesse caso, o comando extrai a imagem de exemple de um bucket do Google Container Registry, e `gcr.io/google-samples/hello-app:1.0` indica a versão específica da imagem a ser extraída. Se nenhuma versão for especifica, a mais recente é usada.

Agora execute o comando `kubectl expose` abaixo para criar um serviço do Kubernetes que expõe o aplicativo ao tráfego externo:

```bash
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
```

Nesse comando temos:

* `--port` para especificar a porta que o contêiner expõe.
* `type="LoadBalancer` cria um balanceador de carga do Compute Engine para o contêiner.

Inspecione o serviço `hello-server` executando `kubectl get`:

```bash
kubectl get service
```

>
> Observação: um endereço IP externo pode levar um minuto para ser gerado. Execute o comando acima novamente se a coluna EXTERNAL-IP estiver com um status pendente.
>

Na saída desse comando, copie o endereço IP externo do serviço da coluna `EXTERNAL IP`.

Veja o aplicativo no navegador da Web usando o endereço IP externo com a porta exposta:

```bash
http://[EXTERNAL-IP]:8080
```

## Limpeza

Execute o seguinte comando para excluir o cluster:

```bash
gcloud container clusters delete [CLUSTER-NAME]
```

Quando solicitado, digite Y para confirmar. A exclusão do cluster pode levar alguns minutos.
