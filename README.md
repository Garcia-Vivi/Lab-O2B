# Lab-O2B

## Descrição do Laboratório:

Este laboratório é baseado no exemplo disponível no site [prometheus.io](https://prometheus.io/docs/introduction/overview/).

E foi usado nesse laboratório prático sobre observabilidade ministrada pelo professor Patrick Cardoso da empresa O2B.

## Tecnologias Utilizadas:
* Linux (baseado no Ubuntu)
* WSL
* Docker
* Golang
* Prometheus
* Grafana
* Alertmanager

## Documentação do Prometheus:

### Introdução ao Prometheus
- **Instalação do Docker e Docker-compose**

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

        
- **Visualizando métricas utilizando o Grafana**

   ```docker
        version: '3'
        services:
          prometheus:
            image: prom/prometheus
            ports:
              - 9090:9090
            volumes:
              - ./prometheus.yml:/etc/prometheus/prometheus.yml
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
    # Executar o servidor
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


    2. Subir o container do promethues e visualizar a métrica 
        
        ping_request_count
    




















  









         


