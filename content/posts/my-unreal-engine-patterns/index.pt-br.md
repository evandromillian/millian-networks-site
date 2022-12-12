---
title: "Meus Padrões da Unreal Engine"
date: 2022-12-11T14:15:08-03:00
description: ""
draft: false
author: "Evandro Millian"
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image-preview.webp"
tags: ["Unreal Engine"]
categories: ["Game Development"]
lightgallery: true
---

A versão mais recente do Unreal Engine está se tornando o mecanismo mais poderoso disponível ao público. Ele contém muitas melhorias apresentadas ao longo dos anos em artigos e conferências como o GDC.

Mas, por ser muito grande, é importante conhecer algumas boas práticas para poder produzir mais em menos tempo, ou ter menos problemas ou erros. Esses são os padrões básicos que uso para meus projetos do Unreal Engine.

Esta lista será atualizada com base nos aprendizados realizados.

### Dicas Gerais

* Sempre comente seu código, seja por snippet, na declaração (C++) ou na declaração do evento/função (Blueprint)
* Eventos/funções devem ter uma única responsabilidade

### Estrutura de Objetos

* Preste atenção com referências fortes em plantas
     * Cada referência a um tipo concreto de Blueprint é um aumento de memória
     * Tente mover o tipo de referência o mais próximo possível dos tipos integrados do Unreal, como **Character**, **Actor**, **ActorComponent**, etc.
     * Use **Soft References** para as classes e carregue-as em tempo de execução usando um dos métodos abaixo:
         * **Async Load Class Asset** precisa de menos configuração para usar
         * **Async Load Primary Asset** precisa de mais configuração, mas é mais flexível

### Blueprints

* Não use execuções de entrada múltipla
     * O que pode acontecer se você precisar substituir o evento **Initialize** em um blueprint filho? O evento **BeginPlay** executará o código vinculado, não o evento **Initialize**. Isso levará a erros no blueprint dos filhos.

    {{<figure src="/images/unreal-engine-patterns/fork.webp" >}}

    * Evite, tanto quanto possível, ter execuções de entrada múltipla. Em código real (que provavelmente será muito maior), será difícil ter certeza sobre o fluxo do código e o resultado do evento/função. Simplicidade é a chave, sempre.
    
* Refatorar um evento complexo em um mais simples
     * O nó **Sequence** é ótimo para organizar o fluxo do seu evento.
     * Divida grandes eventos em vários eventos/funções menores
     * O código repetido também deve ser movido para sua própria função
* Não tenha medo de usar **Set Timer By Event/Function Name** em funções
     * É melhor para a organização, pois o delegado estará em seu próprio evento/função
* Use entrada múltipla para um executivo com cautela, evite na maioria das vezes
     * Isso pode causar confusão, especialmente se um dos eventos for substituído
* Prefira **DoOnce** em vez de sinalizadores booleanos
     * Se **DoOnce** for aberto/fechado em muitos lugares, crie eventos personalizados chamando **Open** e **Close** em vez de usar links **Exec** longos.
* Prefira **Select** em vez de **Branch** para selecionar valores
     * Além da simplificação do fluxo de código, **Select** permite lidar com vários valores e é configurável para muitos tipos de dados (mesmo enums).

{{<figure src="/images/my-unreal-engine-patterns/selects.png" width=600 >}}

### C++

* Implemente a maior parte do comportamento em C++, expondo pontos de extensão ao Blueprint, como exemplos:
     * Funcionalidade básica de **Character**
     * **Actor Components**
     * Integrações com serviços online para controle de contas
     * Expor ao Blueprints algumas funcionalidades que só estão disponíveis em C++
     * Envolva tarefas de longa duração em **latents functions** (vinculadas a Atores) e **async tasks** (instâncias de **BlueprintAsyncActionBase**, que são reutilizáveis)
* **DuplicateObject** copia apenas variáveis de mercado com **UPROPERTY()**
* Para criar uma função que tenha mais de um valor de retorno, basta adicionar os argumentos da função por referência

{{< highlight cpp >}}

UFUNCTION(BlueprintCallable)
float ReturnMultipleValues(FString& TextRet);

{{< /highlight >}}

### C++ to Blueprint Magic

* Use várias entradas (**UFUNCTION** com meta **ExpandEnumAsExecs**) para reduzir a necessidade de nós **Switch** e **Branch** no código Blueprint

{{< highlight cpp >}}
UENUM()
namespace EAuthorityType
{
	enum Type
	{
		Authority,
		Remote,
		Standalone
	};
}

...

UFUNCTION(BlueprintCallable, meta = (ExpandEnumAsExecs = "OutResult"))
void SwitchAuthorityType(AActor* Actor, TEnumAsByte<EAuthorityType::Type>& OutResult);
{{< /highlight >}}

{{<figure src="/images/my-unreal-engine-patterns/multiple_outputs.png" width=400 >}}

* Use **DetermineOutputType** com **DynamicOutputParam** para atualizar dinamicamente o tipo de saída da função BP, sem a necessidade de escrever um K2Node personalizado

{{< highlight cpp >}}
UFUNCTION(BlueprintCallable, meta=(DeterminesOutputType="ActorClass"))
AActor* GetActorOfClass(const UObject* WorldContextObject, TSubclassOf<AActor> ActorClass);

UFUNCTION(BlueprintCallable, meta=(DeterminesOutputType="ActorClass", DynamicOutputParam="OutActors"))
void GetAllActorsOfClass(TSubclassOf<AActor> ActorClass, TArray<AActor*>& OutActors);
{{< /highlight >}}

### Widgets

* Implemente otimizações da documentação do Unreal Engine.
     * Use atualizações orientadas a eventos em vez de atributos limitados
     * Divida widgets complexos em outros menores
     * Use **Spacers** em vez de **Size Boxes** quando possível
     * Evite combinar **Scale Boxes** e **Size Boxes**
     * ~~Use widgets de Rich Text com moderação~~ Usando **Common Text**
     * Fique atento aos custos de animação conforme tabela a seguir:

Custos                        | Exemplos
------------------------------|--------------------------------------------------------------
Sem custo de CPU/menor custo  | Animação controlada via material
Baixo custo de CPU            | Animações com script de blueprint que não requerem **Sequencer**.
Alto custo de CPU             | Animações do sequenciador feitas com o editor de animação da **UMG**.
Altíssimo custo de CPU        | Animações que causam alterações de layout.

* Sobre animações, usaremos transições **CommonActivatableWidgetStack** na maioria das vezes.

### Automation and Validations

* Para melhorar a qualidade dos assets, reduzir erros e aumentar a velocidade de desenvolvimento, o Unreal Engine disponibiliza algumas ferramentas como:
    * [Asset Validators](https://www.youtube.com/watch?v=zRZjNN6jxCI)
    * [Asset/Actor Action Utilities](https://www.youtube.com/watch?v=wJqOn88cU7o&list=PLoObU30LCLpcItHySX3dpP02CDUZw4XNk&t=926s)
    * [Blutility Script Buttons](https://www.youtube.com/watch?v=wJqOn88cU7o&list=PLoObU30LCLpcItHySX3dpP02CDUZw4XNk&t=1360s)
    * [Editor Utility Widgets](https://www.youtube.com/watch?v=wJqOn88cU7o&list=PLoObU30LCLpcItHySX3dpP02CDUZw4XNk&t=1604s)