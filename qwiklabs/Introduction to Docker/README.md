## Visão Geral

O Docker é uma plataforma aberta para desenvolvimento, envio e execução de aplicativos. Com ele você pode separar seus aplicativos de sua infraestrutura e tratá-lo como um aplicativo gerenciado. A abstração que o Docker fornece ajuda a enviar o código mais rápidamente, testar mais rápido, implantar mais rápido e reduzir o ciclo entre a escrita e a execução do código.

Tudo isso é feito com o Docker combinando recursos de contêiner de kernel com fluxos de trabalho e ferramentas que ajudam a gerenciar e implantar seus aplicativos.

Os contêiners Docker podem ser usados diretamente no Kubernetes, o que permite que eles sejam executados no Kubernetes Engine com facilidade. Depois de aprender os fundamenos do Docker, você terá o conjunto de habilidades para começar a desenvolver Kubernetes e aplicativos em contêiners.

## O que você aprenderá

Neste laboratório, você aprenderá a fazer o seguinte:

* Como construir, executar e depurar contêineres Docker.
* Como extrair imagens do Docker do Docker Hub e do Google Container Registry.
* Como enviar imagens do Docker para o Google Container Registry.

## Pré-requisitos

Este é um laboratório de nível introdutório. Presume-se que haja um pouco ou nenhum contato com Docker e contêiners. Familiaridade com o Cloud Shell é sugerida, mas não obrigatória.

## O que você precisa

Para concluir este laboratório, você precisa:

* Acesso a um navegador de Internet padrão (navegador Chrome recomendado).

## Cloud Shell

O Cloud Shell é uma máquina virtual carregada com ferramentas de desenvolvimento. Ele oferece um diretório inicial persistente de 5GB e é executado no Google Cloud. O Cloud Shell também fornece acesso a linha de comando aos recursos do Google Cloud.

O `gcloud` é um CLI do Google Cloud. Ele vem pré-instalado no Cloud Shell e ofere suporte para preenchimento com guia (completar automaticamente).

Você pode listar o nome da conta ativa com este comando:

```bash
gcloud auth list
```

Você pode listar o ID do projeto com este comando:

```bash
gcloud config list project
```

## Olá Mundo

Abra o Cloud Shell e digite o seguinte comando para executar um contêiner hello world para começar:

```bash
docker run hello-world
```

Esse contêiner simples retorna um `Hello from Docker!` na sua tela. Embora isso parece trivial, observe na saída o número de etapas que ele executou. O Docker Daemon procurou a imagem hello-word localmente e não encontrou, então puxou a imagem de um registro público chamado Docker Hub, criou um contêiner a partir dessa imagem e executou o contêiner para você.

Execute o seguinte comando para dar uma olhada na imagem do contêiner extraída do Docker Hub:

```bash
docker images
```

Esta é a imagem obtida do registro público do Docker Hub. O ID da imagem está no formado de hash-SHA256. Quando o Daemon do Docker não consegue encontrar uma imagem local, ele irá, por padrão, procurar a imagem no registro público. Se você executar o mesmo comando de `docker run` notará que o Daemon encontrou a sua imagem no registro local e executou o contêiner a partir dessa imagem. Portanto, não foi necessário extrair do Docker Hub.

Por fim, observe os contêineres em execução executando o seguinte comando:

```bash
docker ps
```

Não há contêineres em execução. Os contêineres hello-world que você executou anteriormente já foram encerrados. Para ver todos os contêineres, incluindo aqueles que já concluíram sua execução, execute `docker ps -a`. Isso mostra a você o Container ID UUID gerado pelo Docker para identificar o contêiner e mais metadados sobre a sua execução. O campo Names também é gerado aleatoriamente, mas pode ser especificado com:

```bash
docker run --name [container-name] hello-world
```

## Construir

Nesse tópico iremos monstrar como construir uma imagem Docker baseada em um aplicativo em Node.

```bash
mkdir test && cd test
```

Crie um `Dockerfile`:

```bash
cat > Dockerfile <<EOF
# Use an official Node runtime as the parent image
FROM node:6

# Set the working directory in the container to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Make the container's port 80 available to the outside world
EXPOSE 80

# Run app.js using node when the container launches
CMD ["node", "app.js"]
EOF
```

Este arquivo instrui o Daemon do Docker sobre como construir sua imagem. Cada layer tem um objetivo e ação específica. Agora você escreverá o aplicativo e, depois disso, construirá a imagem.

```bash
cat > app.js <<EOF
const http = require('http');

const hostname = '0.0.0.0';
const port = 80;

const server = http.createServer((req, res) => {
    res.statusCode = 200;
      res.setHeader('Content-Type', 'text/plain');
        res.end('Hello World\n');
});

server.listen(port, hostname, () => {
    console.log('Server running at http://%s:%s/', hostname, port);
});

process.on('SIGINT', function() {
    console.log('Caught interrupt signal and will exit');
    process.exit();
});
EOF
```

Este é um servidor HTTP simples que escuta na porta 80 e retorna **Hello World**. Agora vamos construir a imagem. Observe novamente o `.`, que significa diretório atual, portanto, você precisa executar este comando de dentro do diretório que contém o Dockerfile:

```bash
docker build -t node-app:0.1 .
```

A execução deste comando pode demorar alguns minutos. O parâmetro `-t` serve para nomear e marcar uma imagem com a sintaxe `name:tag`. O nome da imagem é `node-app` e a tag é `0.1`. A tag é altamente recomendada ao construir imagens Docker. Se você não especificar uma tag, por padrão a tag será `latest` e será mais difícil distinguir as imagens mais recentes das mais antigas. Observer também como cada linha do Dockerfile resulta em camadas de contêiner intermediárias conforme a imagem é construída. Agora, execute o seguinte comando para ver as imagens que você construiu:

```bash
docker images
```

Você verá que possuimos duas imagens com o prefixo `node`. Isso significa que para construir a imagem `node-app`, foi necessária a imagem base `node` e caso você queira remover a imagem `node`, você também terá que remover a imagem `node-app`.

O tamanho dessa imagem é relativamente pequeno em comparação com VM. Outras versões da imagem do Node como `node:slim` e `node:alpine` podem forncer uma imagem ainda menor para facilitar o transporte. 

## Run

Neste módulo, use este código para executar contêineres com base na imagem que você criou:

```bash
docker run -p 4000:80 --name my-app node-app:0.1
```

A tag `--name` permite que você nomeie o contêiner, se desejar. O `-p` instrui o Docker a mapear a parte 4000 do host para a porta 80 do contêiner. Agora você pode acessar o servidor em `http://localhost:4000`. Sem o mapeamento da porta, você não seria capaz de alcançar o contêiner no localhost.

Abra outro terminal (no Cloud Shell, clique no +ícone) e teste o servidor:

```bash
curl http://localhost:4000
```

O contêiner será executado enquanto o terminal inicial estiver em execução. Se deseja que o contêiner seja executado em segundo plano (não vinculado à sessão do terminal), é necessário especificar o parâmetro `-d`.

Feche o terminal inicial e execute o seguinte comando para parar e remover o contêiner:

```bash
docker stop my-app && docker rm my-app
```

Agora execute o seguinte comando para iniciar o contêiner em segundo plano:

```bash
docker run -p 4000:80 --name my-app -d node-app:0.1

docker ps
```

Observe que o contêiner está sendo executado na saída de `docker ps`. Você pode ver os logs executando `docker logs [container_id]`.

>
> Dica: Você não precisa escrever o ID do contêiner inteiro, desde que os caracteres iniciais identifiquem o contêiner de maneira exclusiva.
>

```bash
docker logs [container_id]
```

Edite `app.js` com um editor de texto de sua escolha (por exemplo, nano ou vim) e substitua **Hello World** por outra string:

```js
const server = http.createServer((req, res) => {
    res.statusCode = 200;
      res.setHeader('Content-Type', 'text/plain');
        res.end('Welcome to Cloud\n');
});
```

Crie esta nova imagem e marque-a com 0.2:

```bash
docker build -t node-app:0.2 .
```

Observe que na Etapa 2 usamos uma camada de cache existente. Da Etapa 3 em diante, as camadas são modificadas porque fizemos uma alteração em `app.js`.

## Depurar

Nesse momento em que estamos já familiarizados com a construção , vamos examinar algumas práticas de depuração (análise de logs).

Você pode ver os logs de um contêiner usando `docker logs [container_id]``. Se você quiser seguir a saída do log enquanto o contêiner está em execução, use a opção `-f`.

```bash
docker logs -f [container_id]
```

Às vezes, você desejará iniciar uma sessão Bash interativa dentro do contêiner em execução. Você pode usar o `docker exec` para fazer isso:

```bash
docker exec -it [container_id] bash
```

Os parâmetros `-it` permitem que você interaja com um contêiner alocando um pseudo-tty e mantendo o stdin aberto. 

Saia da sessão Bash. No novo terminal, digite:

```bash
exit
```

Você pode examinar os metadados de um contêiner no Docker usando Docker inspect:

```bash
docker inspect [container_id]
```

Use --formatpara inspecionar campos específicos do JSON retornado. Por exemplo:

```bash
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' [container_id]
```

## Publicar

Agora você vai enviar sua imagem para o Google Container Registry (gcr). Depois disso, você removerá todos os contêineres e imagens para simular um ambiente novo e, em seguida, puxe e execute seus contêineres. 

Para enviar imagens ao seu registro privado hospedado pelo gcr, você precisa marcar as imagens com um nome de registro. O formato é `[hostname]/[project-id]/[image]:[tag]`.

Para gcr:

* `[hostname]` = gcr.io
* `[project-id]` = ID do seu projeto
* `[image]` = nome da sua imagem
* `[tag]` = qualquer tag de string de sua escolha. Se não for especificado, o padrão é "mais recente".

Você pode encontrar o ID do seu projeto executando:

```bash
gcloud config list project
```

Tag `node-app:0.2`. Substitua `[project-id]` pela sua configuração...

```bash
docker tag node-app:0.2 gcr.io/[project-id]/node-app:0.2
```

Envie esta imagem para gcr. Lembre-se de substituir `[project-id]`.

```bash
docker push gcr.io/[project-id]/node-app:0.2
```

Verifique se a imagem existe no gcr visitando o registro de imagens em seu navegador. Você pode navegar através do console de **Ferramentas > Registro Container** ou visite: `http://gcr.io/[project-id]/node-app`


Vamos testar essa imagem. Você pode iniciar uma nova VM, ssh nessa VM e instalar o gcloud. Para simplificar, vamos apenas remover todos os contêineres e imagens para simular um ambiente novo.

Pare e remova todos os recipientes:

```bash
docker stop $(docker ps -q)
docker rm $(docker ps -aq)
```

Você deve remover as imagens filhas (de `node:6`) antes de remover a imagem do nó. Substitua `[project-id]`

```bash
docker rmi node-app:0.2 gcr.io/[project-id]/node-app node-app:0.1
docker rmi node:6
docker rmi $(docker images -aq) # remove remaining images
docker images
```

Nesse ponto, você deve ter um ambiente pseudo-fresco. Puxe a imagem e execute-a. Lembre-se de substituir o `[project-id]`.

```bash
docker pull gcr.io/[project-id]/node-app:0.2
docker run -p 4000:80 -d gcr.io/[project-id]/node-app:0.2
curl http://localhost:4000
```
