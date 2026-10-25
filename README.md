# AWS Certificate Manager SSL/TLS na Prática: Público, Importado e Autoassinado com ACM e ALB

![AWS Certificate Manager - SSL/TLS na Prática](imagens/imagem%20(1).png)

> Laboratório didático de segurança na AWS. Toda a implantação é **manual, pelo Console AWS em português (Brasil)**, sem CLI, SDK ou ferramentas de automação de infraestrutura.

> ✅ **Status: implantado e validado.** Os dois cenários práticos (Cenário 1 — certificado público do ACM com renovação automática; Cenário 2 — certificado autoassinado importado) foram executados de ponta a ponta no Console AWS e validados no navegador. As evidências (capturas de tela) estão em [`TESTES.md`](TESTES.md). Todos os identificadores reais da conta foram substituídos por **variáveis** (ver a tabela **Variáveis do laboratório** em [`IMPLANTACAO.md`](IMPLANTACAO.md)).

## Objetivo

Este laboratório ensina, de forma guiada e reproduzível, como funcionam os certificados **SSL/TLS** na AWS usando o **AWS Certificate Manager (ACM)**. O foco é demonstrar o uso de certificados em um **Application Load Balancer (ALB)**, que realiza a **terminação TLS** (encerra a conexão HTTPS na porta 443) e encaminha o tráfego já descriptografado como HTTP para uma instância **EC2 Linux**.

Ao concluir o laboratório, você será capaz de:

- Diferenciar certificados **públicos** (emitidos pelo ACM), **importados** (de uma CA externa) e **privados** (emitidos por uma CA privada).
- Solicitar e validar um certificado público no ACM por **Validacao_DNS** usando o **Route 53**, até o estado `Issued`.
- Associar um certificado do ACM a um **Listener HTTPS 443** do ALB e entender a **terminação TLS**.
- Importar um certificado externo (autoassinado, apenas para demonstração) e compreender a necessidade de **renovação manual**.
- Entender, de forma conceitual, quando e por que usar uma **CA privada** (AWS Private CA) em redes corporativas.
- Reconhecer pontos importantes do ACM, como a exigência de o certificado estar na **mesma Região do ALB**, a obrigatoriedade de `us-east-1` para o **CloudFront** e a abrangência de um certificado **wildcard**.

![Imagem](imagens/imagem%20(25).png)
![Imagem](imagens/imagem%20(24).png)

## Pré-requisitos

Antes de iniciar, você precisa de:

- **Conta AWS ativa** com permissões para operar ACM, EC2, ELBv2 (ALB, Target Group, Listener), Route 53 e Security Groups. Recomenda-se um ambiente de estudo (não produção).
- **Um domínio sob seu controle** com uma **Hosted Zone** no Route 53 (necessário para a Validacao_DNS e para o registro `alias` que aponta o domínio ao ALB). Nos exemplos deste laboratório, usamos o placeholder fictício `app.exemplo.com` — substitua pelo seu domínio real.
- **Uma VPC com pelo menos 2 subnets públicas em Zonas de Disponibilidade (AZs) distintas** — exigência do ALB. Pode ser a VPC padrão da conta.
- **Acesso ao Console AWS em português (Brasil)**, pois os passos descrevem telas, campos e botões nesse idioma.
- **Noções básicas** de HTTP/HTTPS, DNS e grupos de segurança (Security Groups).
- Para o Cenário 2, acesso a um terminal com **`openssl`** disponível (apenas para gerar um certificado autoassinado de demonstração).

> ⚠️ **Atenção a custos:** o laboratório cria recursos cobrados (ALB, EC2). Consulte os valores estimados e os alertas de custo em [`IMPLANTACAO.md`](IMPLANTACAO.md) **antes** de criar qualquer recurso, e execute o **Procedimento de Limpeza** ao final para não deixar recursos ativos.

## Visão geral da arquitetura

![Visão geral da solução: usuário acessa por HTTPS, Route 53 resolve o domínio para o ALB, o ALB usa o certificado do ACM para a terminação TLS e encaminha HTTP para a EC2 Linux](imagens/imagem%20(23).png)

A arquitetura de referência coloca o **ALB como ponto de terminação TLS**. O caminho de uma requisição HTTPS é:

1. O usuário acessa `https://app.exemplo.com` (porta 443).
2. O **Route 53** resolve o nome por um registro **alias** que aponta para o **DNS name do ALB**.
3. O **Listener HTTPS 443** do ALB apresenta o certificado do ACM e **encerra a conexão TLS** (terminação TLS), descriptografando o tráfego.
4. O ALB encaminha a requisição como **HTTP na porta 80** para a EC2 Linux registrada no **Target Group**.
5. A EC2 responde em HTTP; o ALB re-cifra a resposta na conexão TLS e a entrega ao navegador.

O detalhamento completo do diagrama (um diagrama **Mermaid**), a descrição de cada componente (ACM, ALB, Listener HTTPS, Target Group, EC2 Linux, Route 53, Security Groups e VPC/Subnets) e o "porquê" de cada decisão de segurança estão em [`ARQUITETURA.md`](ARQUITETURA.md).

## Recursos por cenário (Cenário 1 × Cenário 2)

O **Cenário 2** foi implementado com **recursos próprios** (segundo ALB e segundo Target Group), **compartilhando a mesma EC2** e preservando integralmente o Cenário 1. Os nomes abaixo são **variáveis (placeholders)** definidas na tabela **Variáveis do laboratório** em [`IMPLANTACAO.md`](IMPLANTACAO.md); substitua cada `<VARIAVEL>` pelos seus próprios valores:

| Recurso | Cenário 1 | Cenário 2 |
|---------|-----------|-----------|
| ALB | `<ALB_1>` | `<ALB_2>` |
| Target Group | `<TG_1>` | `<TG_2>` |
| EC2 | `<EC2_APP>` | **a mesma** `<EC2_APP>` |
| Security Group do ALB | `<SG_ALB>` | **o mesmo** `<SG_ALB>` |
| Certificado | Público (AWS ACM) | Autoassinado importado |
| Emissor | Amazon | O próprio certificado |
| Renovação | Automática (quando elegível) | Manual |
| Navegador | Confiável (cadeado) | Aviso "Não seguro" |
| HTTPS | 443 | 443 |
| Backend (ALB → EC2) | HTTP 80 | HTTP 80 |

> **Público × autoassinado (resumo):** um **certificado público** do ACM é assinado por uma CA reconhecida, cuja raiz já está no trust store dos navegadores — por isso é **confiável sem aviso**. Um **certificado autoassinado** é assinado por ele mesmo (emissor = subject) e **não** tem uma CA confiável na cadeia — por isso o navegador mostra **"Não seguro"**, mesmo com o status `Emitido` no ACM. Em ambos os casos o **tráfego é cifrado (TLS)**; o que muda é a **validação de identidade/confiança**.

![Imagem](imagens/imagem%20(25).png)

## Resultado da implantação

A implantação real confirmou o comportamento esperado nos dois cenários. O resumo observado foi:

- **Cenário 1 (público):** o acesso a `https://<DOMINIO_PUBLICO>` abriu **sem aviso de certificado** (cadeado fechado). Nos detalhes do certificado, o navegador exibiu um **certificado wildcard** (`<DOMINIO_WILDCARD>`) **emitido pela Amazon** (CA pública `Amazon RSA 2048`), com o `<EC2_APP>` exibindo `Healthy` no Target Group `<TG_1>`.
  <table>
    <tr>
      <td align="center"><img src="imagens/imagem%20(12).png" width="300" alt="Navegador acessando o domínio público com conexão segura (Cenário 1)"><br><sub>Acesso HTTPS seguro ao domínio público</sub></td>
      <td align="center"><img src="imagens/imagem%20(16).png" width="300" alt="Detalhes do certificado wildcard emitido pela Amazon RSA 2048 M04"><br><sub>Certificado wildcard emitido pela Amazon</sub></td>
      <td align="center"><img src="imagens/imagem%20(10).png" width="300" alt="Lista de certificados no ACM"><br><sub>Certificados no ACM</sub></td>
    </tr>
  </table>

- **Cenário 2 (autoassinado importado):** o acesso a `https://<DOMINIO_AUTOASSINADO>` abriu a mesma página, mas o navegador marcou a conexão como **"Não seguro"** com o erro `NET::ERR_CERT_AUTHORITY_INVALID`. Nos detalhes, o certificado aparece com **emissor = titular** (`<DOMINIO_AUTOASSINADO>`), confirmando a natureza autoassinada, e o certificado consta no ACM como **Tipo `Imported`**, **Status `Emitido`**, **Em uso: Sim**.

  <table>
    <tr>
      <td align="center"><img src="imagens/imagem%20(14).png" width="300" alt="Navegador acessando o domínio autoassinado com aviso Não seguro (Cenário 2)"><br><sub>Domínio autoassinado: "Não seguro"</sub></td>
      <td align="center"><img src="imagens/imagem%20(15).png" width="300" alt="Detalhes do certificado autoassinado: emissor igual ao titular"><br><sub>Certificado autoassinado (emissor = titular)</sub></td>
      <td align="center"><img src="imagens/imagem%20(13).png" width="300" alt="Acesso ao ALB 2 pelo DNS da AWS com aviso Não seguro"><br><sub>ALB 2 pelo DNS da AWS</sub></td>
    </tr>
  </table>
- **No ACM**, coexistiram os três certificados do laboratório: o **wildcard público** (`<DOMINIO_WILDCARD>`, Em uso: Sim, qualificado para renovação automática), o **certificado do domínio público** validado por DNS e o **importado autoassinado** (`<DOMINIO_AUTOASSINADO>`, renovação manual).
- **No Route 53**, a Hosted Zone `<ZONA_DNS>` recebeu os registros **A Alias** apontando `<DOMINIO_PUBLICO>` → `<ALB_1>` e `<DOMINIO_AUTOASSINADO>` → `<ALB_2>`, além do **CNAME de validação** do certificado público.

![Registros da Hosted Zone no Route 53: A Alias para os dois ALBs e CNAME de validação](imagens/imagem%20(11).png)

A sequência completa de evidências (Security Groups, EC2, Target Groups, ALBs, ACM, Route 53 e as telas do navegador para os dois cenários) está documentada em [`TESTES.md`](TESTES.md).

## Instruções de uso

Siga os documentos do laboratório nesta ordem:

1. **[`ARQUITETURA.md`](ARQUITETURA.md)** — leia primeiro para entender a arquitetura de referência, os componentes AWS envolvidos e o fluxo da terminação TLS no ALB.
2. **[`IMPLANTACAO.md`](IMPLANTACAO.md)** — execute o passo a passo numerado no Console AWS. O documento cobre:
   - a **tabela de variáveis** do laboratório (placeholders para todos os identificadores) e os cuidados de custo;
   - o **Cenário 1** (certificado público do ACM + Validacao_DNS + Listener HTTPS + alias no Route 53 + renovação automática);
   - o **Cenário 2** (certificado autoassinado gerado no **Windows/PowerShell** ou **Linux/CloudShell**, importado no ACM, exposto por um **segundo ALB** `<ALB_2>` e **segundo Target Group** `<TG_2>` que **compartilham a mesma EC2** do Cenário 1 + renovação manual);
   - o **Cenário 3** (certificados privados / AWS Private CA — **conceitual**, sem provisionar);
   - o **Procedimento de Limpeza** ordenado por dependência, para não deixar recursos cobrados ativos.
3. **[`TESTES.md`](TESTES.md)** — os testes manuais verificáveis (T01 a T11) já foram **executados nesta implantação** e incluem testes **positivos** (acesso HTTP, HTTPS válido, troca de certificado, verificação de expiração) e **negativos** (certificado incorreto para o domínio, certificado não confiável, falha de permissão). Cada teste tem identificador, objetivo, pré-condições, passos, resultado esperado observável, critério de **pass/fail** e as **capturas de tela** que comprovam o resultado.

> **Importante:** este laboratório é **100% manual** e não utiliza Terraform, CloudFormation, CDK, CLI ou SDK. Qualquer script (ex.: `openssl`, `curl`) é curto (até 20 linhas), apenas para demonstração ou validação, e usa placeholders fictícios — nunca credenciais reais.

## Conceitos de certificados SSL/TLS

Esta seção consolida os conceitos de certificados SSL/TLS do laboratório. É o material de revisão rápida do conteúdo.

### Público × importado × privado

| Tipo | Autoridade emissora | Confiável por navegadores públicos? | Renovação | Caso de uso típico |
|------|---------------------|-------------------------------------|-----------|--------------------|
| **Público** (`Certificado_Publico`) | ACM (Amazon Issued) | **Sim**, sem instalação adicional | **Automática** pelo ACM (enquanto a Validacao_DNS permanecer válida) | Sites e APIs expostos na internet |
| **Importado** (`Certificado_Importado`) | CA externa (ou autoassinado, só para demonstração) | Depende da CA; **autoassinado não é confiável** por padrão | **Manual** (gerar e reimportar antes do `notAfter`) | Certificado já adquirido de uma CA externa |
| **Privado** (`Certificado_Privado`) | CA privada (ex.: AWS Private CA) | Somente por clientes que confiam na CA privada (root no **trust store**) | Manual ou automática | Endpoints internos corporativos, mTLS entre serviços |

### Uso de certificados em empresas e domínios externos

O domínio **não precisa estar registrado nem ter seu DNS no Route 53** para utilizar um certificado público do ACM. Se o DNS estiver no Registro.br ou em outro provedor, basta criar nele o **CNAME de validação** informado pelo ACM; depois, configurar o apontamento da aplicação para o ALB.

Em uma empresa, há duas situações comuns:

- **Solicitar um certificado público no ACM:** indicado quando a empresa quer um certificado gerenciado pela AWS, com **renovação automática quando os requisitos forem atendidos**, mesmo usando DNS externo.
- **Importar um certificado existente:** indicado quando a empresa já possui um certificado emitido por uma autoridade reconhecida, como DigiCert ou Sectigo. A importação exige o certificado, a **chave privada correspondente** e, quando aplicável, a cadeia intermediária. Esse certificado pode ser confiável no navegador, mas a **renovação e reimportação são responsabilidade da empresa**.

**O certificado autoassinado do Cenário 2** (`<DOMINIO_AUTOASSINADO>`) também pôde ser importado, mas o navegador apresentou **"Não seguro"**, pois não confia automaticamente em quem o assinou. **Importar um certificado no ACM não o torna confiável.** A chave privada nunca deve ser compartilhada publicamente nem enviada ao GitHub.

### Renovação automática × manual

- **Renovação automática (`Renovacao_Automatica`):** o ACM renova certificados **públicos** emitidos por ele **sem ação do responsável**, desde que o registro **CNAME de validação** continue presente no Route 53. Não exige intervenção manual.
- **Renovação manual (`Renovacao_Manual`):** certificados **importados** exigem que o responsável **gere um novo certificado antes da data `notAfter` e o reimporte** no ACM. Se não for renovado, o certificado expira e as conexões HTTPS passam a ser recusadas ou exibem aviso de certificado inválido.

### Terminação TLS no ALB

Na **terminação TLS (`Terminacao_TLS`)**, o ALB **encerra a conexão TLS na porta 443**, descriptografa o tráfego HTTPS e o encaminha como **HTTP na porta 80** para a EC2 Linux. Vantagens: o certificado é gerenciado em um único ponto (o Listener HTTPS), a renovação automática não exige tocar na EC2, e a aplicação na EC2 pode falar apenas HTTP. O trecho ALB → EC2 trafega HTTP **dentro da VPC**, protegido por Security Groups (a EC2 só aceita a porta 80 a partir do Security Group do ALB).

### Certificado na mesma Região do ALB

Um certificado usado pelo **ALB deve estar provisionado na mesma Região do ALB**. A integração ACM ↔ ALB ocorre pela associação de um certificado do ACM (daquela Região) a um Listener HTTPS do ALB. Um certificado em outra Região **não aparece** para associação ao listener.

### us-east-1 obrigatório para CloudFront

Diferentemente do ALB, um certificado usado pelo **CloudFront deve ser provisionado na Região `us-east-1` (Norte da Virgínia)**, independentemente de onde o conteúdo é servido. A integração ACM ↔ CloudFront associa um certificado de `us-east-1` a uma distribuição do CloudFront. Vale guardar essa diferença: ALB = mesma Região; CloudFront = sempre `us-east-1`.

### Certificado wildcard: cobre apenas um nível de subdomínio

Um **certificado wildcard (`Certificado_Wildcard`)** no formato `*.exemplo.com` cobre **apenas um nível** de subdomínio:

- ✅ **Inclui** `app.exemplo.com` (um nível).
- ❌ **Exclui** `a.b.exemplo.com` (dois níveis).

Ou seja, o `*` substitui **um único rótulo** de subdomínio. Para cobrir `a.b.exemplo.com` seria necessário um wildcard específico como `*.b.exemplo.com`.

### Certificado TLS (ACM) × chave KMS

É comum confundir os dois serviços. A distinção é:

- **Certificado TLS (ACM):** autentica a **identidade de um domínio** e **habilita conexões HTTPS**. Vive no Listener HTTPS do ALB (ou na distribuição do CloudFront). Não é usado para cifrar dados da aplicação.
- **Chave KMS (`KMS`):** destina-se a **cifrar e decifrar dados** (por exemplo, dados em repouso em S3, EBS ou RDS). **Não** emite, substitui ou faz o papel de um certificado TLS no handshake.

Em resumo: **certificado TLS = identidade do domínio e HTTPS**; **chave KMS = criptografia de dados**.

## Estrutura dos documentos do laboratório

| Documento | Conteúdo |
|-----------|----------|
| [`README.md`](README.md) | Este artigo didático: objetivo, pré-requisitos, visão geral, instruções de uso e conceitos de certificados SSL/TLS |
| [`ARQUITETURA.md`](ARQUITETURA.md) | Diagrama Mermaid da arquitetura de referência e descrição dos componentes e do fluxo da terminação TLS |
| [`IMPLANTACAO.md`](IMPLANTACAO.md) | Passo a passo manual no Console AWS, tabela de variáveis, os três cenários e o Procedimento de Limpeza |
| [`TESTES.md`](TESTES.md) | Tabela de testes manuais verificáveis (T01–T11), positivos e negativos, com critério de pass/fail e capturas de tela da execução |
