# Lab-O2B

## Descrição do Laboratório:

Este laboratório é baseado no exemplo disponível no site [prometheus.io](https://prometheus.io/docs/introduction/overview/).

  

## Tecnologias Utilizadas:
* Linux (baseado no Ubuntu)
* WSL
* Docker
* Golang
* Prometheus
* Grafana
* Alertmanager




## Instalação do  Docker-compose

Instalar a ferramenta Compose é fácil, mas primeiro certifique-se de ter o Docker instalado.

Rode esse comando no seu terminal para saber se o Doceker realmente esta instalado.

```
docker --version

```
Se não aparecer uma mensagem semelhante a esta Docker version 20.10.21, significa que você ainda não tem o Docker instalado.

Caso esteja utilizando alguma distribuição Linux, basta realizar a instalação com o seguinte comando:

```
sudo curl -L "https://github.com/docker/compose/releases/download/v2.5.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
Os binários baixados da Internet não podem ser executados por padrão. Portanto, basta usar o programa chmod para aplicar permissões de execução (+x) ao binário que acabamos de baixar. Execute o seguinte comando em seu terminal:

```
sudo chmod +x /usr/local/bin/docker-compose
```
Agora basta executar docker-compose --version para verificar a instalação. Se tudo correr bem, você deverá ver a seguinte saída (ou semelhante) em seu terminal:

```
docker-compose --version
# Docker Compose version v2.5.0
```



  
- **Criação do arquivo de configuração `prometheus.yml`**
  
  ```
   
   global:
      scrape_interval: 15s
    

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']
  
      - job_name: 'simple_server'
        static_configs:
          - targets: ["localhost:8090"]
    
      - job_name: 'node-exporter'
        static_configs:
          - targets: ['localhost:9100']
  
    ```

- **Criação do arquivo de configuração `docker-compose.yml`**

      

   ```docker
        version: '3'
        services:
          prometheus:
            image: prom/prometheus
            ports:
              - 9090:9090
            volumes:
              - ./prometheus.yml:/etc/prometheus/prometheus.yml
              - ./rules.yml:/etc/prometheus/rules.yml
            command:
              - '--config.file=/etc/prometheus/prometheus.yml'
            depends_on:
              - node-exporter
            network_mode: "host"
        
          node-exporter:
            image: prom/node-exporter
            ports:
              - 9100:9100
            network_mode: "host"

          grafana:
            image: grafana/grafana
            ports:
              - 3000:3000
            network_mode: "host"

          alertmanage:  
            container_name: alertmanager
            image: prom/alertmanager
            volumes:
              - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
            ports:
              - 9093:9093
            network_mode: "host"

    ```

 


         
         



### Instrumentando um Servidor HTTP Escrito em GO
1. **Criação do arquivo `server.go`**

    ```go
    package main
    import (
       "fmt"
       "net/http"
    )
    func ping(w http.ResponseWriter, req *http.Request){
       fmt.Fprintf(w,"pong")
    }
    func main() {
       http.HandleFunc("/ping",ping)
       http.ListenAndServe(":8090", nil)
    }
    ```

2. **Build do servidor, execução e teste**

    ```bash
    sudo add-apt-repository ppa:longsleep/golang-backports
    sudo apt update
    sudo apt install golang-1.18
    
    export PATH=$PATH:/usr/local/go/bin
    
    # Buildar o arquivo go
    go build server.go
    # Executar o servidor em outra linha de comando
    ./server

    
    # Acessar http://localhost:8090/ping
    ```

3. **Instrumentação da aplicação no arquivo `server.go`**

    ```go
    package main
    import (
       "fmt"
       "net/http"
       "github.com/prometheus/client_golang/prometheus"
       "github.com/prometheus/client_golang/prometheus/promhttp"
    )
    var pingCounter = prometheus.NewCounter(
       prometheus.CounterOpts{
           Name: "ping_request_count",
           Help: "No of requests handled by Ping handler",
       },
    )
    func ping(w http.ResponseWriter, req *http.Request) {
       pingCounter.Inc()
       fmt.Fprintf(w, "pong")
    }
    func main() {
       prometheus.MustRegister(pingCounter)
       http.HandleFunc("/ping", ping)
       http.Handle("/metrics", promhttp.Handler())
       http.ListenAndServe(":8090", nil)
    }
    ```

4. **Execução do exemplo**

    ```bash
    go mod init prom_example
    go mod tidy
    go build server.go
    ./server
    
    # Acessar http://localhost:8090/metrics
    ```


**Adicionando Prometheus para visualização de Métricas**

 **Execução do Docker-compose e subida dos containers**

Para subir os containers, execute o comando:

```
docker-compose up -d
```

Verificar se os containers estão up, execute o comando:

```
docker-compose ps
```

Acesse a porta http://9090 para abrir o Prometheus. Clique em Status, depois em Targets e verifique se todos estão ups. Observe que dará para coletar as métricas simple_serve, node-exporter e do prometheus.

### Visualizando Métricas Utilizando o Grafana para a criação de Dashboard

Acesse a porta http://3000 para abrir o Grafana. Para o laboratório eu usei para o login e senha: admin. Abrirá a página inicial do Grafana, clique em Data Source, abrirá nova página para você adicionar o Data Source, clique em Prometheus.

Essa página ele pede o name, endereçõ http://localhost:9090 para fazer a integração, clique em Save % Test para salvar e se comunicar com o prometheus.

Na mesma página clique em Dashboard, clique em Import.

Volte para home e acesse Complete - Create your first dashboard. Você tera 3 opções de escolha para criar o seu Dashboard.

## Alertas Baseados em Métricas

**Implementação de alertas utilizando o Alertmanager**

Para utilizar o alertmanager, destrua os containers e suba eles novamente com esses arquivos novos.

Ajuste o arquivo `prometheus.yml`

```

global:
  scrape_interval: 15s

  evaluation_interval: 10s
rule_files:
  - rules.yml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - localhost:9093

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: simple_server
    static_configs:
      - targets: ["localhost:8090"]
  - job_name: node-exporter
    static_configs:
      - targets: ["localhost:9100"]

```

**Criação do arquivo `alertmanager.yml`**

```

global:
  resolve_timeout: 5m
route:
  receiver: webhook_receiver
receivers:
    - name: webhook_receiver
      webhook_configs:
        - url: '<INSERT-YOUR-WEBHOOK>'-> 
          send_resolved: false

```





Nesse laboratório eu utilizei o webhook como receptor. Para criar o seu webhook acesse webhook.site e copie a primeira opção 'Your unique URL'. Cada URL é única.


  **Criação do arquivo `rules.yml`**

  ```
groups:
 - name: Count greater than 5
   rules:
   - alert: CountGreaterThan5
     expr: ping_request_count > 5
     for: 10s

```
    
Acesse o site do Prometheus para verificar se está tudo up. Acesse a métrica `ping_request_count`

Clique em Alerts e verá o alerta configurado. 

Clique em Status, Roles e verá a regra configurada.

Acesse a aplicação e passe de 5 vezes para gerar o alerta

Vá para o webhook.site e veja se já disparou o alerta.

Com esses passos, você estará pronto para explorar e entender melhor o Prometheus, Grafana e Alertmanager no contexto deste laboratório. Boas explorações!












  









         


