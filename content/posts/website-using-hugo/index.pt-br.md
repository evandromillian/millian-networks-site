---
title: "Website com Hugo e AWS"
date: 2022-07-04T10:14:22-03:00
description: ""
draft: false
author: "Evandro Millian"
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image-preview.webp"

tags: ["Hugo", "AWS"]
categories: ["Web Development"]
lightgallery: true
---

Meu primeiro post será sobre as escolhas feitas durante o desenvolvimento deste site. Para cada projeto, seja um site ou um jogo, sempre tento selecionar ferramentas e frameworks que sejam padrões da indústria. É bom sabermos que, mesmo utilizando algumas ferramentas e ajudantes, sempre houve problemas não documentados durante o processo.

### CMS e Theme

Minha primeira decisão foi usar alguma ferramenta CMS para desenvolver o site. A escolha lógica foi usar Wordpress, mas como um grande adepto de ferramentas serverless achei que seria melhor, mais fácil e mais barato usar um gerador de sites estáticos como o [Hugo](https://gohugo.io/). O pacote gerado pode ser hospedado diretamente em uma CDN sem a necessidade de gerenciamento de servidores, e também há uma grande quantidade de templates para criar facilmente a estrutura do site.

A próxima tarefa é escolher o tema para o site. Eu queria que fosse visualmente simples, mas com muitas opções para personalizar. Então escolho o [FeelIt](https://feelit.khusika.com/), um tema bem completo que possui templates para páginas iniciais e de perfil (sem a necessidade de codificá-lo), configurações para vários idiomas e shortcodes estendidos para muitos tipos de diagramas. Para obter a documentação sobre o que são códigos de acesso, consulte a **[documentação oficial do Hugo](https://gohugo.io/content-management/shortcodes/)**.

### Hosting

Para o serviço de hospedagem é melhor usar um provedor de nuvem, e entre todas as opções eu selecionei AWS, que é o que usei para outros projetos anteriores. Criei os recursos manualmente, pois alguns deles não podiam ser criados por uma pilha do CloudFormation, e eu não queria usar o Terraform nem outra ferramenta externa para o deploy.

A pilha necessária é uma combinação de muitos serviços, conforme mostrado no diagrama abaixo:

<!--
{{< mermaid >}}
graph LR;
  subgraph Route53
  A(Hosted Zone)
  end
  subgraph Cloudfront
  B(Distribution)
  end
  subgraph S3
  C(evandromillian.com)
  D(www.evandromillian.com)
  end
  subgraph ACM
  E(certificate)
  end 
  subgraph Lambda
  F(trigger)
  end
  A -> F
  A -> E
  F -- process URL -> B
  B -- origin -> C & D
  E -> B
  style Route53 fill:#E05243,color:white
  style Cloudfront fill:#E05243,color:white
  style S3 fill:#F58536,color:white
  style Lambda fill:#F58536,color:white
  style ACM fill:#C00707,color:white
{{< /mermaid >}}
-->

{{< figure src="/images/website-using-hugo/stack_diagram.webp" >}}

##### S3 Storage Bucket

Vamos começar criando o bucket **S3** e configurando-o para hospedagem estática:

1. No console do **S3**, selecione a tela **Buckets** e clique no botão **Create Bucket**.
2. Defina **Nome do bucket** com o nome do seu domínio prefixado por *"www."* e selecione a região com cuidado, pois todos os outros recursos devem estar na mesma região.
3. Em seguida, desmarque a opção **Bloquear todo o acesso público** e clique na opção de reconhecimento que se abre abaixo.
4. Clique no botão **Create Bucket** e aguarde a conclusão do processo.
5. Em seguida, clique em seu novo bucket na lista de buckets e clique na guia **Propriedades**. Pesquise a seção **Hospedagem de site estático** e clique no botão Editar.
6. Na tela **Editar hospedagem de site estático**, clique na opção Ativar, defina o valor de **Index document** como *index.html* e salve as alterações.
7. Em seguida, você precisará adicionar permissões para que os arquivos contidos sejam acessados ​​a partir da página da web. Clique na guia **Permissões**, edite a seção **Política de bucket**. Adicione o seguinte texto à área de texto, substituindo DOMAIN_NAME em seu domínio (por exemplo, evandromillian.com) e salve as alterações:

```
{
	"Version": "2008-10-17",
	"Id": "S3Policy",
	"Statement": [
		{
			"Sid": "AddPerm",
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::www.DOMAIN_NAME/*"
		}
	]
}
```

Agora seu bucket está pronto para hospedar seu novo site. Se você estiver usando o **Hugo**, poderá implantar seu site diretamente da linha de comando. Adicione as seguintes linhas ao seu arquivo **config.toml**, substituindo DOMAIN_NAME e REGION pelos valores usados para seu website:

```
[[deployment.targets]]
name = ""
URL = "s3://www.DOMAIN_NAME?region=REGION"
```

Na linha de comando, digite *hugo && hugo deploy* para construir e implantar seu site no S3, respectivamente.

Para saber o URL do intervalo, selecione a guia **Propriedades** e depois a seção **Hospedagem de site estático**. Conforme a hospedagem estática estiver habilitada, a URL será exibida na seção.

##### S3 Redirect Bucket

Precisamos de outro bucket do S3 para redirecionar a chamada de não www para www:

1. No console do **S3**, selecione a tela **Buckets** e clique no botão **Create Bucket**
2. Defina **nome do bucket** com o nome do seu domínio e selecione a mesma região usada pelo primeiro bucket
3. Clique no botão **Create Bucket** e aguarde a conclusão do processo.
4. Em seguida, clique em seu novo bucket na lista de buckets e clique na guia **Propriedades**. Pesquise a seção **Hospedagem de site estático** e clique no botão Editar.
5. Na tela **Edit static website hosting**, clique na opção Enable, clique na opção *Redirect requests for an object*, defina o campo *Host name* com o nome de domínio prefixado por *"www."*, defina a opção *Protocol* como *http* e, finalmente, salve as alterações.

##### Cloudfront

A próxima etapa é configurar uma nova distribuição no **Cloudfront**. Embora o site possa ser exposto por um bucket do S3, o **Cloudfront** tem alguns recursos desejáveis:

* Permite que o site seja acessado como HTTPS, associando a ele um certificado SSL
* Como CDN, ele armazena o site em cache para pontos de presença ao redor do mundo, mais próximos dos usuários

Para criar uma distribuição do **Cloudfront**, abra o console do **Cloudfront**, clique no botão **Criar distribuição** e siga as etapas:

1. Preencha o campo **Domínio de origem** com o domínio do site
2. Na seção **Comportamento de cache padrão**, selecione a opção **Redirecionar HTTP para HTTPS**
3. Na seção **Configurações**, adicione dois **Nomes de domínio alternativos**, um como seu nome de domínio e outro com o nome de domínio prefixado por *"www."*
4. Confirme se a opção **IPv6** está ativada
5. Mantenha os outros valores como padrão e clique no botão **Criar distribuição**

Se a origem (no nosso caso, o bucket do S3) for atualizada, o **Cloudfront** precisará de 24 horas para ser atualizado. Felizmente existe uma configuração no **Hugo** config.toml para invalidar o cache. Adicione o campo *cloudFrontDistributionID* com o ID desta distribuição:

```
[[deployment.targets]]
name = ""
URL = "s3://www.DOMAIN_NAME?region=REGION"
cloudFrontDistributionID = "DISTRIBUTION_ID"
```

Depois disso, chama *hugo && hugo deploy* para atualizar as alterações de implantação no site e atualizar os caches do **Cloudfront**.

Para testar se tudo está funcionando, abra o console do **Cloudfront**, copie o *Nome de domínio* da distribuição na lista de distribuições e cole em seu navegador para ver o site aberto.

### Bug no Cloudfront

Durante os testes com a distribuição **Cloudfront**, descobri que ela não lida com o formato de URL para seções Hugo (URL sem nome de página, como index.html ou test-post.html).

Basicamente, há duas maneiras de lidar com esse problema: reescrever todos os links ou usar um **Lambda** para reescrever a URL e anexar **index.html** ao final de cada solicitação. Eu escolho pela segunda opção, porque não quero alterar os links atuais e futuros para todas as páginas. Além disso, é uma boa opção para começar a aprender sobre o Lambda@Edge e alguns de seus casos de uso.

Então vamos criar o trigger lambda nas seguintes etapas:

1. Acesse o console do **Lambda** e clique em **Criar função**
2. Use o modelo *Autor do zero*
3. **Runtime** deve ser Node.js XX, pois nosso exemplo de código foi escrito com essa linguagem
4. **Arquitetura** deve ser *x86_64* para permitir a implantação em **Lambda@Edge**
5. Em **Alterar função de execução padrão**, selecione a opção *Criar uma nova função com permissões básicas do Lambda*
6. Clique no botão **Criar função** para continuar
7. Na tela de função, cole o seguinte código na área *Fonte do código*:

```
'use strict';
exports.handler = (event, context, callback) => {
    // Extract the request from the CloudFront event that is sent to Lambda@Edge, 
    // then extract the URI from the request
    var request = event.Records[0].cf.request;
    var olduri = request.uri;

    // Match any '/' that occurs at the end of a URI. Replace it with a default index
    request.uri = olduri.replace(/\/$/, '\/index.html');
    
    // Return to CloudFront
    return callback(null, request);
};
```

8. Clique no botão *Implantar* para confirmar o código

Para que a função intercepte solicitações do **Cloudfront**, precisamos adicionar algumas permissões à sua função:

1. Na guia **Configuração**, selecione a opção **Permissões**
2. Clique no link da função dentro da seção **Execution Role** para abrir a configuração da função
3. Agora, no console do **IAM**, selecione a guia **Relações de confiança**
4. Clique no botão *Editar política de confiança* à direita da seção **Entidades confiáveis**
5. Precisamos adicionar **Lambda@Edge** como uma entidade confiável para chamar nossa função. Para isso, substitua a política pelo snippet abaixo (não pela adição de **edgelambda.amazonaws.com**):

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Service": [
					"lambda.amazonaws.com",
					"edgelambda.amazonaws.com"
				]
			},
			"Action": "sts:AssumeRole"
		}
	]
}
```

6. Clique no botão *Atualizar política* para confirmar as alterações e feche o console do **IAM**

De volta ao console do **Lambda**, podemos finalmente associar a função à distribuição do **Cloudfront**:

1. Na guia **Configuração**, selecione a opção **Acionadores** e clique no botão **Adicionar acionador**
2. Na tela **Adicionar gatilho**, selecione a opção **Cloudfront**
3. Clique no botão **Implantar no Lambda@Edge**
4. Na caixa de diálogo **Deploy to Lambda@Edge**, selecione a distribuição que deseja associar (o ID do console do **Cloudfront**), selecione *Origin request* no **Cloudfront event**, clique em ** Confirme a caixa de seleção deploy do Lambda@Edge** e finalize com o botão **Deploy**

Agora podemos testar a distribuição que está redirecionando para todas as páginas corretamente. No console do **Cloudfront**, copie o *Nome de domínio* da distribuição na lista de distribuições e cole no navegador para ver o site aberto.

##### Favicon

Para criar o favicon, usei um serviço como o **[Red Ketchup](https://redketchup.io/favicon-generator)**, que gera imagens para navegadores desktop e mobile em um único pacote. Ele também tem as opções para gerar ícone arredondado. Copie todas as imagens na pasta **static** na estrutura do seu projeto.

##### Imagens

Existem muitos sites que disponibilizam imagens gratuitas para os posts. Usei **[StockFreeImages](https://www.stockfreeimages.com/)** para algumas imagens.

### Conclusão

Espero que isso ajude a criar e implantar um site do **Hugo** de maneira fácil. Futuramente podemos usar uma pilha **CloudFormation** para criar toda a estrutura na AWS, bem como comparar a complexidade ao usar outros provedores como Google, Azure e Digital Ocean.

### Referências

* [Hugo forum post with error not finding index.html page](https://discourse.gohugo.io/t/on-path-page-not-found-but-index-html-it-opens/27587/13)
* [Implementing Default Directory Indexes in Amazon S3-backed Amazon CloudFront Origins Using Lambda@Edge](https://aws.amazon.com/blogs/compute/implementing-default-directory-indexes-in-amazon-s3-backed-amazon-cloudfront-origins-using-lambdaedge/)
* [Cloudformation template for Cloudfront hosted Hugo website](https://github.com/keaeriksson/hugo-s3-cloudfront)
* [Using GitHub Actions and Hugo Deploy to Deploy a Static Site to AWS](https://capgemini.github.io/development/Using-GitHub-Actions-and-Hugo-Deploy-to-Deploy-to-AWS/)