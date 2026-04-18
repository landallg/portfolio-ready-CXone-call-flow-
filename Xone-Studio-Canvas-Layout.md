# CXone Studio Canvas Layout – ACME Support Call Flow

This document shows the full block-by-block Studio flow for a production-ready ACME Support call flow. It demonstrates menu routing, data dips, callback logic, error handling, and tier-based support routing.

## Overview

## High-Level Flow Diagram

## Block-by-Block Breakdown


## Error Handling

## Callback Logic

## Data Dips

## Conclusion

## High-Level Flow Diagram
```mermaid
flowchart TD

    %% ===== BEGIN =====
    A0([Begin]) --> A1[SetVariable: contactId = {ContactId}]
    A1 --> A2[SetVariable: ani = {ANI}]
    A2 --> A3[SetVariable: dnis = {DNIS}]
    A3 --> A4[SetVariable: attemptCount = 0]
    A4 --> A5[Log: Call started]
    A5 --> A6[PlayAudio: Welcome]
    A6 --> BH[Business Hours Check]

    %% ===== BUSINESS HOURS =====
    BH -->|Open| CL[Customer Lookup]
    BH -->|Closed| BH1[PlayAudio: Closed Message]
    BH1 --> BH2[Skill: AfterHours_Skill]
    BH2 --> END1([End])

    %% ===== CUSTOMER LOOKUP =====
    CL -->|Success| CL1[SetVariable: customerTier]
    CL1 --> CL2[Log: Lookup Success]
    CL2 --> MM[Main Menu]

    CL -->|Error| CL3[Log: Lookup Failed]
    CL3 --> CL4[SetVariable: customerTier = Unknown]
    CL4 --> MM

    %% ===== MAIN MENU =====
    MM -->|1: Sales| S1[PlayAudio: Connecting to Sales]
    S1 --> S2[Skill: Sales_Skill]
    S2 --> END2([End])

    MM -->|2: Support| STR[Support Tier Routing]

    MM -->|3: Callback| CB[Callback Flow]

    MM -->|Timeout| T1[PlayAudio: Didn't get that]
    T1 --> T2[Increment attemptCount]
    T2 -->|attemptCount < 3| MM
    T2 -->|attemptCount >= 3| MAXA[Max Attempts]

    MM -->|Error| ERR[Error Handler]

    %% ===== SUPPORT TIER ROUTING =====
    STR -->|Gold| G1[PlayAudio: Priority Support]
    G1 --> G2[Skill: GoldSupport_Skill]
    G2 --> END3([End])

    STR -->|Silver| SL1[PlayAudio: Routing to Support]
    SL1 --> SL2[Skill: SilverSupport_Skill]
    SL2 --> END4([End])

    STR -->|Default| D1[PlayAudio: General Support]
    D1 --> D2[Skill: GeneralSupport_Skill]
    D2 --> END5([End])

    %% ===== CALLBACK FLOW =====
    CB --> CB1[PlayAudio: Enter callback number]
    CB1 --> CB2[CollectInput: callbackNumber]
    CB2 -->|Valid| CB3[CreateCallback]
    CB3 --> CB4[PlayAudio: Callback Scheduled]
    CB4 --> END6([End])

    CB2 -->|Invalid| CB5[PlayAudio: Invalid Number]
    CB5 --> CB

    %% ===== MAX ATTEMPTS =====
    MAXA --> MA1[PlayAudio: Trouble Understanding]
    MA1 --> MA2[Skill: GeneralSupport_Skill]
    MA2 --> END7([End])

    %% ===== ERROR HANDLER =====
    ERR --> E1[PlayAudio: Error Message]
    E1 --> E2[Log: Error Handler Triggered]
    E2 --> E3[Skill: GeneralSupport_Skill]
    E3 --> END8([End])
```

`
