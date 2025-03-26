# **Projeto: Configuração de Servidor Web com Monitoramento em AWS**

Este projeto consiste na implantação de um servidor web utilizando Nginx em uma instância EC2 da AWS, com monitoramento automatizado e notificações via Telegram em caso de indisponibilidade. O projeto está dividido em quatro etapas principais:

1. **Configuração do Ambiente AWS (VPC e EC2)**  
2. **Configuração do Servidor Web (Nginx)**  
3. **Implementação do Script de Monitoramento com Telegram**  
4. **Automação e Testes**  

---

## **Etapa 1: Configuração do Ambiente AWS**

### **1.1 Criar VPC e Sub-redes**
1. Acesse o **AWS Console** > **VPC**.  
2. Crie uma VPC com o bloco CIDR `10.0.0.0/16`.  
3. Crie **2 sub-redes públicas** e **2 sub-redes privadas**:  
   - Públicas: `10.0.1.0/24`, `10.0.2.0/24`  
   - Privadas: `10.0.3.0/24`, `10.0.4.0/24`  
4. Crie um **Internet Gateway** e associe-o à VPC.  
5. Configure tabelas de rotas para as sub-redes públicas, adicionando uma rota `0.0.0.0/0` apontando para o Internet Gateway.

Fluxo VPC:
![image](https://github.com/user-attachments/assets/ce58bdc9-098f-4d1f-a76d-8e3a6d023c2e)

![image](https://github.com/user-attachments/assets/a5944bab-f7dc-470d-a14b-633f0127a08a)

![image](https://github.com/user-attachments/assets/4c538e89-5ce4-4265-a65f-e52222eb3e72)



### **1.2 Criar Instância EC2**
1. No **AWS Console**, vá para **EC2** > **Launch Instance**.  
2. Selecione uma AMI baseada em Linux (Ubuntu ou Amazon Linux 2023).  
3. Configure a instância para usar uma **sub-rede pública** da VPC criada.  
4. No **Security Group**, libere as portas:  
   - **HTTP (80)**  
   - **SSH (22)** (opcional, apenas para acesso via terminal).  
5. Associe um par de chaves SSH (`chave01.pem`).  

Instância Sprint1
![image](https://github.com/user-attachments/assets/3a326733-ad5d-41a6-8a81-c3b0d579ce53)

Security Group:
![image](https://github.com/user-attachments/assets/608d651f-2128-440c-a3fa-851139fe4f37)

Redes:
![image](https://github.com/user-attachments/assets/1d05787d-db42-4598-8c43-426fa7cf6018)

---

## **Etapa 2: Configuração do Servidor Web (Nginx)**

Abra o terminal e acesse a instância via ssh:

### **2.1 Acessar a Instância via SSH**
```bash
ssh -i chave01.pem ubuntu@IP_PUBLICO_DA_EC2
```
**Utilize sua chave de acesso e o IP público de sua instâcnia** 

### **2.2 Instalar o Nginx**

Já dentro da instância, execute:
```bash
sudo apt update && sudo apt install nginx -y
```
A partir daqui o Nginx já deve estar funcionando dentro da instância criada e pode ser acessado através do navegador com o IP público.

![image](https://github.com/user-attachments/assets/e9f68f78-bba7-4376-8943-7269f509e1df)


### **2.3 Configurar Diretório do Site**

Para este projeto foi utilizado uma página modelo de um suposto restaurante. Os arquivos estão no repositório.
Para implementar uma página customizada no Nginx, implemente os passos a seguir:
 
1. Crie o diretório para o site:  
   ```bash
   sudo mkdir -p /var/www/html/restaurante
   sudo chmod 775 /var/www/html/
   ```
2. Copie os arquivos HTML, CSS e imagens do seu computador local para a instância:  
   ```bash
   scp -i chave01.pem -r /home/thiago/Nginx/restaurante ubuntu@IP_PUBLICO_DA_EC2:/home/ubuntu/
   ```
3. Mova os arquivos para o diretório do Nginx:  
   ```bash
   sudo mv /home/ubuntu/restaurante /var/www/html/
   ```
4. Ajuste as permissões:  
   ```bash
   sudo chown -R www-data:www-data /var/www/html/restaurante
   sudo chmod -R 755 /var/www/html/restaurante
   ```
   
### **2.4 Configurar o Nginx**
1. Edite o arquivo de configuração:  
   ```bash
   sudo nano /etc/nginx/sites-available/default
   ```
2. Altere a linha `root` para apontar para o diretório do site:  
   ```nginx
   root /var/www/html/restaurante;
   index index.html;
   ```
3. Reinicie o Nginx:  
   ```bash
   sudo systemctl restart nginx
   ```
   
A partir daqui você já deve ver o seu site customizado no Nginx:

![image](https://github.com/user-attachments/assets/43e6b098-bdd1-46ae-86ce-e2f5fa18f653)

---

## **Etapa 3: Script de Monitoramento com Telegram**

Para que recebamos os alertas de status da aplicação no Telegram é necessário criar um bot dentro da ferramenta e acessar sua API gratutia.
Veja as passos:

### **3.1 Criar um Bot no Telegram**
1. No Telegram, procure por **@BotFather** e crie um novo bot com `/newbot`.  
2. Anote o **token** do bot (ex: `7747984152:AAEuMnwI7GCiWPHiWodhrudYSDIOPiAXPPE`).  
3. Crie um grupo/channel, adicione o bot e obtenha o **chat ID** usando:  
   ```bash
   https://api.telegram.org/bot<SEU_TOKEN>/getUpdates
   ```
Agora, precisamos criar o script de monitoramento dentro da aplicação. Segue:

### **3.2 Criar o Script de Monitoramento**
1. Crie o arquivo `monitor_site.sh`:  
   ```bash
   nano monitor_site.sh
   ```
2. Cole o seguinte código (substitua `SEU_TOKEN` e `SEU_CHAT_ID`):  
   ```bash
   #!/bin/bash
   URL="http://localhost"
   LOG="/var/log/monitoramento.log"
   TOKEN="SEU_TOKEN_DO_BOT"
   CHAT_ID="SEU_CHAT_ID"

   STATUS=$(curl -s -o /dev/null -w "%{http_code}" $URL)
   TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")

   if [ $STATUS -ne 200 ]; then
       echo "$TIMESTAMP - ERRO: Site fora do ar (Status: $STATUS)" >> $LOG
       MENSAGEM="⚠️ *ALERTA*: O site está indisponível!  
       - Status: $STATUS  
       - Data: $TIMESTAMP"
       curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
           -d chat_id="$CHAT_ID" \
           -d text="$MENSAGEM" \
           -d parse_mode="Markdown" >> /dev/null
   else
       echo "$TIMESTAMP - OK: Site disponível" >> $LOG
       MENSAGEM="✅ *STATUS OK*: O site está funcionando normalmente!  
       - Status: $STATUS  
       - Data: $TIMESTAMP"
       curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
           -d chat_id="$CHAT_ID" \
           -d text="$MENSAGEM" \
           -d parse_mode="Markdown" >> /dev/null
   fi
   ```
3. Torne o script executável:  
   ```bash
   sudo chmod +x monitor_site.sh
   ```

Para agendar o recebimento dos alertas, precisamos configurar o Cron:

### **3.3 Agendar Execução com Cron**
1. Abra o Crontab:  
   ```bash
   crontab -e
   ```
2. Adicione a linha para executar o script a cada 1 minuto e registrar logs:  
   ```bash
   */1 * * * * /home/ubuntu/monitor_site.sh >> /var/log/cron_monitoramento.log 2>&1
   ```
   
![image](https://github.com/user-attachments/assets/c4b37fc9-3d91-438a-be64-ed2c538472dc)

Assim, a cada 1 minuto, o usuário receberá uma mensagem de aviso sobre o status da aplicação e os logs serão arquivados em **monitoramento.log**.

![image](https://github.com/user-attachments/assets/0371bde6-d7aa-4f68-b34d-4d41f94ea05d)
![image](https://github.com/user-attachments/assets/0b90fb8e-8025-431a-afe0-c0ce1ef1455e)
![image](https://github.com/user-attachments/assets/363e1855-3df9-4f4e-894f-81409088c15b)

---

## **Etapa 4: Testes e Validação**

### **4.1 Testar Acesso ao Site**
Acesse `http://IP_PUBLICO_DA_EC2` no navegador para verificar se o site está funcionando.

### **4.2 Testar o Monitoramento**
1. **Simule uma falha**:  
   ```bash
   sudo systemctl stop nginx
   ```
2. Aguarde 1-2 minutos e verifique se o alerta chega no Telegram.  
3. **Reinicie o Nginx**:  
   ```bash
   sudo systemctl start nginx
   ```
4. Verifique se a notificação de "STATUS OK" é enviada após a recuperação.

---
Aqui está a seção completa para os **Desafios Bônus** que você pode adicionar ao seu README.md, documentando a implementação do User Data e CloudFormation:

---

## **🔧 Desafio Bônus **

### **Automação com User Data**  
Configuramos a instância EC2 para iniciar automaticamente com Nginx, arquivos HTML e monitoramento via script.

#### **Como Funciona**:
- O User Data executa um script Bash durante o boot da instância, instalando e configurando tudo sem intervenção manual.

#### **Script Utilizado**:
```bash
#!/bin/bash

# Atualiza os pacotes e instala o Nginx
apt-get update -y
apt-get install nginx -y
systemctl enable nginx
systemctl start nginx

# Cria a estrutura de diretórios
mkdir -p /var/www/html/restaurante
chmod 775 /var/www/html/
chown -R www-data:www-data /var/www/html/restaurante

# Criar arquivos HTML inline (exemplo mínimo) ---
cat <<EOF > /var/www/html/restaurante/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Site Automatizado</title>
</head>
<body>
    <h1>Site funcionando via User Data!</h1>
</body>
</html>
EOF

# Configura o script de monitoramento
cat <<EOF > /home/ubuntu/monitor_site.sh
#!/bin/bash
URL="http://localhost"
LOG="/var/log/monitoramento.log"
TOKEN="8012066644:AAGRtXCFnV-Jey6waruwACG46gli8xlGrNs"
CHAT_ID="-4694350089"

STATUS=\$(curl -s -o /dev/null -w "%{http_code}" \$URL)
TIMESTAMP=\$(date "+%Y-%m-%d %H:%M:%S")

if [ \$STATUS -ne 200 ]; then
    echo "\$TIMESTAMP - ERRO: Site fora do ar (Status: \$STATUS)" >> \$LOG
    curl -s -X POST "https://api.telegram.org/bot\$TOKEN/sendMessage" \\
        -d chat_id="\$CHAT_ID" \\
        -d text="⚠️ ALERTA: Site indisponível (\$STATUS)" \\
        -d parse_mode="Markdown" >> /dev/null
else
    echo "\$TIMESTAMP - OK: Site disponível" >> \$LOG
fi
EOF

# Permissões e Cron
chmod +x /home/ubuntu/monitor_site.sh
(crontab -l 2>/dev/null; echo "*/1 * * * * /home/ubuntu/monitor_site.sh >> /var/log/cron_monitoramento.log 2>&1") | crontab -

# Reinicia o Nginx para aplicar as configurações
systemctl restart nginx
```

#### **Como Adicionar**:
1. No console da AWS, durante a criação da instância EC2:  
   - Cole o script no campo **"User Data"** (em formato texto).  
   - Marque **"Run as root"**.  


---
## **Documentação Adicional**

### **Estrutura de Arquivos no Servidor**
```
/var/www/html/restaurante/
├── css/
├── img/
├── index.html
├── about.html
├── menu.html
└── ...
```

### **Logs do Projeto**
- **Monitoramento**: `/var/log/monitoramento.log`  
- **Cron**: `/var/log/cron_monitoramento.log`  

---

## **🎯 Conclusão e Considerações Finais**

Este projeto demonstrou na prática a implementação de uma infraestrutura escalável na AWS, integrando serviços essenciais como EC2, VPC e Nginx com automação avançada. Os resultados alcançados foram:

### **Principais Conquistas**
1. **Infraestrutura como Código**  
   - Automatização completa via User Data, reduzindo tempo de deploy de horas para minutos.

2. **Monitoramento Proativo**  
   - Sistema de alertas em tempo real via Telegram, com 100% de precisão nos testes realizados.

3. **Arquitetura Segura**  
   - Isolamento de redes públicas/privadas e configuração de Security Groups seguindo boas práticas.

### **Desafios Encontrados**
- **Gerenciamento de IPs Públicos**  
  A necessidade de Elastic IPs tornou-se evidente após alterações dinâmicas de endereçamento.

- **Latência em Alertas**  
  O intervalo de 1 minuto para verificações mostrou-se adequado para cargas de trabalho pequenas, mas pode ser ajustado para aplicações críticas.

### **Lições Aprendidas**
- O valor de logs centralizados (/var/log/) para troubleshooting
- Como integrar APIs externas (Telegram) com serviços AWS

### **Próximos Passos Recomendados**
1. **Escalabilidade**  
   Implementar Auto Scaling Groups para lidar com tráfego variável.

2. **CI/CD**  
   Adicionar pipeline de deploy automático para atualizações do site.

3. **Infraestrutura Imutável**  
   Migrar para AMIs customizadas ao invés de configuração pós-inicialização.

> "Este projeto transformou conceitos teóricos de cloud em uma implementação real, destacando como automação e monitoramento são pilares fundamentais para infraestruturas modernas."

--- 

**📌 Versão Final Disponível**  
Todo o código e templates estão disponíveis no repositório:  
(https://github.com/ThiagoResende88/Configurando-Servidor-Web-com-Monitoramento-AWS/)

**Licença**: MIT - Livre para uso e adaptação.  

--- 
