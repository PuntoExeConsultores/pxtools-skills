# PXFlowController — Flow Orchestration Pattern

## What it is

PXFlowController generates **workflow WebPanels** orchestrating sequences of actions, confirmations and iterations. It turns complex flow logic (which normally takes a lot of hand-written code) into a declarative XML definition.

## Parent Objects

It can be attached to:
- `Procedure` — a flow following a process
- `Transaction` — a flow following a transaction
- `(None)` — a standalone flow

## Objects it generates

From **a single instance**, it generates:

| Object | GeneXus type | Naming | Condition |
|--------|--------------|--------|-----------|
| FlowController (Desktop) | WebPanel | `Ct{Element.name}` | Always |
| FlowControllerResponsive | WebPanel | `Ct{Element.name}` | Always |

Each `level` in the instance generates a pair of WebPanels (Desktop + Responsive).

## XML structure of the instance

```xml
<instance>
  <!-- Root attributes -->
  <!-- templatesGroup: the template group to apply -->

  <level name="MyFlow" description="Invoicing flow">
    <!-- Level properties -->
    <!-- name: the flow's name -->
    <!-- description: a description -->
    <!-- masterPage: the MasterPage to use (<default> or a reference) -->
    <!-- theme: the Theme to use (<default> or a reference) -->
    <!-- isGXFlowTask: bool — GXFlow integration -->
    <!-- generateWeb: enum{<default>;True;False} -->
    <!-- generateWebResponsive: enum{<default>;True;False} -->
    <!-- generateSD: enum{<default>;True;False} -->
    <!-- templateObject: WebPanel template (Desktop) -->
    <!-- templateResponsiveObject: WebPanel template (Responsive) -->
    <!-- templateSDObject: SDPanel template (Smart Devices) -->

    <parameters>
      <parameter name="InvoiceId" />
      <parameter name="CustomerId" />
    </parameters>

    <blocksLevels>
      <blocksLevel name="Main">
        <!-- A block of lines: each line is a step of the flow -->
        <linesBlock lineFrom="1">
          <mainCode>
            <![CDATA[
              // GeneXus code executed on line 1
              &InvoiceTotal = GetInvoiceTotal(&InvoiceId)
            ]]>
          </mainCode>

          <actions>
            <action name="Confirm"
                    callType="Link"
                    linkType="PXInstance"
                    instanceObject="PXParameterRequestConfirmInvoice"
                    instanceLevel="Level1"
                    instanceLevelNode="Level"
                    target="New"
                    popupWidth="400"
                    popupHeight="300"
                    actionLine="1"
                    nextLine="2">
              <parameters>
                <parameter name="InvoiceId" />
              </parameters>
            </action>

            <action name="Cancel"
                    callType="Subroutine"
                    subroutine="CancelFlow"
                    actionLine="1"
                    nextLine="99">
            </action>
          </actions>

          <confirms>
            <confirm name="ConfirmSend"
                     question="Do you want to send the invoice?"
                     questionType="Constant"
                     popupWidth="350"
                     popupHeight="200"
                     confirmLine="1"
                     nextLine="3">
              <responses>
                <response responseValue="Yes" responseLine="2" />
                <response responseValue="No" responseLine="1" />
              </responses>
            </confirm>
          </confirms>
        </linesBlock>

        <linesBlock lineFrom="2">
          <mainCode><![CDATA[
            // Line 2: send the invoice
            SendInvoice(&InvoiceId)
          ]]></mainCode>
        </linesBlock>
      </blocksLevel>
    </blocksLevels>

    <codes>
      <code type="Start" data="...startup code..." />
      <code type="Refresh" data="...refresh code..." />
      <code type="Subroutine" name="CancelFlow" data="...code..." />
    </codes>

    <variables>
      <variable name="InvoiceTotal" basedOn="InvoiceTotal" />
    </variables>
  </level>
</instance>
```

## The concept of line blocks

The flow is organised into **numbered lines**. Each block of lines (`linesBlock`) defines:

```
┌─────────────────────────────────────────────┐
│ Line block (lineFrom=1)                     │
│                                             │
│  1. mainCode: the GeneXus code to run       │
│  2. iterationCode: iteration code           │
│     (optional, for loops)                   │
│  3. actions: the available actions          │
│     └─ each action has a nextLine           │
│  4. confirms: confirmations                 │
│     └─ each response has a responseLine     │
│                                             │
│  The flow jumps to nextLine/responseLine    │
│  according to the action or response chosen │
└─────────────────────────────────────────────┘
```

### Execution flow

```
Line 1 ──► mainCode ──► Actions/Confirms
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
            Action A   Action B    Confirm
            nextLine=2 nextLine=3   │
                │          │    ┌───┴───┐
                ▼          ▼    ▼       ▼
             Line 2     Line 3 Yes     No
                              resp=4  resp=1
                               │        │
                               ▼        ▼
                            Line 4   Line 1
                                    (goes back)
```

## Action types (callType)

| callType | Description | Key properties |
|----------|-------------|----------------|
| `Link` | Invokes an object or an instance | linkType, gxObject/instanceObject, target |
| `External Link` | A link to an external URL/object | externalObject |
| `Client Text Print` | Client-side text printing | gxObject |
| `Subroutine` | Invokes a local subroutine | subroutine |

### linkType for Link actions

| linkType | Description | Property |
|----------|-------------|----------|
| `GXObject` | Calls a GeneXus object directly | gxObject (reference) |
| `PXInstance` | Calls a PXTools instance | instanceObject + instanceLevel + instanceLevelNode |

### Patterns invocable through PXInstance

An action with `linkType="PXInstance"` can invoke:
- **PXWorkWith** (Selection, View, Edit, Prompt)
- **PXParameterRequest** (Level)
- **PXComposer** (Level)
- **PXFlowController** (another flow)
- **PXOAV** (OAV management)

### Action target

| target | Behaviour |
|--------|-----------|
| `Self` | Navigates in the same window |
| `New` | Opens in a popup/new window |

When `target="New"`:
- `popupWidth` / `popupHeight` control the size
- `closeWindowControl` lets you control the popup's closing
- `closeWindowControlCondition` evaluates whether the close is accepted
- `hasPostCode` enables code that runs after the popup closes

## Confirmations

Confirmations (`confirm`) are modal dialogs with several possible responses:

```xml
<confirm name="ConfirmAction"
         question="Are you sure?"
         questionType="Constant"    <!-- or Variable -->
         gxObject="WbConfirm"       <!-- a custom confirmation WebPanel (optional) -->
         responseDomain="MyDomain"  <!-- the domain of the responses -->
         confirmLine="1"
         nextLine="2">
  <responses>
    <response responseValue="Accept" responseLine="3" />
    <response responseValue="Cancel" responseLine="1" />
  </responses>
</confirm>
```

- They can use a custom WebPanel (`gxObject`) or the standard PXTools popup
- Each response directs the flow to a different line
- `responseLine="1"` in the response blocks is reserved for validation

## Iterations

The `iterationCode` node allows loops inside the flow:

```xml
<linesBlock lineFrom="5">
  <iterationCode iterationLineNext="5">
    <![CDATA[
      // This code runs on each iteration
      // If iterationLineNext=5 (the same line), it iterates
      For each Invoice where InvoiceStatus = "Pending"
    ]]>
  </iterationCode>
  <mainCode>
    <![CDATA[
      // The iteration's main code
      ProcessInvoice(InvoiceId)
    ]]>
  </mainCode>
</linesBlock>
```

## Code hooks (Codes)

| type | When it runs |
|------|--------------|
| `Start` | At the start of the WebPanel (the Start event) |
| `Refresh` | In the Refresh event |
| `Load` | In the Load event |
| `Subroutine` | A subroutine callable by name |

## Variables

The flow's variables are declared with a complete definition:

```xml
<variable name="Total"
          basedOn="InvoiceTotal"    <!-- based on an attribute -->
          domain="Numeric10_2"      <!-- or based on a domain -->
          SDT="MySDT"               <!-- or based on an SDT -->
          dataType="Numeric"        <!-- or a primitive type -->
          length="10"
          decimals="2"
          collection="false"
          dimension="Scalar" />
```

## Dual-platform capability

Every level has `generateWeb`, `generateWebResponsive` and `generateSD` properties. The .Pattern defines two Object entries per level:

- `FlowController` → generates a WebPanel with the `FlowControllerWebForm.dll` template (HTML)
- `FlowControllerResponsive` → generates a WebPanel with the `FlowControllerAbstractForm.dll` template (abstract)

## Example: continuous creation with confirmation (PXFlowControllerContinuousCreate)

This instance orchestrates **creating records continuously**: after creating one, it asks whether another of the same kind should be created, and chains the navigation according to the answer.

```xml
<?xml version="1.0" encoding="utf-16"?>
<instance>
  <level name="ContinuousCreate" generateWebResponsive="True">
    <blocksLevels>
      <blocksLevel>
        <!-- Line 1: create another record of the same kind? -->
        <linesBlock lineFrom="1">
          <mainCode><![CDATA[
            &SDTContinuousCreate = RetSDTContinuousCreate.Udp()
            If not &SDTContinuousCreate.Enabled
              ControllerGotoLine 4          // Go straight to the Selection
            Else
              ControllerConfirm CreateAgain // Ask the user
            EndIf
          ]]></mainCode>

          <confirms>
            <confirm name="CreateAgain"
                     gxObject="PuConfirm, PXTools.APIs"
                     responseDomain="Boolean"
                     question="Do you want to create another record of the same kind?"
                     nextLine="1">
              <responses>
                <response responseValue="True" responseLine="2" />   <!-- Yes: create a new one -->
                <response responseValue="False" responseLine="3" />  <!-- No: go to the Selection -->
              </responses>

              <!-- Line 2: create the record and navigate to the View -->
              <linesBlock lineFrom="2">
                <mainCode><![CDATA[
                  AddOrder.Call(&Context.SecurityUserTenantId, 0,
                    &SDTContinuousCreate.Kind, ...)
                  For Each
                    Where TenantId = &Context.SecurityUserTenantId
                    Where KindId = &SDTContinuousCreate.Kind
                    If KindSkipsHeaderData
                      ControllerAction GoToViewDetail
                    Else
                      ControllerAction GoToViewHeader
                    EndIf
                  EndFor
                ]]></mainCode>
                <actions>
                  <!-- An action invoking the PXWorkWith View, the Header section -->
                  <action name="GoToViewHeader"
                          instanceObject="PXWorkWithOrders, MyApp"
                          instanceLevel="Order"
                          instanceLevelNode="View"
                          instanceLevelViewSection="Header"
                          nextLine="999">
                    <parameters>
                      <parameter name="TrnMode.Insert" />
                      <parameter name="0" />
                      <parameter name="&EmptyStatus" />
                    </parameters>
                  </action>
                  <!-- An action invoking the PXWorkWith View, the Detail section -->
                  <action name="GoToViewDetail"
                          instanceObject="PXWorkWithOrders, MyApp"
                          instanceLevel="Order"
                          instanceLevelNode="View"
                          instanceLevelViewSection="Detail"
                          nextLine="999">
                  </action>
                </actions>
              </linesBlock>

              <!-- Line 3: no repeat, go to the Selection -->
              <linesBlock lineFrom="3">
                <mainCode><![CDATA[
                  DelSDTContinuousCreate.Call()
                  ControllerAction GoToSelection
                ]]></mainCode>
                <actions>
                  <action name="GoToSelection"
                          instanceObject="PXWorkWithOrders, MyApp"
                          instanceLevel="Order"
                          instanceLevelNode="Selection"
                          nextLine="999" />
                </actions>
              </linesBlock>
            </confirm>
          </confirms>
        </linesBlock>

        <!-- Line 4: go to the Selection (when continuous creation is disabled) -->
        <linesBlock lineFrom="4">
          <mainCode><![CDATA[ControllerAction GoToSelection]]></mainCode>
          <actions>
            <action name="GoToSelection"
                    instanceObject="PXWorkWithOrders, MyApp"
                    instanceLevel="Order"
                    instanceLevelNode="Selection"
                    nextLine="1" />
          </actions>
        </linesBlock>
      </blocksLevel>
    </blocksLevels>
    <variables>
      <variable name="SDTContinuousCreate" SDT="SDTContinuousCreate, MyApp" />
      <variable name="EmptyStatus" basedOn="OrderStatus" />
      <variable name="EmptyKind" basedOn="OrderKind" />
    </variables>
  </level>
</instance>
```

### Diagram of the flow

```
Line 1: is continuous creation enabled?
    │
    ├── NO ──► Line 4 ──► GoToSelection (PXWorkWithOrders)
    │
    └── YES ──► Confirm "Create another record of the same kind?"
                 │
                 ├── True (Line 2) ──► create the record ──►
                 │     ├── if it skips the header ──► View.Detail
                 │     └── otherwise ──► View.Header
                 │
                 └── False (Line 3) ──► clear the state ──► GoToSelection
```

**Key points of the example:**
- It uses `ControllerGotoLine` to jump lines without any user action
- It uses `ControllerConfirm` to show a confirmation dialog
- It uses `ControllerAction` to run a defined action
- The actions reference PXWorkWith with `instanceLevelViewSection` to navigate to a specific tab
- `generateWebResponsive="True"` generates both the Desktop and Responsive versions

## Relationship with the other patterns

```
PXFlowController
├── Can invoke ──► PXWorkWith (Selection/View/Prompt)
├── Can invoke ──► PXParameterRequest (confirmations, data capture)
├── Can invoke ──► PXComposer (composed screens)
├── Can invoke ──► PXOAV (dynamic attribute management)
├── Can invoke ──► another PXFlowController (nested flows)
└── Can invoke ──► any GeneXus object (Procedures, WebPanels)
```
