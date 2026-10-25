# AWS Certificate Manager SSL/TLS na Prática: Testes

## Introdução

Este documento reúne os **testes de funcionamento** do laboratório de ACM. Todos os testes aqui descritos são **procedimentos manuais verificáveis**, executados pelo próprio estudante (via navegador e/ou comandos curtos de validação no terminal). **Não há testes automatizados** neste laboratório: o entregável é a configuração manual de serviços AWS (ACM, ALB, EC2 Linux, Route 53) pelo Console AWS em português (Brasil), e a verificação é feita observando o comportamento real dos recursos provisionados.

> ✅ **Execução realizada.** Esta rodada do laboratório foi implantada e os testes dos dois cenários foram executados com **resultado PASS**. As **capturas de tela** que comprovam cada resultado estão reunidas na seção [Evidências da execução](#evidências-da-execução) ao final deste documento. Os testes que dependem de condições adicionais não montadas nesta rodada (T03, T05, T07) permanecem marcados como **não executados**. As imagens contêm elementos reais do console (IDs de conta, instância, VPC) por serem evidências; no texto dos documentos, use sempre as **variáveis/placeholders**.

Os testes se dividem em dois tipos:

- **Testes positivos**: confirmam o comportamento esperado em condições corretas (acesso HTTP, HTTPS válido, troca de certificado no listener, verificação de expiração).
- **Testes negativos**: confirmam o tratamento correto de condições incorretas (certificado incorreto para o domínio, certificado não confiável e falha de permissão).

### Critério de pass/fail

Cada teste possui um **resultado esperado observável** e um **resultado de falha** correspondente. O critério de encerramento é objetivo:

- **PASS (aprovado)**: o resultado observado corresponde ao campo **Resultado esperado**.
- **FAIL (reprovado)**: o resultado observado corresponde ao campo **Resultado de falha** (ou diverge do resultado esperado). Nesse caso, o teste indica um problema de configuração ou um comportamento negativo corretamente demonstrado, conforme o tipo do teste.

### Campos obrigatórios de cada teste

Todo teste é composto pelos campos obrigatórios abaixo, representados como colunas na tabela:

- **ID**: identificador numérico único e sequencial (T01, T02, ...).
- **Objetivo**: o que o teste pretende validar.
- **Pré-requisito**: pré-condições que precisam estar satisfeitas antes da execução.
- **Procedimento**: passos ordenados de execução.
- **Resultado esperado**: resultado observável que caracteriza o PASS.
- **Resultado de falha**: resultado observável que caracteriza o FAIL.
- **Explicação**: o porquê do comportamento observado.

> **Teste não concluído:** se um teste **não possuir todos os campos obrigatórios** (ID, objetivo, pré-condições, passos ordenados, resultado esperado observável e condição de encerramento com critério de pass/fail), ele deve ser marcado como **não concluído** e não pode ser considerado aprovado nem reprovado até que todos os campos sejam preenchidos.

Os valores de domínio usados nos exemplos são **placeholders fictícios** (por exemplo, `app.exemplo.com`), distinguíveis de credenciais ou domínios reais. Substitua-os pelos seus próprios valores ao executar.

## Tabela de testes (T01–T11)

| ID | Objetivo | Pré-requisito | Procedimento | Resultado esperado | Resultado de falha | Explicação |
|----|----------|---------------|--------------|--------------------|--------------------|------------|
| **T01** *(Positivo — PASS)* | Validar que a aplicação na EC2 Linux responde a requisições **HTTP** na porta 80 retornando uma página HTML estática. | EC2 Linux provisionada com servidor web servindo HTML na porta 80; acesso de rede do cliente de teste à instância (ou ao Target Group). | 1. Obter o endereço HTTP de teste da aplicação (IP/DNS da EC2 ou alvo do Target Group). 2. Acessar `http://<endereco>:80/` pelo navegador **ou** executar o script de validação `curl` (ver abaixo). 3. Observar o código de status e o corpo da resposta. | Resposta **HTTP 200** com o conteúdo da **página HTML estática** da aplicação. | Timeout, recusa de conexão (`connection refused`) ou código de status diferente de 200, indicando que a app/porta 80 não está ativa. | A aplicação na EC2 fala **apenas HTTP** na porta 80; a camada TLS vive no ALB. Este teste isola a saúde da aplicação de origem antes de testar o HTTPS fim a fim. |
| **T02** *(Positivo — PASS)* | Validar o acesso **HTTPS válido** ao domínio por meio do ALB, com cadeia de certificado confiável. | Cenário 1 concluído: Certificado_Publico no estado `Issued`, associado ao Listener HTTPS 443; registro **alias** no Route 53 apontando o domínio para o DNS name do ALB; Target Group `healthy`. | 1. Acessar `https://app.exemplo.com/` pelo navegador. 2. Inspecionar o cadeado/detalhes do certificado (emissor e validade). 3. Opcionalmente, executar o script `curl` de validação de TLS (ver abaixo). | Conexão **TLS estabelecida com cadeia de certificado confiável** (sem avisos do navegador) e a **página HTML** da aplicação é exibida. **Observado:** cadeado fechado em `https://<DOMINIO_PUBLICO>`, certificado wildcard `<DOMINIO_WILDCARD>` emitido por `Amazon RSA 2048` (ver [Evidências](#evidências-da-execução)). | Aviso de certificado no navegador, erro de handshake TLS, ou a página não carrega — indicando problema no certificado, no listener ou no apontamento DNS. | O Route 53 resolve o domínio para o ALB; o Listener HTTPS 443 apresenta o certificado do ACM e encerra o TLS (**terminação TLS**), encaminhando HTTP para a EC2. |
| **T03** *(Negativo)* | Demonstrar o comportamento quando o certificado apresentado **não corresponde ao domínio** acessado (name mismatch). | Um certificado cujo nome (CN/SAN) **não** cobre o domínio de acesso associado ao Listener HTTPS (ex.: certificado para `outro.exemplo.com` enquanto se acessa `app.exemplo.com`). | 1. Associar ao Listener HTTPS um certificado que não cobre o domínio de teste. 2. Acessar `https://app.exemplo.com/` pelo navegador. 3. Observar a mensagem de erro de segurança exibida. | O navegador **exibe erro de incompatibilidade de nome** (ex.: `NET::ERR_CERT_COMMON_NAME_INVALID` / *name mismatch*), bloqueando o acesso por padrão. | A conexão é aceita como válida sem qualquer aviso — o que indicaria que o certificado realmente cobre o domínio (cenário não aplicável a este teste negativo). | A validação TLS exige que o nome do certificado corresponda ao domínio (CN/SAN). Quando não corresponde, o cliente rejeita a identidade apresentada. |
| **T04** *(Negativo)* | Demonstrar que um **certificado importado não confiável** (autoassinado) gera aviso no navegador. | Cenário 2 concluído: Certificado_Importado **autoassinado** (tipo `Imported` no ACM) associado ao Listener HTTPS 443. | 1. Associar o certificado autoassinado importado ao Listener HTTPS. 2. Acessar `https://app.exemplo.com/` pelo navegador. 3. Observar o aviso de segurança exibido. | O navegador **exibe aviso de certificado não confiável** (ex.: `NET::ERR_CERT_AUTHORITY_INVALID` / *certificate not trusted*), pois a cadeia não remonta a uma autoridade raiz confiável. | O navegador aceita o certificado sem aviso — o que contraria a natureza de um certificado autoassinado (cenário não aplicável). | Certificados **autoassinados não são confiáveis por padrão** para navegadores públicos: não há uma CA reconhecida no trust store que valide a cadeia. |
| **T05** *(Positivo)* | Validar a **alteração do certificado** associado ao Listener HTTPS e que o HTTPS reflete a troca. | Pelo menos dois certificados disponíveis no ACM (ex.: o público `Issued` e um importado); Listener HTTPS 443 já existente. | 1. No Console, editar o Listener HTTPS 443 do ALB. 2. Selecionar/trocar o **certificado padrão** para o outro certificado do ACM. 3. Salvar e reacessar `https://app.exemplo.com/`, inspecionando os detalhes do certificado apresentado. | O **novo certificado aparece selecionado** no listener e a conexão HTTPS passa a **apresentar o novo certificado** (emissor/validade correspondentes ao escolhido). | O listener continua apresentando o certificado antigo, ou não aceita a troca — indicando que a alteração não foi aplicada. | O listener é o ponto onde o certificado é efetivamente usado; trocar o certificado padrão altera, de imediato, a identidade apresentada no handshake TLS. |
| **T06** *(Positivo — PASS)* | Verificar a **data de expiração** (`notAfter`) do certificado e o cumprimento da **janela mínima de antecedência** para renovação. | Certificado presente no ACM (público ou importado) com data `notAfter` legível; janela mínima de antecedência definida (ex.: **30 dias**). | 1. No ACM, abrir o certificado e localizar a data de expiração (`notAfter`). 2. Calcular os dias restantes entre a data atual e `notAfter`. 3. Comparar com a janela mínima (ex.: 30 dias). 4. Opcionalmente, validar o `notAfter` com o script `openssl`/`curl` (ver abaixo). | O campo **`notAfter` é legível** e os **dias restantes são ≥ 30** (janela mínima). Para certificado público, a renovação é automática enquanto o CNAME de validação permanecer no Route 53; para importado, a renovação é manual (gerar novo e reimportar antes do `notAfter`). | Dias restantes **< 30** sem ação de renovação planejada, ou `notAfter` ilegível/ausente — sinalizando risco de expiração e interrupção do HTTPS. | A expiração do certificado derruba o HTTPS; monitorar `notAfter` com antecedência permite renovar antes da falha (automática no ACM público; manual no importado). |
| **T07** *(Negativo)* | Demonstrar o comportamento de uma **falha de permissão (IAM)** ao operar um recurso do laboratório. | Um usuário/role **sem** a permissão necessária para a ação testada (ex.: sem `acm:ImportCertificate` ou sem permissão de associar certificado ao listener). | 1. Autenticar-se com a identidade de permissão restrita. 2. Tentar executar a ação negada (ex.: importar certificado no ACM ou associá-lo ao listener). 3. Observar a mensagem de erro de autorização retornada. | O serviço **nega a ação** e retorna uma mensagem de erro que identifica **origem (quem — ARN do principal)**, **ação negada** (ex.: `acm:ImportCertificate`) e **recurso afetado** (ARN do recurso). Exemplo: `User: arn:aws:iam::<conta>:user/<usuario> is not authorized to perform: acm:ImportCertificate on resource: <arn-do-recurso>`. | A ação é concluída com sucesso — o que indica que a identidade possuía a permissão (cenário não aplicável a este teste negativo). | O IAM aplica o princípio do menor privilégio: sem a permissão específica, a ação é negada e o erro identifica claramente **quem**, **o quê** e **sobre qual recurso**. |
| **T08** *(Positivo — PASS)* | Validar a **importação** do certificado autoassinado no ACM. | `app2.crt` e `app2.key` gerados (Passo 15) via OpenSSL, região `<REGIAO>`. | 1. ACM → **Importar certificado**. 2. Colar o PEM do certificado e da chave; cadeia em branco. 3. Importar e abrir o certificado. | Certificado listado com **Tipo = `Imported`** e **Status = `Emitido`**, domínio `<DOMINIO_AUTOASSINADO>`. **Observado:** certificado importado presente no ACM com `Em uso: Sim` (ver [Evidências](#evidências-da-execução)). | A importação falha (par chave/cert inválido ou PEM incompleto) com erro do ACM. | O ACM armazena o certificado importado; `Emitido` indica armazenamento bem-sucedido, **não** confiança do navegador. |
| **T09** *(Positivo — PASS)* | Validar a **associação** do certificado importado ao Listener HTTPS:443 do segundo ALB. | `<ALB_2>` criado; certificado importado `Emitido` no ACM. | 1. EC2 → Load Balancers → `<ALB_2>` → Listener HTTPS:443 → Editar. 2. Selecionar o certificado importado `<DOMINIO_AUTOASSINADO>`. 3. Salvar. | O listener HTTPS:443 exibe o **certificado importado** como padrão; o ALB passa a apresentá-lo no handshake. **Observado:** `<ALB_2>` ativo servindo o certificado autoassinado (ver [Evidências](#evidências-da-execução)). | O certificado não aparece na lista (região errada) ou a troca não é salva. | O certificado é associado ao **listener** (não à EC2); é o ponto da terminação TLS. |
| **T10** *(Positivo — PASS)* | Validar o **Health Check** do segundo Target Group (`<TG_2>`). | `<TG_2>` criado (HTTP:80, `/`) com `<EC2_APP>` registrada. | 1. EC2 → Grupos de destino → `<TG_2>` → Destinos. 2. Acompanhar o status de integridade. | Destino **`healthy`** em até 300s. **Observado:** `<EC2_APP>` com status `Healthy` em `<TG_2>` (a mesma instância também aparece `Healthy` em `<TG_1>`), confirmando o compartilhamento da EC2 (ver [Evidências](#evidências-da-execução)). | Permanece `unhealthy` (httpd parado, SG sem 80 do `<SG_ALB>`, ou caminho de health check errado). | O ALB só encaminha para destinos saudáveis; dois TGs podem compartilhar a mesma EC2. |
| **T11** *(Negativo — PASS)* | Demonstrar o **aviso "Não seguro"** ao acessar o segundo ALB por HTTPS com o certificado autoassinado. | Cenário 2 concluído: HTTPS:443 do `<ALB_2>` com o certificado importado. | 1. Acessar `https://<DOMINIO_AUTOASSINADO>/` (ou o DNS `<DNS_ALB_2>`). 2. Observar o aviso. 3. Inspecionar o certificado (emissor, nome, validade). | A página HTML carrega, mas o navegador exibe **"Não seguro"**; o certificado apresentado é o autoassinado `<DOMINIO_AUTOASSINADO>`. **Observado:** aviso "Não seguro" / `NET::ERR_CERT_AUTHORITY_INVALID`, com **emissor = titular** no visualizador de certificado (ver [Evidências](#evidências-da-execução)). | O navegador aceita sem aviso (não aplicável a autoassinado) ou a página não carrega. | TLS está ativo (dados cifrados), mas a **cadeia de confiança/identidade** falha (autoassinado + possível *name mismatch* com o DNS do ALB). |

### Scripts de validação (curtos, apenas validação — sem provisionamento)

Os trechos abaixo são **apenas para validação** (não provisionam recursos) e usam **placeholders fictícios** (`app.exemplo.com`). Cada trecho tem no máximo 20 linhas. Substitua os placeholders pelos seus valores reais ao executar.

**T01 — validar HTTP na porta 80:**

```bash
# Valida resposta HTTP 200 e exibe o inicio do HTML retornado pela app na EC2
curl -i -s http://app.exemplo.com:80/ | head -n 20
```

**T02 — validar HTTPS com cadeia confiavel:**

```bash
# Falha (saida de erro) se a cadeia TLS nao for confiavel; mostra apenas o status HTTP
curl -I --fail https://app.exemplo.com/
```

**T06 — verificar a data de expiracao (notAfter) do certificado:**

```bash
# Exibe a data notAfter do certificado apresentado pelo dominio (via handshake TLS)
echo | openssl s_client -connect app.exemplo.com:443 -servername app.exemplo.com 2>/dev/null \
  | openssl x509 -noout -enddate
```

## Evidências da execução

As capturas abaixo foram coletadas durante a implantação real e comprovam os resultados PASS registrados na tabela. As imagens exibem elementos reais do console (IDs de conta, instância, VPC, DNS) por serem evidências de execução; no texto do laboratório, use sempre as variáveis/placeholders.

### Infraestrutura base (Cenário 1)

**Security Group do ALB (`<SG_ALB>`) — regras de entrada HTTPS 443 e HTTP 80:**

![Security Group do ALB com regras de entrada 443 e 80](imagens/imagem%20(3).png)

**Instância EC2 (`<EC2_APP>`) em execução (T01):**

![Instância EC2 da aplicação em execução](imagens/imagem%20(2).png)

**Target Group do Cenário 1 (`<TG_1>`) com a EC2 `Healthy` (T01/T10):**

![Target Group 1 com destino Healthy](imagens/imagem%20(5).png)

**Detalhes do Target Group 1 — 1 destino Íntegro (T10):**

![Detalhes do Target Group 1 com 1 destino íntegro](imagens/imagem%20(8).png)

**Application Load Balancer do Cenário 1 (`<ALB_1>`) ativo:**

![ALB 1 ativo, com detalhes do balanceador](imagens/imagem%20(6).png)

### Certificados no ACM

**ACM — os três certificados do laboratório: importado autoassinado, público do domínio e wildcard público (T06/T08):**

![ACM listando os três certificados: importado autoassinado, público do domínio e wildcard](imagens/imagem%20(10).png)

### Geração do certificado autoassinado (Cenário 2)

**Geração do certificado autoassinado com OpenSSL (PowerShell):**

![Geração do certificado autoassinado com OpenSSL no PowerShell](imagens/imagem%20(17).png)

**Arquivos `app2.crt` e `app2.key` gerados localmente:**

![Arquivos app2.crt e app2.key na pasta do laboratório](imagens/imagem%20(18).png)

**Conteúdo PEM do certificado `app2.crt` (usado no campo "Corpo do certificado" da importação, T08):**

![Conteúdo PEM do certificado app2.crt](imagens/imagem%20(19).png)

**Arquivo `app2.key` (chave privada, conteúdo ocultado — nunca publicar):**

![Arquivo app2.key aberto com o conteúdo ocultado](imagens/imagem%20(20).png)

### Infraestrutura do Cenário 2

**Target Group do Cenário 2 (`<TG_2>`) com a mesma EC2 `Healthy` (T10):**

![Target Group 2 com a mesma EC2 Healthy](imagens/imagem%20(4).png)

**Destinos do Target Group 2 — EC2 `Healthy` (T10):**

![Destinos do Target Group 2 com a EC2 Healthy](imagens/imagem%20(9).png)

**Application Load Balancer do Cenário 2 (`<ALB_2>`) ativo (T09):**

![ALB 2 ativo servindo o certificado importado](imagens/imagem%20(7).png)

### DNS (Route 53)

**Registros A Alias para `<DOMINIO_PUBLICO>` e `<DOMINIO_AUTOASSINADO>` e o CNAME de validação:**

![Route 53 com os registros A Alias dos dois cenários e o CNAME de validação](imagens/imagem%20(11).png)

### Resultado no navegador

**Cenário 1 — `https://<DOMINIO_PUBLICO>` abre seguro, sem aviso (T02):**

![Página da aplicação acessada por HTTPS seguro no Cenário 1](imagens/imagem%20(12).png)

**Cenário 1 — detalhes do certificado: wildcard `<DOMINIO_WILDCARD>` emitido por `Amazon RSA 2048 M04` (T02/T06):**

![Visualizador do certificado wildcard emitido pela Amazon](imagens/imagem%20(16).png)

**Cenário 2 — `https://<DOMINIO_AUTOASSINADO>` exibe "Não seguro" (T11):**

![Página do Cenário 2 com aviso Não seguro no navegador](imagens/imagem%20(14).png)

**Cenário 2 — detalhes do certificado autoassinado: emissor = titular, validade de 1 ano (T06/T11):**

![Visualizador do certificado autoassinado com emissor igual ao titular](imagens/imagem%20(15).png)

**Cenário 2 — acesso pelo DNS do ALB (`<DNS_ALB_2>`) também marcado como "Não seguro" (T11):**

![Acesso ao ALB 2 pelo DNS da AWS com aviso Não seguro](imagens/imagem%20(13).png)
