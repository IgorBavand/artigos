## Jenkins

Jenkins é um sistema de servidor de automação de código aberto, com integração e entrega contínuas, que permite a criar, testar e implementar  softwares, tornando, assim, o processo de desenvolvimento mais ágil e eficiente.

No exemplo abaixo estarei mostrando o passo a passo para a utilização do Jenkins no deploy de uma aplicação Spring Boot utilizando pipeline.

### Instalação do jenkins via Docker

Primeiro criei um network para colocar os containers que desejo organizar.

`docker network create minha_network`

Em seguida realizo a instalação do Jenkins.

```
docker run -d -u root --name jenkins
  --network minha_network --privileged
  -v /var/run/docker.sock:/var/run/docker.sock
  -v $(which docker):/usr/bin/docker
  -p 8080:8080 -p 50000:50000
  jenkins/jenkins:lts
```

### Configuração

Após a inicialização do container o Jenkins estará disponível na porta 8080 ([http://localhost:8080](http://localhost:8080)), no priimeiro acesso será solicitado a senha de administrador, essa senha pode ser acessada com o seguinte comando no terminal:

`docker exec <container_id> cat /var/jenkins_home/secrets/initialAdminPassword`

lembre-se de substituir o `<container_id>` pelo id do seu container.

Após a senha será feita a solicitação para criar um novo usuario e a instalação dos plugins.

### Criando Job

Vá até o canto superior esquedo da tela e clique em `nova tarefa` ou `new job`

Dê um nome ao seu job, selecione `pipeline` e depois clique para continuar.

[criando_job]

Em seguida vá até a até `Build Triggers` selecione a opção `construir periodicamento o SCM` e digite `* * * * *` na angenda, caso queira verificar constantemente alterações no seu repositório.

Feito isso, vá até `pipeline` e configure a url do seu repositório git, a branch que deseja usa, e ao final o `Script Path` coloque o valor que usará no nome do arquivo de configuração que será adicionado na aplicação, nesse exemplo usarei `Jenkinsfile`

[configurando_git]

[jenkinsfile]

Agora é so salvar.

### Configurando Dockerfile no projeto

Para configurar o primeiro passo é criar um arquivo chamado `Dockerfile` na raiz do seu projeto, esse passo é muito importante pois o Dockerfile é responsavel por definir como será gerada a imagem para o docker.

Nesse caso estarei usando um projeto que utiliza o java 17, o arquivo Dockerfile ficou assim:

```
FROM openjdk:17-jdk-slim
COPY target/keycloakauth-0.0.1-SNAPSHOT.jar /app/keycloakauth-0.0.1-SNAPSHOT.jar
WORKDIR /app
EXPOSE 8340
CMD ["java", "-jar", "keycloakauth-0.0.1-SNAPSHOT.jar"]
```

Para encontar os arquivos `.jar` execute o comando `mvn clean install -Dmaven.test.skip=true` e será gerado o jar dentro da pasta `target`

### Configurando o Jenkinsfile no projeto

O jenkins file será o responsável por dizer quais passos (stages) o Jenkins deve executar antes do deploy, para configurarmos vamos primeiro criar na raiz do nosso projeto um arquivo chamado `Jenkinsfile`, aqui pode ser configurado diversas coisas, mas no caso desse exemplo as steps serã:

* package
* test
* build
* deploy

vou definir também no arquivo o nome do container, porta e o ambiente que ele estará sendo executado. (dev)

lembrando que no spring, para que funcione essa configuração de ambiente deve ser criado o arquivo `application-dev.yml`

esse é o arquivo `Jenkinsfile` criado conforme as especificações acima:

```
pipeline {
    agent any
    environment {
        PORT = "8340:8340"
        CONTAINER_NAME = "keycloak-auth-api"
        ENVIRONMENT = "dev"
    }
    tools {
        maven 'Maven3.6'
    }
    stages {
        stage('package') {
            steps {
                sh 'mvn clean package -DskipTests=true'
            }
        }
        stage('test') {
            steps {
                sh 'mvn clean verify'
            }
        }
        stage('build') {
            steps {
                sh '''
                    docker build -t ${CONTAINER_NAME} .
                '''
            }
        }
        stage('deploy') {
            steps {
                script {
                    def containerExists = sh(script: "docker ps -a --format '{{.Names}}' | grep ${CONTAINER_NAME}", returnStatus: true)
                    if (containerExists == 0) {
                        sh "docker stop ${CONTAINER_NAME}"
                        sh "docker rm ${CONTAINER_NAME}"
                    }
                    sh '''
                        docker run -d --name=${CONTAINER_NAME} --restart=unless-stopped -p ${PORT} \
                        --net minha_network \
                        -e SPRING_PROFILES_ACTIVE=${ENVIRONMENT} \
                        ${CONTAINER_NAME}
                    '''
                }
            }
        }
    }
}

```

### Conclusão

Após a configuração anterior faça o push das alterações para a branch que definiu no Jenkins e já estará funcionando.

[stages]

Container da aplicação após o sucesso no Jenkins:
