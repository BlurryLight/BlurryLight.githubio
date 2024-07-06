
---
title: "UE Programmatically Modifying Material Nodes"
date: 2023-06-11T12:56:00+08:00
draft: false 
categories: [ "UE"]
isCJKLanguage: true
slug: "64b1b73b"
toc: true
mermaid: false
fancybox: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---

Note: I've tested the code on UE5 as well. There are some minor API changes, but the logic still applies.

# Introduction

Recently, I've a need to create a fairly complicated material in a **programmatically** way.
Even if the desired material is created by Tech-Artists, I need to modify some nodes in it **automatically**.

After some searching, there is already a great article about **Adding** nodes to material, see 

> Ref：[UE4 - Programmatically create a new material and inner nodes - Isara Tech.](https://isaratech.com/ue4-programmatically-create-a-new-material-and-inner-nodes/)

However, there is a lack for **Modifying** nodes in material. 


# Adding Nodes

Go for [UE4 - Programmatically create a new material and inner nodes - Isara Tech.](https://isaratech.com/ue4-programmatically-create-a-new-material-and-inner-nodes/) firstly. It's a good starting-point.

Also, there is some reference code in Engine plugins, some dataformat import plugins face same problem: they need to convert material infos parsed from dataformat to UE material node graph.

Source file `Engine/Plugins/Enterprise/DatasmithImporter/Source/DatasmithImporter/Private/DatasmithMaterialExpressions.cpp` has many examples, especially 
`FDatasmithMaterialExpressions::AddCroppedUVMappingExpression` and many others.

Technically, the logic for adding nodes is quite simple:
- Create a `UMateiralExpression*` with the appropriate Outer, usually the material itself
- Assign necessary properties about the node to the `UMateiralExpression*`
- Add the `Expression` to the mat with `Mat->Expressions.Add(Expression)`
- Connect it with other input/output pins by`UMateiralExpression->ConnectExpression` 


# Modifying Nodes

`Modifying Nodes` is a bit complicated.
By `Modifying Nodes`, I'm not referring to changing the properties of the nodes(which is trivial), but changing the node itself. For example, replacing a `Multiply` node with a `Add` node, or more complicated, replacing a `Texture2D` node with a `TextureCube` node, which requires adding some auxiliary nodes to get it to work.

The problem is that by replacing requires us to **delete** the old node and break its links, **create** a new node, and **reconnect** the pins to the new node.


## Graph & Expression

To manipulate nodes and pins, firstly we need basic understanding about the underlying structures in Material Bluepring. 
It seems there are two data structures for material node graph: `MaterialGraph` and `Expression`.
The `Expression` may be more low-level.
And the two of them can sync with each other.

- From Dirty Expression Build MateiralGraph

```cpp
if (!Mat->MaterialGraph)
{
    Mat->MaterialGraph = CastChecked<UMaterialGraph>(FBlueprintEditorUtils::CreateNewGraph(Mat, NAME_None, UMaterialGraph::StaticClass(), UMaterialGraphSchema::StaticClass()));
    Mat->MaterialGraph->Material = Mat;
    Mat->MaterialGraph->RebuildGraph();
}
```

- From Dirty Mateiral Graph build Expression

```cpp
Mat->MaterialGraph->LinkMaterialExpressionsFromGraph();
```

I found it is much simpler to manipulate pins connections in `MaterialGraph` Level, since there are some utility functions from UE for it. 
But for create nodes, we have to do that in `Expression` Level. so the sync functions between them are needed.


# Replacing Texture2D nodes with TextureArray nodes

Before Replacing:
![UE-Programmatically-modifying-material-nodes-2023-06-11-13-37-00](https://img.blurredcode.com/img/UE-Programmatically-modifying-material-nodes-2023-06-11-13-37-00.png?x-oss-process=style/compress)

After Replacing:(I mannually dragged some nodes to make it clear)
![UE-Programmatically-modifying-material-nodes-2023-06-11-13-45-47](https://img.blurredcode.com/img/UE-Programmatically-modifying-material-nodes-2023-06-11-13-45-47.png?x-oss-process=style/compress)

the key code as follows:

Step 0: Scan all Texture2D nodes

```cpp
	TArray<UMaterialExpressionTextureSampleParameter* >  ParamsTextureSamples;
	for (int32 ExpressionIndex = 0; ExpressionIndex < Mat->Expressions.Num(); ExpressionIndex++)
	{
		auto* ExpressionPtr = Cast<UMaterialExpressionTextureSampleParameter>(Mat->Expressions[ExpressionIndex]);
		if (ExpressionPtr)
		{
			ParamsTextureSamples.Add(ExpressionPtr);
		}
	}
```


Step 1: creating Array node, and using `MaterialGraph`-related functions to break old pins and replace the new node

```cpp
UMaterialExpression* TexCoordExpression = TargetExpression->Coordinates.Expression;
UMaterialExpressionTextureSampleParameter2DArray* Tex2DArrayExpression=
    NewObject<UMaterialExpressionTextureSampleParameter2DArray>(Mat);
Tex2DArrayExpression->ParameterName = *FString::Printf(TEXT("%s_Arr"),*TargetExpression->ParameterName.ToString());
Tex2DArrayExpression->SamplerType = SAMPLERTYPE_Color;
Tex2DArrayExpression->Group = TargetExpression->Group;
Tex2DArrayExpression->Texture = DummyTex;
Tex2DArrayExpression->MaterialExpressionEditorX = TargetExpression->MaterialExpressionEditorX;
Tex2DArrayExpression->MaterialExpressionEditorY = TargetExpression->MaterialExpressionEditorY - 96;
Mat->MaterialGraph->AddExpression(Tex2DArrayExpression,/*bUserInvoke*/ false);
Mat->Expressions.Add(Tex2DArrayExpression);

auto Tex2DArrayGraphNode = Cast<UMaterialGraphNode>(Tex2DArrayExpression->GraphNode);

Tex2DArrayGraphNode->ReplaceNode(GraphNode);
TArray<UEdGraphPin*> Tex2DArrayGraphInputPins;
Tex2DArrayGraphNode->GetInputPins(Tex2DArrayGraphInputPins); 
// break the uv pin input because we need to add auxiliary nodes after.
// because float2 uv will not compile when connecting a pin with float3
GraphNode->GetGraph()->GetSchema()->BreakPinLinks(*Tex2DArrayGraphInputPins[0],true);

GraphNode->GetGraph()->GetSchema()->BreakNodeLinks(*GraphNode);
Mat->MaterialGraph->LinkMaterialExpressionsFromGraph();
```

Step 2: create auxiliary nodes

![UE-Programmatically-modifying-material-nodes-2023-06-11-13-50-59](https://img.blurredcode.com/img/UE-Programmatically-modifying-material-nodes-2023-06-11-13-50-59.png?x-oss-process=style/compress)

```cpp
auto * LayerExpression= Cast<UMaterialExpressionConstant>(UMaterialEditingLibrary::CreateMaterialExpression(Mat,UMaterialExpressionConstant::StaticClass()));
LayerExpression->R = 0.0;

FString FunctionString = TEXT("/Engine/Functions/Engine_MaterialFunctions02/Utility/BreakOutFloat2Components.BreakOutFloat2Components");
UMaterialFunction*  BreakOutFloat2 = LoadObject<UMaterialFunction>(nullptr, *FunctionString, nullptr, LOAD_None, nullptr);
UMaterialExpressionMaterialFunctionCall* BreakOutFloat2Expression=
    Cast<UMaterialExpressionMaterialFunctionCall>(UMaterialEditingLibrary::CreateMaterialExpression(Mat,UMaterialExpressionMaterialFunctionCall::StaticClass()));
BreakOutFloat2Expression->MaterialFunction = BreakOutFloat2;
BreakOutFloat2Expression->UpdateFromFunctionResource();

TexCoordExpression->ConnectExpression(BreakOutFloat2Expression->GetInput(0),0);

FunctionString = TEXT("/Engine/Functions/Engine_MaterialFunctions02/Utility/MakeFloat3.MakeFloat3");
UMaterialFunction*  MakeFloat3= LoadObject<UMaterialFunction>(nullptr, *FunctionString, nullptr, LOAD_None, nullptr);
UMaterialExpressionMaterialFunctionCall* MakeFloat3Expression =
    Cast<UMaterialExpressionMaterialFunctionCall>(UMaterialEditingLibrary::CreateMaterialExpression(Mat,UMaterialExpressionMaterialFunctionCall::StaticClass()));
MakeFloat3Expression->MaterialFunction = MakeFloat3;
MakeFloat3Expression->UpdateFromFunctionResource();

BreakOutFloat2Expression->ConnectExpression(MakeFloat3Expression->GetInput(0),0); // u
BreakOutFloat2Expression->ConnectExpression(MakeFloat3Expression->GetInput(1),1); // v
LayerExpression->ConnectExpression(MakeFloat3Expression->GetInput(2),0); // w

MakeFloat3Expression->ConnectExpression(&(Tex2DArrayExpression->Coordinates),0); // uvw
```

Step4:  remove the old unneeded Texutre2D nodes and rebuild graph

```cpp
Mat->Expressions.Remove(GraphNode->MaterialExpression);
Mat->RemoveExpressionParameter(GraphNode->MaterialExpression);

Mat->MaterialGraph->RebuildGraph();
```


# Reference:

full code available:

<script src="https://gist.github.com/BlurryLight/ec74297784e614e2cf005e86a9c64460.js"></script>