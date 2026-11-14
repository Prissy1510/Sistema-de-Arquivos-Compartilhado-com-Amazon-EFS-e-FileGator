# ☁️ Arquitetura de Armazenamento Compartilhado Multi-AZ na AWS (EFS + EC2 + SSM + FileGator)

Projeto prático de infraestrutura em nuvem desenvolvido na **AWS (Amazon Web Services)** com foco em **alta disponibilidade**, **desacoplamento de armazenamento** e **segurança em camadas**.

A solução implementa duas instâncias **Amazon EC2** distribuídas em diferentes Zonas de Disponibilidade (*Multi-AZ*), servindo a aplicação web **FileGator** (gerenciador de arquivos open-source baseado em PHP) integrada a um sistema de arquivos compartilhado **Amazon EFS (Elastic File System)**, garantindo persistência e sincronização de dados em tempo real.

---

## 🏗️ Arquitetura da Solução

![Diagrama de Arquitetura AWS EFS Multi-AZ](/imagem/Captura%20de%20Tela%20(2805).png)

* **Computação:** 2 instâncias Amazon EC2 (`t2.micro` com Amazon Linux 2023) distribuídas em sub-redes públicas distintas (`us-east-2a` e `us-east-2b`).
* **Armazenamento Compartilhado:** Amazon EFS (modo Regional) montado simultaneamente em `/mnt/efs` nas duas instâncias via NFSv4.1.
* **Segurança em Camadas (Zero Trust / Least Privilege):**
  * **Encadeamento de Security Groups:** O Security Group do EFS (`SG-EFS`) aceita tráfego na porta `2049 (NFS)` exclusivamente proveniente do Security Group das instâncias web (`SG-EC2`), bloqueando qualquer outra origem.
  * **Acesso Sem Chaves SSH:** Gerenciamento remoto realizado via **AWS Systems Manager (SSM) Session Manager** utilizando IAM Role (`AmazonSSMManagedInstanceCore`), eliminando a necessidade de pares de chaves `.pem` e mantendo a porta `22 (SSH)` fechada para a internet.
* **Aplicação Web:** Pilha LAMP (Linux, Apache, PHP) executando o **FileGator**, com o diretório de armazenamento redirecionado para o EFS por meio de um **Link Simbólico (Symlink)**.

---

## 🛠️ Etapas de Implementação na AWS

### 1. Segurança e Controle de Acesso (Security Groups & IAM)

* **`SG-EC2-PriscillaCorrea`:** Liberação de tráfego HTTP (`porta 80`) para `0.0.0.0/0` permitindo o acesso web à aplicação FileGator.

![Security Group EC2](/imagem/Captura%20de%20Tela%20(2808).png)

* **`SG-EFS-PriscillaCorrea`:** Liberação de tráfego NFS (`porta 2049`) com origem restrita ao ID do grupo `SG-EC2-PriscillaCorrea` (*Security Group Chaining*).

![Security Group EFS](/imagem/Captura%20de%20Tela%20(2809).png)

* **`Role-EC2-SSM-PriscillaCorrea`:** Criação de IAM Role vinculada às instâncias EC2 com a política gerenciada `AmazonSSMManagedInstanceCore` para acesso seguro via terminal no navegador sem exposição da porta 22.

![IAM Role SSM](/imagem/Captura%20de%20Tela%20(2810).png)

---

### 2. Provisionamento do Amazon EFS e Instâncias EC2

* **Amazon EFS (`EFS-PriscillaCorrea`):** Sistema de arquivos configurado no modo **Regional** com *Mount Targets* nas sub-redes de cada Zona de Disponibilidade associados ao `SG-EFS-PriscillaCorrea`.

![Lista de Sistemas de Arquivos Amazon EFS](/imagem/Captura%20de%20Tela%20(2811).png)

![Detalhes do Sistema de Arquivos Amazon EFS](/imagem/Captura%20de%20Tela%20(2812).png)

* **Instâncias Amazon EC2 (`EC2-Web-01` e `EC2-Web-02`):** Lançadas em sub-redes públicas distintas (`us-east-2a` e `us-east-2b`) sem par de chaves SSH, com a IAM Role anexada e ponto de montagem configurado em `/mnt/efs`.

![Instâncias EC2 em Execução](/imagem/Captura%20de%20Tela%20(2814).png)

---

## 💻 Configuração do Servidor Linux (Comandos Executados)

Após conectar à instância via **AWS SSM Session Manager**, foram executados os seguintes passos no terminal Linux:

### 1. Validação da Montagem do EFS e Criação do Diretório Compartilhado
```bash
# Validar ponto de montagem do EFS
df -h | grep efs

# Criar diretório compartilhado e ajustar permissões
sudo mkdir -p /mnt/efs/arquivos
sudo chmod 777 /mnt/efs/arquivos
```

### 2. Instalação da Pilha Web (Apache + PHP)
```bash
sudo dnf update -y
sudo dnf install -y httpd php php-cli php-fpm php-mbstring php-xml php-json php-zip unzip wget
sudo systemctl enable --now httpd
```

### 3. Implantação do FileGator e Integração com EFS via Link Simbólico
```bash
# Download e extração do FileGator
cd /var/www
sudo wget [https://github.com/filegator/static/raw/master/builds/filegator_latest.zip](https://github.com/filegator/static/raw/master/builds/filegator_latest.zip)
sudo unzip filegator_latest.zip
sudo rm filegator_latest.zip

# Redirecionamento do repositório local para o Amazon EFS via Symlink
sudo rm -rf /var/www/filegator/repository
sudo ln -s /mnt/efs/arquivos /var/www/filegator/repository

# Ajuste de proprietário e permissões POSIX
sudo chown -R apache:apache /var/www/filegator
sudo chmod -R 775 /var/www/filegator/private
sudo chmod 777 /mnt/efs/arquivos
```

### 4. Configuração do VirtualHost no Apache
Criação do arquivo `/etc/httpd/conf.d/filegator.conf`:
```apache
<VirtualHost *:80>
    DocumentRoot /var/www/filegator/dist

    <Directory /var/www/filegator/dist>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/filegator_error.log
    CustomLog /var/log/httpd/filegator_access.log combined
</VirtualHost>
```

Reinicialização do servidor web:
```bash
sudo systemctl restart httpd
```

---

## 🔍 Troubleshooting Realizado na Prática

Durante a validação da montagem automática do EFS no boot (`cloud-init`), o utilitário `amazon-efs-utils` apresentou falha de resolução DNS (`Falha ao resolver fs-0083476fe95c68a6c.efs.us-east-2.amazonaws.com`) devido à configuração de *DNS Hostnames* da VPC do ambiente de laboratório.

* **Solução aplicada:** Identificação do endereço IP privado do *Mount Target* correspondente à Zona de Disponibilidade da instância na aba *Network* do EFS e montagem direta via protocolo `nfs4`:
```bash
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <IP_MOUNT_TARGET>:/ /mnt/efs
```

---

## ✅ Validação da Aplicação Web (FileGator)

### 1. Gerenciamento e Criação de Usuários
Acesso ao painel administrativo do FileGator via IP público da instância EC2 e criação do usuário dedicado com permissões de leitura, escrita e upload:

![Gerenciamento de Usuários no FileGator](/imagem/Captura%20de%20Tela%20(2829).png)

### 2. Upload e Persistência de Arquivos
Validação do envio de arquivos pela interface web autenticada com o usuário criado:

![Upload de Arquivos no FileGator](/imagem/Captura%20de%20Tela%20(2831).png)

---

## 🧹 Encerramento e Boas Práticas de FinOps

Após a validação da arquitetura e dos testes de persistência, todos os recursos provisionados foram encerrados para evitar custos desnecessários na conta AWS:
1. Encerramento (*Terminate*) das instâncias `EC2-Web-01` e `EC2-Web-02`.
2. Exclusão do sistema de arquivos `EFS-PriscillaCorrea` e seus *Mount Targets*.
3. Remoção dos Security Groups (`SG-EC2-PriscillaCorrea` e `SG-EFS-PriscillaCorrea`) e da IAM Role (`Role-EC2-SSM-PriscillaCorrea`).