---
date: '2023-02-20'
menu:
  corda-5-beta:
    identifier: corda-5-beta-cdl
    parent: corda-5-beta-tools-add-ons
    weight: 1000
section_menu: corda-5-beta
title: "CorDapp Design Language (CDL)"
---

The CorDapp Design Language (CDL) is a set of diagrammatic views you can use to concisely and accurately guide the design of your CorDapp. Using CDL in the design phase of your CorDapp allows you to:

* Represent and reason about core aspects of the Corda design including: Ledger data, shared logic, permissioning, and privacy.
* Reason about more complex CorDapp designs through abstraction from the code and standardisation.
* Produce complete, well thought-out CorDapp designs.
* Document standardised design patterns.
* Degrade gracefully. You do not always have to put in all of the detail, but it should be consistent at the level of detail you choose.

## Who Should use the CDL

CorDapp Design Language is for anyone who wants to articulate and understand non-trivial CorDapp designs. This may include:
* CorDapp Architects
* CorDapp Developers
* Business Analysts
* Application Testers
* Quality Assurance Analysts
* Security Analysts
* Technical Auditors

It can also be used to communicate to non-technical audiences, for example in marketing materials, though most likely in a simplified form.

## Main CDL views

There are three main CDL views, each of which serves a different purpose in communicating a CorDapp design:

* [Smart Contract view](#smart-contract-view) — the primary view of CDL, which represents the design of your Corda smart contract.
* [Ledger Evolution view](#ledger-evolution-view-with-privacy-overlay) — a view of the smart contract in action, with representations of transactions and states on the Ledger.
* [Business Process Modelling Notation (BPMN) views](#business-process-modelling-notation-bpmn-view) — shows the process flows of your CorDapp.

<!--In the sections below, you will find an overview of each CDL view. For a more in depth guide, take a look at the [CDL views documentation](../../../en/tools/cdl/cdl-views.md).-->

### Smart Contract View

This is the primary view of CDL. It represents the design of your Corda smart contracts. In Corda, smart contracts consist of one or more Corda states, which represent the data going onto the Ledger, together with a Corda contract that restricts what can be done with the data in those states.

#### Example Smart Contract View

{{< figure zoom="cdl-agreement-smart-contract-full.png" width="800" title="Click to zoom image in new tab/window" >}}

This diagram shows the three statuses an agreement can have, together with the possible transitions from one status to another, and the constraints governing both statuses and transitions.

The black filled circle on the left represents the initial, uncreated, state of an agreement, which is brought into existence by being proposed, while the circle on the right represents the terminal, completed state in which the agreement has been fulfilled.

The base assumption behind the view is that Corda States can be in a number of *statuses* which might represent, for example, different stages on a multiparty workflow. We define rules for how the smart contract can move between statuses and the constraints which must be satisfied with each transition.

This view is conceptually modelled on a Finite State Machine. The classic example of which is a washing machine that has two states: **Full of water** and **Empty**. The Finite State Machine of the washing machine should have the constraint that the user can only execute the **Open door** command when the washing machine is in the **Empty** State. 

For simple CorDapp smart contracts, there may only be one, implicit status. You can still use the Smart Contract view to communicate the design, it just devolves down to a diagram with only one state box.

<!--A detailed explanation of the elements which make up the Smart Contract view can be found [here](../../../en/tools/cdl/smart-contract-view/cdl-smart-contract-view.md).

A Lucidchart template with the CDL Smart Contract view standard shapes [can be found here](https://app.lucidchart.com/invitations/accept/6adacd29-482f-45ca-9bdd-57252d64c8fc).

{{< note >}}
You need an active Lucidchart account to access this folder.
{{< /note >}}

In addition, the section [CDL to Code](../../../en/tools/cdl/cdl-to-code/cdl-to-code.md) section shows a standardised way to transform the CDL Smart Contract view into code.-->

### Ledger Evolution View (with Privacy overlay)

If the Smart Contract view defines the design of a smart contract, the Ledger Evolution view shows that smart contract in action.

The Corda Ledger can be considered as a Directed Acyclic Graph (DAG) of states and transactions connected by input/output relationships. States are generated as outputs to a transaction, and go on to be consumed as the inputs of other transactions. The transactions have associated properties, such as the commands and who signed them.

You can not show the whole Corda Ledger in a single view (and it would not be useful to try). However, it is useful to show a sub-graph of the wider Ledger which concerns a particular set of transitions for the smart contract in question, and that is what the Ledger view does.

#### Example Ledger Evolution View

{{< figure zoom="cdl-agreement-naive-billing-ledger-evolution-charlie.png" width="800" title="Click to zoom image in new tab/window" >}}

In this example, both states and transactions are represented as "nodes" in the graph. There is a connecting arrow from a transaction to a state when the state is an output of the transaction, and from a state to a transaction when the state is consumed by that transaction.

The Ledger Evolution view also has an additional Privacy overlay represented as the purple and orange lines in the above diagram. 
Each overlay is from the perspective of a Party on the ledger. 
It starts with a state which the Party is a participant on and then traces back through the DAG to show all the previous transactions the Party's node will receive a copy of through the transaction resolution mechanism. 
The green circles indicate that privacy has been preserved, where as the red circles indicate a privacy leak.

With this view, you have insight on privacy from the design phase. An unintended privacy leak can be a show stopper for a CorDapp. It is best to consider this early on, not at the point when your customer's information security team will not sign off the deployment.

<!--A detailed explanation of the elements which make up the Ledger Evolution view can be found [here](../../../en/tools/cdl/ledger-evolution-view/cdl-ledger-evolution-view.md).

A Lucidchart template with the CDL Ledger Evolution view standard shapes can be found [here](https://app.lucidchart.com/invitations/accept/6adacd29-482f-45ca-9bdd-57252d64c8fc).-->


### Business Process Modelling Notation (BPMN) View

The Business Process Modelling Notation view is, as the name suggests, a BPMN diagram showing the process flows of your CorDapp. You can use this view to describe the macro Business Process flows or, alternatively, the back and forth communication within a Corda flow.

#### Example BPMN View

{{< figure zoom="cdl-bpmn-simultaneous-action.png" width="750" title="Click to zoom image in new tab/window" >}}

The notation follows BPMN v2 standards, with a few small additions:

* Where a Corda transaction is written to the Ledger, this is represented as a simultaneous task/action on the swim lane of every participant to the transaction. Each state is marked with a small Corda logo.
* The initiator of the transaction should be indicated on the version of the action that appears in the initiator's swim lane.
* You can add an optional blue box to show further salient details about the transaction, such as the name of the flow which triggered it or the command used in the transaction.

<!--Lucidchart has a standard Shapes library for BPMN 2.0 diagrams, in addition a Lucidchart template with the CDL Ledger Evolution view standard shapes can be found [here](https://app.lucidchart.com/invitations/accept/6adacd29-482f-45ca-9bdd-57252d64c8fc).

A detailed explanation of the elements which make up the BPM view can be found [here](../../../en/tools/cdl/bpmn-view/cdl-bpmn-view.md).-->
