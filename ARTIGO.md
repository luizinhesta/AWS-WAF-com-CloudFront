# Protegendo uma aplicação web com AWS WAF: XSS e geo-bloqueio na borda do CloudFront

## Introdução

Este projeto é um laboratório educacional que demonstra, de forma prática e controlada, a diferença entre uma aplicação publicada **sem AWS WAF** e a mesma aplicação publicada **com AWS WAF**. O problema que ele ajuda a entender é concreto: como uma camada de proteção na borda da rede consegue barrar requisições maliciosas **antes** que elas cheguem à origem da aplicação.

Para tornar essa diferença visível, o laboratório publica **dois endpoints** que servem exatamente o **mesmo conteúdo**, a partir de um único bucket Amazon S3 privado:

- **Sem WAF** — uma requisição com padrão de *Cross-Site Scripting* (XSS) chega normalmente à origem.
- **Com WAF** — a mesma requisição é **bloqueada com HTTP 403** na borda do CloudFront, antes de tocar o S3.

O endpoint protegido recebe duas defesas complementares: uma que inspeciona o **conteúdo** da requisição (XSS na *query string*) e outra que controla a **origem geográfica** (bloqueio de acessos de fora do Brasil, com uma página HTML de acesso negado personalizada). É importante destacar o caráter defensivo do exercício: o teste de XSS usa apenas uma *string* com padrão característico de ataque contra os próprios endpoints do laboratório. **Nenhum script é executado** — o S3 serve apenas arquivos estáticos — e não há geração de tráfego em volume, DDoS ou *stress test*.

![Visão geral do laboratório](<imagens/imagem%20(1).png>)

## Arquitetura

A arquitetura foi desenhada de modo que a **única variável** entre os dois endpoints seja a presença (ou não) do WAF. Ambas as distribuições CloudFront apontam para o **mesmo bucket S3 privado**, o que isola o efeito da Web ACL e torna a comparação honesta.

```
                          INTERNET
                             |
                         Route 53
                             |
             +---------------+---------------+
             |                               |
             v                               v
  site-sem-waf.dominio.com        site-com-waf.dominio.com
             |                               |
             v                               v
    CloudFront SEM WAF              CloudFront COM WAF
             |                               |
             |                            AWS WAF
             |                               |
             +---------------+---------------+
                             |
                             v
                        S3 PRIVADO
                             |
                        index.html
```

No **fluxo sem WAF**, o Route 53 resolve o subdomínio para a distribuição CloudFront, que entrega o conteúdo do S3 via OAC. Como não existe Web ACL associada, uma requisição com padrão de XSS não é inspecionada e chega à origem normalmente (tende a retornar `200`).

No **fluxo com WAF**, a Web ACL `waf-lab-xss` é avaliada na borda do CloudFront, antes da origem, e nesta ordem:

1. **`Block-Fora-do-Brasil`** — se a origem estiver fora do Brasil, bloqueia com **HTTP 403** e devolve a página HTML personalizada de acesso negado.
2. **`Block-XSS-Lab`** — se a origem for o Brasil mas a *query string* contiver o padrão de XSS, bloqueia com **HTTP 403** padrão.

Em ambos os bloqueios o S3 **não** é acessado. A regra de geo tem prioridade mais alta, então uma origem estrangeira recebe a página de acesso negado antes de qualquer inspeção de XSS.

![Fluxo e proteções na borda](<imagens/imagem%20(4).png>)

Um ponto central da arquitetura é o **acesso privado ao S3 via Origin Access Control (OAC)**: o bucket permanece privado, com *Block Public Access* ativado, e a *bucket policy* autoriza apenas o serviço `cloudfront.amazonaws.com`, restrito às duas distribuições do laboratório através da condição `AWS:SourceArn`. Assim, o conteúdo só é servido através do CloudFront, nunca por acesso direto ao S3.

## Serviços utilizados

| Serviço | Função no projeto |
|---|---|
| **Amazon Route 53** | Zona hospedada do domínio com dois registros Alias (tipo A) apontando para as duas distribuições CloudFront. |
| **AWS Certificate Manager (ACM)** | Certificado TLS público em `us-east-1` (região exigida pelo CloudFront), validado por DNS, habilitando o HTTPS. |
| **Amazon CloudFront** | Duas distribuições com OAC e HTTPS: uma sem Web ACL e outra associada à Web ACL do WAF. |
| **AWS WAF** | Uma Web ACL (`waf-lab-xss`) de escopo *CloudFront (Global)* com duas regras: `Block-XSS-Lab` (inspeção de XSS na *query string*) e `Block-Fora-do-Brasil` (correspondência geográfica com resposta HTML personalizada). |
| **Amazon S3** | Bucket privado de origem, com *Block Public Access* ativado, servindo o mesmo conteúdo para as duas distribuições. |
| **Amazon CloudWatch** | Métricas `AllowedRequests` / `BlockedRequests` e *Sampled Requests* do WAF para comprovar o bloqueio. |

## Implementação

A implantação foi feita **integralmente pelo Console da AWS**, seguindo uma ordem pensada para evitar bloqueios do assistente (o console novo do WAF exige um recurso já existente para associar a proteção). As etapas principais foram:

1. **Amazon S3** — criação de um bucket privado (com *Block Public Access*) e *upload* dos arquivos do site (`index.html`, `style.css`, `script.js`) na raiz.
2. **AWS Certificate Manager** — solicitação de um certificado público em `us-east-1`, preferindo o curinga `*.dominio` para cobrir os dois subdomínios, com validação por DNS via Route 53.
3. **Amazon CloudFront** — criação das **duas distribuições** apontando para o mesmo bucket, ambas com *Origin Access Control*, HTTPS e `index.html` como objeto raiz padrão. Nenhuma delas ativa a proteção gerenciada paga do assistente; o WAF é adicionado depois, com regras enxutas.
4. **AWS WAF** — criação da Web ACL `waf-lab-xss` com escopo *CloudFront (Global)*, associada apenas à distribuição protegida. Nela foram criadas a regra `Block-XSS-Lab` (inspeciona todos os parâmetros de consulta com transformação *URL decode* e ação *Block*) e a regra `Block-Fora-do-Brasil` (correspondência geográfica negada para o Brasil, com resposta personalizada 403 usando o corpo HTML `acesso-negado-br`).
5. **Política do bucket S3** — conferência de que as **duas** distribuições estão autorizadas no `AWS:SourceArn`, já que a segunda criação pode sobrescrever a autorização da primeira.
6. **Amazon Route 53** — criação dos registros Alias (tipo A) `site-sem-waf` e `site-com-waf` apontando para as respectivas distribuições.

O passo a passo detalhado, com cada campo e opção do Console, está documentado em `IMPLANTACAO.md`. A decisão de manter apenas uma Web ACL com duas regras personalizadas, sem *Managed Rule Groups*, foi deliberada para manter o custo do laboratório no mínimo.

![Criação das distribuições e associação da Web ACL](<imagens/imagem%20(28).png>)

## Testes e resultados

Os testes foram executados **exclusivamente contra os dois endpoints do laboratório**, com um total de quatro requisições — sem *flood*, *stress test* ou DDoS. O objetivo é comparar o comportamento dos endpoints sem e com WAF.

**Resultado esperado:**

| Teste | Sem WAF | Com WAF |
|---|---|---|
| Requisição normal (do Brasil) | 200 | 200 |
| XSS controlado (do Brasil): `?search=<script>alert(1)</script>` | 200 | 403 BLOCKED |
| Qualquer rota (de fora do Brasil) | 200 | 403 + página "Acesso negado" |

**Teste de XSS.** No endpoint sem WAF, tanto a requisição normal quanto a requisição com o *payload* de XSS carregam a página (HTTP 200), pois não há inspeção. No endpoint com WAF, a requisição normal passa (200), mas a requisição com o padrão `<script>alert(1)</script>` na *query string* é bloqueada pela regra `Block-XSS-Lab` com **HTTP 403**, antes de chegar ao S3.

![Requisição no endpoint sem WAF](<imagens/imagem%20(8).png>)
![Requisição normal no endpoint com WAF](<imagens/imagem%20(7).png>)

Além do navegador, o projeto oferece scripts em Python (`tests/test-xss.py`) e PowerShell (`tests/test-xss.ps1`) que fazem as quatro requisições automaticamente e exibem os códigos HTTP lado a lado, tratando o **403 como resultado esperado**. A saída obtida confirma o comportamento:

```
========================================
AWS WAF LAB 01 - XSS
========================================

SEM WAF
  Normal.................... 200
  XSS....................... 200

COM WAF
  Normal.................... 200
  XSS....................... 403 BLOCKED

========================================
```

![Saída dos scripts de teste](<imagens/imagem%20(10).png>)

**Teste de geo-bloqueio.** Acessando o endpoint com WAF a partir do Brasil, o site carrega normalmente (200). Ao acessar de fora do Brasil (usando VPN ou instância em outro país), a regra `Block-Fora-do-Brasil` responde com **HTTP 403** e a **página HTML personalizada** de acesso negado, renderizada no navegador. O endpoint sem WAF permanece acessível de qualquer país.

![Página de acesso negado para origem fora do Brasil](<imagens/imagem%20(3).png>)
![Contraste entre os endpoints no geo-bloqueio](<imagens/imagem%20(6).png>)

**Comprovação no CloudWatch.** Após rodar os testes, as métricas da Web ACL confirmam o bloqueio: o gráfico de *BlockedRequests* aumenta e, na aba de *Sampled Requests*, a requisição de XSS aparece com **Ação: BLOCK** e **regra correspondente `Block-XSS-Lab`**, com a URI contendo o padrão `<script>...`. Da mesma forma, uma requisição vinda de fora do Brasil aparece como *BLOCK* pela regra `Block-Fora-do-Brasil`, com o país de origem diferente de BR.

## Problemas encontrados

Durante a implantação, alguns pontos exigem atenção e foram tratados no próprio guia como situações a diagnosticar:

- **Ordem de criação dos recursos.** O console atual do WAF exige selecionar um recurso a proteger ao criar o "pacote de proteção". Se a distribuição CloudFront ainda não existir, o fluxo trava. A solução adotada foi criar as **distribuições antes** do WAF.
- **Política do bucket sobrescrita.** Como as duas distribuições usam o mesmo bucket, a criação da segunda pode sobrescrever a autorização OAC da primeira na *bucket policy*. Por isso há uma etapa dedicada a conferir que **as duas** distribuições estão presentes no `AWS:SourceArn`.
- **Regra de geo invertida.** A regra `Block-Fora-do-Brasil` depende do *Negate statement* (NOT) ativo. Sem ele, a regra bloqueia justamente o Brasil. Quando o próprio Brasil recebe 403, a verificação recomendada é conferir se o NOT está ativado.
- **403 em requisições normais.** Se requisições comuns (sem XSS) do Brasil retornam 403, o problema tende a estar na política do bucket / OAC, não no WAF — a orientação é revisar a etapa de conferência da *bucket policy*.
- **Cache e propagação.** Mudanças no WAF e no CloudFront levam algum tempo para propagar, e o cache do navegador pode mascarar o resultado. A recomendação prática é testar com Ctrl+F5, em aba anônima ou pelos scripts.
- **VPN resolvendo no Brasil.** No teste de geo, a VPN pode estar conectada a um servidor no próprio Brasil. Antes de concluir que o bloqueio falhou, confirma-se o país real do IP em um site de "meu IP".

## O que aprendi

Trabalhando neste laboratório, entendi na prática que o AWS WAF atua **na borda**, antes da origem, e que isolar uma única variável (a presença do WAF) é o que torna a demonstração convincente. Ver a mesma requisição retornar 200 em um endpoint e 403 no outro deixou claro o valor de inspecionar o tráfego antes de ele alcançar a aplicação.

Também aprendi que detalhes de ordem importam: criar o CloudFront antes do WAF, definir o objeto raiz padrão e conferir a *bucket policy* após a segunda distribuição são passos que, se ignorados, quebram o fluxo. A regra de geo com *Negate statement* me mostrou como uma condição lógica invertida pode produzir exatamente o efeito oposto ao desejado. Por fim, percebi que o OAC é a forma atual e recomendada de manter o S3 privado, e que confirmar o bloqueio pelas métricas e *Sampled Requests* do CloudWatch fecha o ciclo entre configurar e comprovar.

## Conclusão

O laboratório atinge seu objetivo: publica dois endpoints com o mesmo conteúdo e demonstra, de forma controlada e defensiva, como o AWS WAF bloqueia um padrão de XSS na *query string* e nega acessos de fora do Brasil com uma página personalizada, tudo na borda do CloudFront e sem tocar o S3. Os resultados obtidos — 200 no endpoint sem WAF, 403 no endpoint protegido, e a confirmação nas métricas do CloudWatch — batem com o resultado esperado.

Como conhecimento adquirido, fica uma visão integrada de como Route 53, ACM, CloudFront, S3, WAF e CloudWatch se combinam para entregar uma aplicação estática protegida, com acesso privado à origem e proteção observável. É uma base sólida para os próximos laboratórios da série sobre AWS WAF.

## Repositório

- **Repositório:** projeto **AWS-WAF-com-CloudFront** no GitHub.
- **Documentação do projeto:**
  - `README.md` — visão geral, objetivo e serviços utilizados.
  - `ARQUITETURA.md` — arquitetura e detalhamento do funcionamento das regras.
  - `IMPLANTACAO.md` — passo a passo completo de implantação pelo Console AWS, incluindo testes e exclusão dos recursos.
  - `site/` — arquivos do site (`index.html`, `style.css`, `script.js`) e a página `acesso-negado.html`.
  - `tests/` — scripts de teste em Python (`test-xss.py`) e PowerShell (`test-xss.ps1`).
