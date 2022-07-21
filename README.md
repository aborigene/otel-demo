# Laboratório de Observabilidade

Demo baseada em microserviços com o objetivo de demonstrar conceitos de Observabilidade, OpenTelemetry e como podemos atingir Observabilidade com Dynatrace

![Architecture](https://raw.githubusercontent.com/martinnirtl/otel-demo/master/docs/img/architecture-diagram.png)

## Explicando a aplicação

A aplicação em si não tem um propósito específico nem oferece uma bela interface do usuário. Ele apenas fornece um endpoint simples para os usuários se inscreverem com e-mail, nome e senha e envia um e-mail de confirmação posteriormente. Toda a comunicação do serviço é feita via HTTP.

Embora existam alguns serviços ao redor para tornar este exemplo mais representativo, os principais componentes são os seguintes:

1. Backend Service
2. Mail Service
3. Template Service

O serviço de back-end e o serviço de modelo serão monitorados por meio do OneAgent e criarão alguns intervalos personalizados do OpenTelemetry por meio de instrumentação manual. O serviço intermediário - o Mail Service - será instrumentado apenas com OpenTelemetry.

O procedimento de inscrição pode ser descrito em 6 passos simples:

1. Signup-endpoint é chamado com uma chamada HTTP-post com e-mail, nome e senha no corpo
2. Após a validação do endereço de e-mail, o usuário é armazenado no banco de dados do Mongo
3. O serviço de back-end chama o ponto de extremidade de envio do serviço de correio via HTTP-post para enviar um e-mail de confirmação de inscrição
4. O Mail Service chama o Template Service via gRPC para renderizar o corpo do e-mail
5. Após renderizar o email, o Template Service armazena o resultado no cache do Redis e retorna
6. Finalmente, o Mail Service chama um provedor de correio como serviço externo (por exemplo, Sendgrid) para enviar o e-mail

## Executar a Demo

Vamos executar a demo num Kubernetes, de acordo com o apresentado abaixo. Recomendo o uso do minikube, assim não temos problemas de acessso. Sendo assim deve ser seguido o passo a passo de instalação do minikube e em seguida a sua subida conforme o comando abaixo:

```
minikube start --memory 6096 --addons=ingress
```

Se der um erro reclamando da quantidade de memória, diminuir para 4096 ou deixar sem o parâmetro da memória, mas pode ser que o cluster fique sobrecarregado durante as atividades e pare de rodar.

Depois que o minikube subir rodar em um terminar separado o seguinte comando:

```
minikube tunnel
```

Este comando permitirá o acesso às aplicações rodando dentro do cluster Kubernetes.


Para subir o ambiente vamos executar os passos abaixo

1. Setup do Graylog

Os arquivos para deploy do lab Graylog estão no diretório ```<RAIZ_DO_REPO>/kubernetes/graylog```.

Acesse esse diretório para a execução dos comandos:

```cd <RAIZ_DO_REPO>/kubernetes/graylog```

Você deve alterar a senha padrão para fazer login na interface web do Graylog, para isso você deve executar o seguinte
comando em seu terminal

```
echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d " " -f 1

```
Este comando solicitará que você insira sua senha e, em seguida, copie a senha com hash gerada e guarde em um arquivo temporário.

Edit o arquivo graylog-deploy.yaml:

```vim gryalog-deploy.yaml```

Nesse arquivo adicione os valores listados nos próximos passos.

Você precisa personalizar o valor GRAYLOG_HTTP_EXTERNAL_URI para que ele aponte para seu host local ou remoto, este é o endereço real do seu cluster ou o IP de acesso do seu minikube

```
  - name: GRAYLOG_HTTP_EXTERNAL_URI
    value: #your_remote_or_localhost_ip
```
Inicialmente não altere o valor acima, ajustaremos esse valor após o deploy do serviço.

No passo abaixo adicione a senha que você gerou anteriormente:

```
  - name: GRAYLOG_ROOT_PASSWORD_SHA2
    value: generate_hashed_password_aqui
```

Você pode verificar o arquivo de configuração do Graylog [graylog.conf](https://github.com/Graylog2/graylog2-server/blob/master/misc/graylog.conf) para mais detalhes.

#### Implantando a stack Graylog:

Para implantar a pilha, você cria um deploy kubernetes da seguinte maneira:

```
kubectl create ns graylog
kubectl create -f es-deploy.yaml
kubectl create -f mongo-deploy.yaml
kubectl apply -f graylog-deploy.yaml
```

Para o acesso funcionar precisamos fazer um ajuste, precisamos pegar o IP real do meu serviço e atualizar a configuração, para isso execute o seguinte comando:

```
kubectl get service -n graylog
```

Procure na lista o IP real referente ao serviço graylog3, altere o arquivo graylog-deploy.yaml adicionando esse IP:

```vim gryalog-deploy.yaml```

```
  - name: GRAYLOG_HTTP_EXTERNAL_URI
    value: #IP_COLETADO_NO_PASSO_ANTERIOR
```

Após a alteração do arquivo aplicá-lo novamente:

```
kubectl apply -f graylog-deploy.yaml
```

Você pode verificar a implantação usando o seguinte comando:

```
 kubectl get deploy -n graylog
```
 
Você também pode verificar os pods criados por estes deploys:

```
 kubectl get pods -n graylog
```
 
#### Faça login na interface web do Graylog:

Depois de iniciar o Graylog, você pode fazer login na interface web Graylog da seguinte maneira:

<br/><br/>![Página de login do Graylog](kubernetes/graylog/images/loging-page.PNG)

> Altere <your_ip_address> pelo seu endereço definido nos passos acima
> Usuário: admin
> Senha: a que você criou nos passos acima

#### Criando uma entrada TCP Gelf
Após o login temos que criar uma entrada para receber as mensagens de logs.
Para fazer isso:

- Acessar System -> Input
- Escolher GELF TCP da lista
- Preencher o nome dessa input como "FluentBit integration"
- Clicar em salvar

2. Setup do FluentBit

```
cd <base do repositorio>/kubernetes/fluentbit
kubectl create namespace logging
kubectl apply -f service-account.yaml
kubectl apply -f rbac-role.yam
kubectl apply -f role-binding.yaml
```

Editar o arquivo fluent-bit-cm.yaml na sessão output-graylog.conf adicionar o ip interno do serviço do Graylog e a porta da input, como apresentado na interface, ficará algo assim:

```
  output-graylog.conf: |
    [OUTPUT]
        Name                    gelf
        Match                   *
        Host                    10.0.10.144
        Port                    12201
        Mode                    tcp
        Gelf_Short_Message_Key  log 
```

Executar os deploys:

```
kubectl apply -f fluent-bit-cm.yaml
kubectl apply -f fluent-bit-graylog-ds.yaml
```

3. Setup do Prometheus

```
cd <raiz do repositorio>/kubernetes/prometheus
kubectl create ns monitoring
kubectl apply -f clusterRole.yaml
kubectl apply -f config-map.yaml -n monitoring
kubectl apply -f prometheus-deployment.yaml -n monitoring
kubectl apply -f prometheus-service.yaml -n monitoring
```

Aguardar alguns minutos e verificar a IP/porta em que o serviço do Prometheus está exposto:

```
kubectl get service/prometheus-service -n monitoring 
```

Apontar o navegador para o IP/porta para acessar o serviço

4. Setup do Jaegger

Instalar o cert manager:
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.2/cert-manager.yaml
```

Configurar um ingress controller, no minikube usar o parâmetro
```
--addons=ingress
```
Seguir com a aplicação dos arquivos:

```
cd <raiz do repositorio>/kubernetes/jaeger

kubectl create namespace observability
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.32.0/jaeger-operator.yaml -n observability

```
Verificar que operator está rodando:
```
kubectl get deployment jaeger-operator -n observability
```

Criar o Jaeger:
```
kubectl apply -f jaeger.yaml -n observability
```

Aguardar alguns minutos e verificar a IP/porta em que o serviço do Jaeger está exposto:

```
kubectl get service/simplest-query -n monitoring 
```
ou
```
kubectl get ingress -n observability
```

Apontar o navegador para o IP/porta para acessar o serviço, pode ser o IP do seriço ou do Ingress

5. Setup do Grafana

```
cd <raiz do repositorio>/kubernetes/grafana
kubectl apply -f grafana-datasource-config.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Aguardar alguns minutos e verificar a IP/porta em que o serviço do Grafana está exposto:

```
kubectl get service/grafana -n observability 
```

Apontar o navegador para o IP/porta para acessar o serviço

A senha padrão para o primeiro login é:
User: admin
Senha: admin

6. Setup da Aplicação

```
cd <raiz do repositorio>/kubernetes/services
kubectl apply -f .
```

7. Setup do coletor OpentTelemetry

```
cd <raiz do repositorio>/kubernetes/opentelemetry
kubectl apply -f opentelemetry-collector.yaml 
```

A aplicão não tem interface exposta, o gerador de carga irá gerar tráfego para os testes.

### Exercício extra - Envio de logs vai HTTP

O objetivo deste exercício é executar o envio de logs vai HTTP, apenas para demonstrar outra forma de enviar logs para o graylog.

Para isso temos um job que gera logs aleatórios ao ambiente, para iniciar esse job deve-se executar:

```
cd <raiz do repositório>/kubernetes/graylog
kubectl create -f cronJob.yaml
```

Nesse momento como estão os logs no graylog? O que está faltando?

Como podemos melhorara a visualização e filtragem desses logs?

Analise um pouco e tente entender o que está acontecendo, depois veja os detalhes desse laboratório nesse documento: GraylogExecicioExtra.md [GraylogExecicioExtra.md](https://github.com/aborigene/otel-demo/blob/master/kubernetes/graylog/GraylogExecicioExtra.md) for more details.


### Tendo problemas ou dificuldades ou encontrou um bug?

Envie um email: [igor.simoes@dynatrace.com](mailto:igor.simoes@dynatrace.com), [krishnarupa@gmail.com](mailto:krishnarupa@gmail.com)
