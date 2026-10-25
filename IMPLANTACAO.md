# AWS Certificate Manager SSL/TLS na Prática: Implantação (ACM + ALB)

> Guia prático de implantação: traz o passo a passo manual dos dois cenários, com os cuidados, os testes e os aprendizados essenciais.
>
> ✅ **Implantação realizada e validada** pelo Console AWS. As evidências de execução estão em [`TESTES.md`](TESTES.md).
>
> Execução 100% manual pelo Console AWS. Região de referência: `<REGIAO>`. O certificado do ACM deve ficar na **mesma região do ALB**.

## Variáveis do laboratório

> Os valores abaixo são **variáveis (placeholders)**. Substitua cada `<VARIAVEL>` pelos seus próprios nomes/valores ao executar. Nenhum identificador real da sua conta ou console deve ser versionado.

| Variável | Descrição | Exemplo de valor |
|----------|-----------|------------------|
| `<REGIAO>` | Região AWS usada no laboratório | `us-east-1` |
| `<SG_ALB>` | Security Group do ALB | `alb-lab01-sg` |
| `<SG_EC2>` | Security Group da EC2 | `ec2-lab01-sg` |
| `<EC2_APP>` | Instância EC2 da aplicação | `ec2-app-lab01` |
| `<TG_1>` | Target Group do Cenário 1 | `meu-tg-lab01` |
| `<ALB_1>` | Application Load Balancer do Cenário 1 | `meu-alb-lab01` |
| `<TG_2>` | Target Group do Cenário 2 | `meu-tg-lab01-2` |
| `<ALB_2>` | Application Load Balancer do Cenário 2 | `meu-alb-lab01-2` |
| `<DOMINIO_PUBLICO>` | Domínio do certificado público (Cenário 1) | `app.exemplo.com` |
| `<DOMINIO_AUTOASSINADO>` | Domínio do certificado autoassinado (Cenário 2) | `app2.exemplo.com` |
| `<DOMINIO_WILDCARD>` | Wildcard que cobre o domínio público | `*.exemplo.com` |
| `<ZONA_DNS>` | Hosted Zone (Route 53) do domínio | `exemplo.com` |
| `<DNS_ALB_2>` | DNS público gerado para o `<ALB_2>` | `<alb-2>.<REGIAO>.elb.amazonaws.com` |
| `<DATA_EXPIRACAO>` | Data `notAfter` do certificado importado | `DD/MM/AAAA` |

> **Antes de começar:** utilize a mesma Região `<REGIAO>` para ACM e ALBs; os ALBs exigem ao menos duas subnets em AZs diferentes. O certificado público do ACM em si não gera cobrança, mas **cada ALB** tem cobrança por hora e capacidade, e a EC2 também pode gerar custos. Reutilizar uma EC2 reduz custos; os dois ALBs não são gratuitos.

---

## Cenário 1 — Certificado público do ACM + ALB

1. **EC2 > Security Groups > Criar grupo de segurança.** Nome `<SG_ALB>`, VPC do laboratório. Entrada: HTTPS 443 e HTTP 80 de `0.0.0.0/0` (a 80 será usada apenas para redirecionamento HTTP → HTTPS).

   ![Security Group do ALB com regras de entrada 443 e 80](imagens/imagem%20(3).png)

2. **EC2 > Security Groups > Criar grupo de segurança.** Nome `<SG_EC2>`, mesma VPC. Entrada: HTTP 80 com origem = grupo `<SG_ALB>` (referência de SG, não CIDR).
3. **EC2 > Instâncias > Executar instâncias.** Configurar a instância:
   - **Nome:** `<EC2_APP>`.
   - **AMI:** Amazon Linux (última versão).
   - **Tipo:** `t3.micro`.
   - **Par de chaves (login):** selecionar **Continuar sem par de chaves** somente se já houver acesso viável por EC2 Instance Connect ou Session Manager (com requisitos de rede/IAM atendidos).
   - **Configurações de rede > Editar:** VPC do laboratório; **Sub-rede** = uma sub-rede **pública**; **Atribuir IP público automaticamente** = **Habilitar**; **Grupo de segurança** = existente `<SG_EC2>`.
   - (Reutilize uma EC2 de outro lab se já existir na mesma VPC/região.)

   ![Instância EC2 da aplicação em execução](imagens/imagem%20(2).png)

4. **EC2 > a instância > User data** (em Configurações avançadas, na criação) **ou EC2 Instance Connect** (após iniciar). Publicar uma página HTTP apresentável na porta 80:

   ```bash
   #!/bin/bash
   dnf install -y httpd || yum install -y httpd
   systemctl enable --now httpd
   TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
   AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
   IID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
   cat > /var/www/html/index.html <<HTML
   <!doctype html>
   <html lang="pt-br">
   <head>
     <meta charset="utf-8">
     <meta name="viewport" content="width=device-width, initial-scale=1">
     <title>Aplicacao Segura HTTPS</title>
     <style>
       :root { color-scheme: dark; }
       * { box-sizing: border-box; margin: 0; padding: 0; }
       body { font-family: "Segoe UI", system-ui, sans-serif; min-height: 100vh;
         display: flex; align-items: center; justify-content: center;
         background: linear-gradient(135deg, #0f2027, #203a43, #2c5364); color: #f5f7fa; }
       .card { background: rgba(255,255,255,0.06); backdrop-filter: blur(10px);
         border: 1px solid rgba(255,255,255,0.12); border-radius: 18px;
         padding: 48px 56px; text-align: center; max-width: 560px;
         box-shadow: 0 20px 50px rgba(0,0,0,0.4); }
       .badge { display: inline-block; background: #2ecc71; color: #06301a;
         font-weight: 700; font-size: 13px; letter-spacing: .5px;
         padding: 6px 14px; border-radius: 999px; margin-bottom: 20px; }
       h1 { font-size: 30px; margin-bottom: 12px; }
       p { color: #c7d2da; line-height: 1.6; margin-bottom: 24px; }
       .meta { display: flex; gap: 14px; justify-content: center; flex-wrap: wrap; }
       .meta span { background: rgba(0,0,0,0.25); border-radius: 10px;
         padding: 10px 16px; font-size: 13px; color: #9fe6c0; }
       footer { margin-top: 28px; font-size: 12px; color: #7f8e99; }
     </style>
   </head>
   <body>
     <div class="card">
       <span class="badge">CONEXAO SEGURA &bull; TLS</span>
       <h1>Aplicacao disponivel por HTTPS</h1>
       <p>Esta pagina e servida por HTTP na instancia EC2 e exposta com
       terminacao TLS no Application Load Balancer, usando um certificado do ACM.</p>
       <div class="meta">
         <span>Instancia: $IID</span>
         <span>Zona: $AZ</span>
       </div>
       <footer>Servidor web Apache httpd</footer>
     </div>
   </body>
   </html>
   HTML
   ```

5. **EC2 > Grupos de destino > Criar grupo de destino.** Tipo Instâncias, nome `<TG_1>`, HTTP 80, health check em `/`.
6. **Target Group `<TG_1>` > Destinos > Registrar destinos.** Selecionar `<EC2_APP>`, porta 80, registrar.
7. **Target Group `<TG_1>` > Destinos.** Aguardar o destino ficar `healthy` (até 300s).

   ![Target Group do Cenário 1 com a EC2 em estado Healthy](imagens/imagem%20(5).png)

   ![Detalhes do Target Group do Cenário 1 com 1 destino íntegro](imagens/imagem%20(8).png)

8. **EC2 > Load Balancers > Criar load balancer > Application Load Balancer.** Nome `<ALB_1>`, esquema internet-facing, IPv4, VPC do lab, ≥ 2 subnets em 2 AZs, SG `<SG_ALB>`. Em **Listeners e roteamento**, deixe o listener inicial como **HTTP:80** encaminhando para o grupo de destino `<TG_1>` (o listener HTTPS:443 é criado no passo 11, após o certificado ficar `Issued`). Criar load balancer.

   ![ALB do Cenário 1 ativo, com detalhes do balanceador](imagens/imagem%20(6).png)

9. **ACM (mesma região do ALB) > Solicitar certificado > Público.** Domínio `<DOMINIO_PUBLICO>` (ou certificado wildcard `<DOMINIO_WILDCARD>`, que o cobre), validação por DNS.
10. **ACM > o certificado > Criar registros no Route 53** (cria o CNAME de validação). Aguardar o status mudar para `Issued`.

   ![Lista de certificados no ACM, incluindo o certificado público e o wildcard emitidos](imagens/imagem%20(10).png)

11. **EC2 > Load Balancers > `<ALB_1>` > Listeners e regras > Adicionar listener.** Criar o listener **HTTPS:443**, ação padrão Forward para `<TG_1>` (o mesmo grupo de destino), política TLS recomendada. (O HTTP:80 do passo 8 pode ser mantido ou removido; o que faz a terminação TLS é este HTTPS:443.)
12. **No listener HTTPS:443 > Certificado SSL/TLS.** Origem Do ACM, selecionar o certificado público válido que cobre `<DOMINIO_PUBLICO>` (`Issued`). Salvar.
13. **Route 53 > zona DNS correspondente > Criar registro.** Criar `<DOMINIO_PUBLICO>` como registro **A Alias** para `<ALB_1>` (o nome relativo depende da zona hospedada: por exemplo, `app` na zona `<ZONA_DNS>`).

   ![Registros da Hosted Zone no Route 53: A Alias e CNAME de validação](imagens/imagem%20(11).png)

14. **Navegador.** Acessar `https://<DOMINIO_PUBLICO>` → HTTPS sem aviso de certificado e página HTML. Confira o emissor e o nome do certificado.

   ![Navegador acessando o domínio público com conexão segura](imagens/imagem%20(12).png)

   ![Detalhes do certificado wildcard emitido pela Amazon RSA 2048 M04](imagens/imagem%20(16).png)

> **Por que funciona?** O navegador confia na CA pública emissora. O ALB termina o TLS na 443 e encaminha HTTP 80 à EC2. **Renovação automática:** aplicável a certificados públicos do ACM elegíveis, desde que as condições de validação e uso continuem atendidas (inclusive o CNAME no DNS). O domínio pode estar em DNS externo: nesse caso publique o CNAME lá; o atalho de criação automática no Route 53 não se aplica.

---

## Cenário 2 — Certificado autoassinado importado (segundo ALB)

> Cria um **segundo ALB** (`<ALB_2>`) e um **segundo Target Group** (`<TG_2>`), **compartilhando a mesma EC2** `<EC2_APP>` e o mesmo SG `<SG_ALB>`. O Cenário 1 permanece **intacto**. Domínio usado: `<DOMINIO_AUTOASSINADO>`. **Este segundo ALB também gera custos.**

### Gerar o certificado autoassinado (escolha A ou B)

**Opção A — Windows / PowerShell:**

```powershell
winget install --id ShiningLight.OpenSSL.Light --exact
# Execute na pasta do laboratório onde deseja salvar os arquivos:
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" req -x509 -newkey rsa:2048 -sha256 -nodes -days 365 -keyout app2.key -out app2.crt -subj "/CN=<DOMINIO_AUTOASSINADO>" -addext "subjectAltName=DNS:<DOMINIO_AUTOASSINADO>"
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" x509 -in app2.crt -noout -subject -issuer -dates -ext subjectAltName
```

**Opção B — Linux / CloudShell:**

```bash
openssl req -x509 -newkey rsa:2048 -sha256 -nodes -days 365 \
  -keyout app2.key -out app2.crt \
  -subj "/CN=<DOMINIO_AUTOASSINADO>" \
  -addext "subjectAltName=DNS:<DOMINIO_AUTOASSINADO>"
openssl x509 -in app2.crt -noout -subject -issuer -dates -ext subjectAltName
```

> Os arquivos `app2.crt` (certificado público) e `app2.key` (chave privada) ficam no diretório da execução. **Nunca publique `app2.key` no GitHub; inclua-o no `.gitignore`.** No Windows, use o executável pelo caminho completo caso `openssl` não esteja no PATH.

![Geração do certificado autoassinado com OpenSSL no PowerShell](imagens/imagem%20(17).png)

![Arquivos app2.crt e app2.key gerados na pasta do laboratório](imagens/imagem%20(18).png)

![Conteúdo PEM do certificado app2.crt](imagens/imagem%20(19).png)

![Arquivo app2.key aberto (conteúdo ocultado)](imagens/imagem%20(20).png)

### Passos

15. Gerar o certificado por uma das opções acima. Arquivos: `app2.crt` e `app2.key`. O certificado é **autoassinado** (emissor = próprio domínio), portanto não é confiável automaticamente pelos navegadores.
16. **ACM (`<REGIAO>`) > Importar certificado.** Corpo = PEM de `app2.crt`; chave privada = PEM de `app2.key`; cadeia = vazia, por ser autoassinado. Importar. Confirmar **Tipo `Imported`**, **Status `Emitido`**. **Atenção:** `Emitido` significa disponível no ACM, não que o navegador confia nele.

    ![Certificado importado no ACM: Tipo Importado, Em uso Sim](imagens/imagem%20(10).png)

17. **EC2 > Grupos de destino > Criar grupo de destino.** Instâncias, nome `<TG_2>`, HTTP 80, health check `/`, mesma VPC. Registrar `<EC2_APP>` na porta 80. Aguardar `healthy`.

    ![Target Group do Cenário 2 com a EC2 em estado Healthy](imagens/imagem%20(4).png)

    ![Destinos do Target Group do Cenário 2 com a EC2 Healthy](imagens/imagem%20(9).png)

18. **EC2 > Load Balancers > Criar load balancer > Application Load Balancer.** Nome `<ALB_2>`, internet-facing, IPv4, mesma VPC, ≥ 2 subnets em 2 AZs, SG `<SG_ALB>`. Listener inicial HTTP:80 → `<TG_2>`.

    ![ALB do Cenário 2 ativo, com detalhes do segundo balanceador](imagens/imagem%20(7).png)

19. **`<ALB_2>` > Listeners e regras.** HTTP:80 = **Redirecionar para HTTPS:443 (301)**. Criar/editar HTTPS:443 = Forward para `<TG_2>`, certificado padrão **importado `<DOMINIO_AUTOASSINADO>`**, política TLS recomendada. Se ainda houver um certificado antigo em **Certificados adicionais (SNI)**, desvincule-o após conferir que não é necessário.
20. **Route 53 > zona DNS > Criar registro.** Criar **A Alias** `<DOMINIO_AUTOASSINADO>` → `<ALB_2>`. **Navegador:** acessar `https://<DOMINIO_AUTOASSINADO>` → aviso **"Sua conexão não é particular"**, erro `NET::ERR_CERT_AUTHORITY_INVALID`. Em laboratório controlado, conferir o endereço e prosseguir via **Avançadas**, se oferecido. A página deverá carregar; em **detalhes do certificado**, confirmar **emitido para** e **emitido por** `<DOMINIO_AUTOASSINADO>`. Evite testar só pelo DNS `elb.amazonaws.com`, pois isso também pode causar erro de nome incompatível.

    ![Navegador acessando o domínio autoassinado com aviso Não seguro](imagens/imagem%20(14).png)

    ![Acesso ao ALB 2 pelo DNS da AWS com aviso Não seguro](imagens/imagem%20(13).png)

21. **ACM > certificado `<DOMINIO_AUTOASSINADO>` > Detalhes.** Confirmar **Em uso: Sim**, verificar vencimento (`notAfter`; no exemplo, `<DATA_EXPIRACAO>`). A renovação é **manual**: gerar novo certificado e **Reimportar certificado** antes de expirar. No Target Group `<TG_2>`, confirmar **1 destino Healthy**; foi o resultado obtido no teste.

    ![Detalhes do certificado autoassinado: emissor igual ao titular e validade de 1 ano](imagens/imagem%20(15).png)

> **Diagnóstico (problema encontrado no teste):** o `<TG_2>` chegou a mostrar `Unhealthy — Request timed out` e depois ficou `Healthy`. Se acontecer, verifique o `httpd` ativo na EC2, a regra **HTTP:80** do `<SG_EC2>` com origem `<SG_ALB>`, o health check em `/` com HTTP 200 e os intervalos de verificação. Não atribua a mudança a uma correção específica sem evidência.
>
> **Entenda a diferença:** um certificado **importado de uma CA pública reconhecida** (ex.: DigiCert/Sectigo), com cadeia válida e domínio compatível, **não** provoca o mesmo aviso. O problema aqui é ser **autoassinado**, não ser **importado**. O domínio também pode ser gerenciado fora da AWS; é suficiente validar o domínio no DNS competente e apontá-lo corretamente.

---

## Cenário 3 — Certificados privados / AWS Private CA (conceitual)

Somente teórico. **Nenhuma AWS Private CA será provisionada** (serviço pago, ~400 USD/CA/mês). Pontos a entender:

- Certificado **público**: CA pública, confiável globalmente, para sites/APIs na internet.
- Certificado **privado**: CA privada (ex.: AWS Private CA), confiável só onde o root da CA está instalado no trust store; usado em serviços internos e mTLS.
- Casos de uso de certificado privado: endpoints internos sem domínio público roteável; mTLS entre microsserviços.

---

## Procedimento de Limpeza (ordem por dependência)

### Limpeza só do Cenário 2 (sem tocar no Cenário 1, na EC2 nem no SG compartilhado)

- **C2-L1. EC2 > Load Balancers.** Excluir `<ALB_2>` (remove os listeners junto).
- **C2-L2. EC2 > Grupos de destino.** Excluir `<TG_2>`. (A EC2 `<EC2_APP>` permanece.)
- **C2-L3. Route 53.** Remover apenas o registro **A Alias `<DOMINIO_AUTOASSINADO>`** (não o domínio nem os CNAMEs do cenário 1).
- **C2-L4. ACM.** (Opcional) Excluir o certificado **importado** `<DOMINIO_AUTOASSINADO>`, depois que não estiver mais em uso. Não excluir o wildcard público do cenário 1.

> Preservados: `<EC2_APP>`, `<SG_ALB>`, `<SG_EC2>` e todo o Cenário 1. A exclusão de um ALB **não** exclui automaticamente seus registros DNS.

### Limpeza do Cenário 1 (quando encerrar o lab inteiro)

Ordem: Listener → ALB → Target Group → registros Route 53 → EC2 → certificados ACM → Security Groups.

- **L1. EC2 > Load Balancers > `<ALB_1>` > Listeners e regras.** Excluir o listener HTTPS:443.
- **L2. EC2 > Load Balancers.** Excluir `<ALB_1>`.
- **L3. EC2 > Grupos de destino.** Excluir `<TG_1>`.
- **L4. Route 53 > sua Hosted Zone.** Excluir somente o Alias `<DOMINIO_PUBLICO>` usado pelo laboratório. Excluir o CNAME de validação **apenas se nenhum outro certificado depender dele**. Nunca apagar a Hosted Zone compartilhada.
- **L5. EC2 > Instâncias.** Encerrar `<EC2_APP>`. **Não encerre** se a EC2 for reutilizada de outro lab — apenas desassocie o `<SG_EC2>`.
- **L6. ACM.** Excluir **somente certificados exclusivos do laboratório e que não estejam em uso**; preservar certificados públicos/wildcards compartilhados com outros ambientes.
- **L7. EC2 > Security Groups.** Excluir primeiro `<SG_EC2>`, depois `<SG_ALB>`.

> Se uma exclusão falhar por dependência, remova o dependente primeiro e tente de novo. Faça a limpeza do Cenário 2 **antes** da do Cenário 1.

---

## Checklist de validação rápida

- [ ] Cenário 1: `https://<DOMINIO_PUBLICO>` abre sem aviso de certificado; ALB 1 com destino `Healthy`.
- [ ] Cenário 2: `https://<DOMINIO_AUTOASSINADO>` exibe `NET::ERR_CERT_AUTHORITY_INVALID` e a aplicação abre após prosseguir conscientemente no laboratório.
- [ ] O certificado visto em `<DOMINIO_AUTOASSINADO>` tem **subject/SAN** correspondente e emissor autoassinado.
- [ ] ACM confirma certificado importado `Em uso: Sim`; TG 2 mostra `Healthy`.
- [ ] Chave privada `app2.key` protegida e fora do repositório.
- [ ] Antes de encerrar: remover ALBs não necessários para evitar cobrança.

