# Projeto: Servidor Web com Monitoramento e Alertas na AWS

Este repositório contém a documentação e os scripts para o projeto final da disciplina de Linux do Programa de Bolsas DevSecOps. O objetivo é configurar um ambiente web funcional na nuvem da AWS, monitorar sua disponibilidade e receber alertas automáticos em caso de falha.

## Tecnologias Utilizadas

* **AWS (Amazon Web Services):**
    * **VPC:** Para criação da rede privada na nuvem.
    * **EC2:** Como servidor virtual para hospedar a aplicação.
    * **Security Groups:** Para atuar como nosso firewall.
* **Linux (Ubuntu/Amazon Linux):** Sistema operacional do servidor.
* **Nginx:** Servidor web para hospedar a página HTML.
* **Bash Scripting:** Para criar o script de monitoramento.
* **Cron:** Para automatizar a execução do script.
* **Discord/Telegram:** Como plataforma para recebimento de alertas.

---

## Etapa 1: Configuração do Ambiente AWS

A primeira etapa consiste em criar a infraestrutura de rede e o servidor na AWS.

1.  **Criação da VPC:**
    * Acesse o console da AWS e navegue até o serviço de VPC.
    * Utilize o assistente "VPC e mais" para criar uma nova VPC.
    * Configure a VPC com 2 sub-redes públicas e 2 sub-redes privadas.
    

2.  **Criação da Instância EC2:**
    * No console do EC2, lance uma nova instância.
    * **TAGS:** Coloque as tags `necessárias` de acordo com o que você precisa.
    * **AMI:** `Ubuntu`.
    * **Tipo de Instância:** `t2.micro` (elegível para o nível gratuito).
    * **VPC/Sub-rede:** Associe a instância à VPC criada e a uma das **sub-redes públicas**.
    * **Atribuir endereço publico automaticamente:** `Habilitado`.
    * **Par de Chaves:** Crie e faça o download de um novo par de chaves (`.pem`) para garantir o acesso seguro.
    * **Security Group:** Crie um novo grupo de segurança permitindo tráfego de entrada nas seguintes portas:
        * `HTTP (porta 80)` - Origem: `Anywhere-IPv4 (0.0.0.0/0)`
        * `SSH (porta 22)` - Origem: `Meu IP` (Recomendado por segurança)
        * `Obs:Crie o segurity group antes de criar a EC2` 

3.  **Acesso via SSH:**
    * Com a instância em execução, utilize o endereço IP público e o arquivo `.pem` para se conectar via SSH.
    ```bash
    # Primeiro, ajuste as permissões da chave
    chmod 400 sua-chave.pem

    # Conecte-se
    ssh -i sua-chave.pem ubuntu@SEU_IP_PUBLICO
    ```

---

## Etapa 2: Configuração do Servidor Web (Nginx)

Com o acesso ao servidor, o próximo passo é instalar e configurar o Nginx.

1.  **Instalação do Nginx:**
    ```bash
    # Para Ubuntu
    sudo apt update 
    sudo apt install nginx
    ```

2.  **Habilitação do Serviço:**
    * Inicie o serviço e configure-o para iniciar automaticamente com o sistema.
    ```bash
    sudo service nginx start
    sudo systemctl enable nginx 
    sudo service nginx status -> Status deve aparecer como running.
    ```

3.  **Criação da Página HTML:**
    * Crie uma página `index.html` simples para ser exibida. O caminho do arquivo varia com o sistema operacional:
        * **Ubuntu:** `/var/www/html/index.html`
    ```html
    <!DOCTYPE html>
    <html lang="pt-br">
    <head>
        <meta charset="UTF-8">
        <title>Projeto Linux</title>
    </head>
    <body>
        <h1>Meu Servidor Nginx está Funcionando!</h1>
        <p>Este é o projeto de configuração de servidor web com monitoramento.</p>
    </body>
    </html>
    ```
    * Use uma página existente no linux:
        * No seu terminal sem ser o que está conectado na maquina na nuvem:
        * **Ubuntu:** scp -i [sua-chave.pem] [arquivo-local-para-enviar] [usuario]@[ip-publico]:[caminho-remoto-onde-salvar]
        * `Obs:podem ser enviados tudo que você precisa index.html,style.css,funtion.js e imagens`

4.  **Teste:** Acesse `http://SEU_IP_PUBLICO` em um navegador para verificar se a página está sendo exibida.

---

## Etapa 3: Monitoramento e Notificações

Nesta etapa, criamos o script que verifica a saúde do site e o automatizamos com o `cron`.

1.  **Criação do Script de Monitoramento:**
    * Crie o arquivo de script no servidor.
    ```bash
    sudo nano /usr/local/bin/monitor.sh
    ```
    * Abaixo está o exemplo de script para notificações via **Discord**. Adapte a variável `WEBHOOK_URL` com a sua.

    ```bash
    #!/bin/bash

    SITE_URL="http://SEU_IP_PUBLICO"
    LOG_FILE="/var/log/monitor.log"
    WEBHOOK_URL="SUA_URL_DE_WEBHOOK_AQUI"

    HTTP_STATUS=$(curl -o /dev/null -s -w "%{http_code}" $SITE_URL)

    notificacao(){
        MESSAGE=$1
        TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
        echo "[$TIMESTAMP] $MESSAGE" >> $LOG_FILE
        curl -H "Content-Type: application/json" -X POST -d "{\"content\": \"${MESSAGE}\"}" "${WEBHOOK_URL}"
    }

    if [ "$HTTP_STATUS" -eq 200 ]; then
        TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
        echo "[$TIMESTAMP] Site está online. Status: $HTTP_STATUS" >> $LOG_FILE
    else
        notificacao "Site está fora do ar! Status retornado: $HTTP_STATUS"
    fi
    ```
    * Torne o script executável:
    ```bash
    sudo chmod +x /usr/local/bin/monitor.sh
    ```

2.  **Automatização com Cron:**
    * Agende a execução do script para rodar a cada minuto.
    ```bash
    # Abra o editor do crontab do usuário root
    sudo crontab -e
    ```
    * Adicione a seguinte linha no final do arquivo e salve:
    ```
    * * * * * /usr/local/bin/monitor.sh > /dev/null 2>&1
    ```

---

## Etapa 4: Testes e Validação

Para garantir que a solução funciona como esperado:

1.  **Verifique o Funcionamento Normal:**
    * Acesse o site pelo navegador.
    * Observe o arquivo de log para ver as mensagens de "Site está online".
        ```bash
        tail -f /var/log/monitor.log
        ```

2.  **Simule uma Falha:**
    * Pare o serviço do Nginx para simular uma queda.
        ```bash
        sudo service nginx stop
        ```
    * Aguarde até um minuto e verifique se a notificação de alerta chegou no seu canal do Discord.
    * Verifique o arquivo de log para ver a mensagem de erro.

3.  **Restaure o Serviço:**
    ```bash
    sudo service start nginx
    ```

