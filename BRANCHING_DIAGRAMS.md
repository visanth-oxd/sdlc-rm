# Branching Strategy Visual Diagrams

## Complete Branch-to-Environment Flow

```mermaid
graph TB
    subgraph "Production Flow (n)"
        M[main branch]
        M -->|Deploy| BLD1[BLD-stable-01]
        BLD1 -->|Promote| INT1[INT-stable-01]
        INT1 -->|Promote| PRE1[PRE-stable-01]
        PRE1 -->|Promote| PRD1[PRD-stable-01]
    end
    
    subgraph "Beta Flow (n+1)"
        R1[release/n+1 branch]
        R1 -->|Deploy| BLD2[BLD-beta-01]
        BLD2 -->|Promote| INT2[INT-beta-01]
        INT2 -->|Promote| PRE2[PRE-beta-01]
    end
    
    subgraph "Alpha Flow (n+2)"
        R2[release/n+2 branch]
        R2 -->|Deploy| BLD3[BLD-alpha-01]
        BLD3 -->|Promote| INT3[INT-alpha-01]
    end
    
    R1 -.->|Promote to Production| M
    R2 -.->|Promote to Beta| R1
```

## Hotfix Workflow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Main as main branch
    participant Hotfix as hotfix/bug-fix
    participant BLD as BLD-stable-01
    participant INT as INT-stable-01
    participant PRE as PRE-stable-01
    participant PRD as PRD-stable-01
    participant R1 as release/n+1
    participant R2 as release/n+2
    
    Dev->>Main: Checkout main
    Dev->>Hotfix: Create hotfix branch
    Dev->>Hotfix: Fix bug & commit
    Hotfix->>BLD: Deploy & test
    BLD->>INT: Promote & test
    INT->>PRE: Promote & test
    PRE->>PRD: Promote to production
    Dev->>Main: Create PR: hotfix → main
    Main->>Main: Merge hotfix
    Main->>R1: Cherry-pick fix
    Main->>R2: Cherry-pick fix
```

## Feature Development Workflow

```mermaid
graph LR
    subgraph "Beta Feature (n+1)"
        R1[release/n+1] -->|Create| F1[feature/new-feature]
        F1 -->|Develop| F1
        F1 -->|Deploy| BLD2[BLD-beta-01]
        BLD2 -->|Test| INT2[INT-beta-01]
        F1 -->|PR| R1
        R1 -->|Merge| R1
    end
    
    subgraph "Alpha Feature (n+2)"
        R2[release/n+2] -->|Create| F2[feature/future-feature]
        F2 -->|Develop| F2
        F2 -->|Deploy| BLD3[BLD-alpha-01]
        BLD3 -->|Test| INT3[INT-alpha-01]
        F2 -->|PR| R2
        R2 -->|Merge| R2
    end
```

## Release Promotion Flow

```mermaid
graph TD
    R2[release/n+2<br/>Alpha n+2] -->|When ready| R1[release/n+1<br/>Beta n+1]
    R1 -->|When ready| M[main<br/>Production n]
    
    M -->|Tag| V1[v1.2.0]
    R1 -->|Tag| V2[v1.3.0-beta.1]
    R2 -->|Tag| V3[v1.4.0-alpha.1]
    
    M -->|Deploy| PROD[PRD-stable-01]
    R1 -->|Deploy| BETA[PRE-beta-01]
    R2 -->|Deploy| ALPHA[INT-alpha-01]
```

## Multi-Instance Environment Flow

```mermaid
graph TB
    subgraph "Stable Environments"
        M[main] --> BLD1[BLD-stable-01]
        M --> BLD2[BLD-stable-02]
        BLD1 --> INT1[INT-stable-01]
        BLD2 --> INT2[INT-stable-02]
        INT1 --> PRE1[PRE-stable-01]
        INT2 --> PRE2[PRE-stable-02]
        PRE1 --> PRD1[PRD-stable-01]
        PRE2 --> PRD2[PRD-stable-02]
    end
    
    subgraph "Beta Environments"
        R1[release/n+1] --> BLD3[BLD-beta-01]
        R1 --> BLD4[BLD-beta-02]
        BLD3 --> INT3[INT-beta-01]
        BLD4 --> INT4[INT-beta-02]
        INT3 --> PRE3[PRE-beta-01]
        INT4 --> PRE4[PRE-beta-02]
    end
```

## Branch Protection and Merge Flow

```mermaid
graph TB
    subgraph "Branch Types"
        M[main<br/>🔒 Protected<br/>2 Reviews Required]
        R1[release/n+1<br/>🔒 Protected<br/>1 Review Required]
        R2[release/n+2<br/>🔒 Protected<br/>1 Review Required]
        H[hotfix/*<br/>🔒 Protected<br/>2 Reviews Required]
        F[feature/*<br/>✅ Open<br/>1 Review Required]
    end
    
    F -->|PR| R1
    F -->|PR| R2
    H -->|PR| M
    R1 -->|PR| M
    R2 -->|PR| R1
    
    style M fill:#ff6b6b
    style R1 fill:#4ecdc4
    style R2 fill:#95e1d3
    style H fill:#ffe66d
    style F fill:#a8e6cf
```

## CI/CD Pipeline Flow

```mermaid
graph LR
    subgraph "On Push to Branch"
        P[Push Event] --> D[Detect Branch Type]
        D -->|main| S1[Deploy to BLD-stable-01]
        D -->|release/n+1| S2[Deploy to BLD-beta-*]
        D -->|release/n+2| S3[Deploy to BLD-alpha-*]
        D -->|hotfix/*| S1
        D -->|feature/*| T[Run Tests Only]
    end
    
    subgraph "Environment Promotion"
        S1 -->|Tests Pass| I1[INT-stable-01]
        S2 -->|Tests Pass| I2[INT-beta-*]
        S3 -->|Tests Pass| I3[INT-alpha-*]
        
        I1 -->|Tests Pass| PRE1[PRE-stable-01]
        I2 -->|Tests Pass| PRE2[PRE-beta-*]
        
        PRE1 -->|Approval| PRD1[PRD-stable-01]
    end
```

## Service Deployment Matrix

```mermaid
graph TB
    subgraph "Service A"
        SA_M[main] --> SA_BLD[BLD-stable-01]
        SA_R1[release/n+1] --> SA_BLD_B[BLD-beta-01]
    end
    
    subgraph "Service B"
        SB_M[main] --> SB_BLD[BLD-stable-01]
        SB_R1[release/n+1] --> SB_BLD_B[BLD-beta-01]
    end
    
    subgraph "Service C"
        SC_M[main] --> SC_BLD[BLD-stable-01]
        SC_R1[release/n+1] --> SC_BLD_B[BLD-beta-01]
    end
    
    SA_M -.->|Coordinated Release| SB_M
    SB_M -.->|Coordinated Release| SC_M
```

## Version Timeline

```mermaid
timeline
    title Release Timeline
    
    section Current (n)
        Production : main : PRD-stable-01
        Hotfixes   : hotfix/* → main
    
    section Next Release (n+1)
        Beta       : release/n+1
        Features   : feature/* → release/n+1
        Testing    : BLD-beta-* → INT-beta-* → PRE-beta-*
    
    section Future (n+2)
        Alpha      : release/n+2
        Features   : feature/* → release/n+2
        Testing    : BLD-alpha-* → INT-alpha-*
```

## Decision Tree: Which Branch to Use?

```mermaid
graph TD
    START[Need to make changes?] --> Q1{Is it a<br/>critical production bug?}
    
    Q1 -->|Yes| HOTFIX[Create hotfix/*<br/>from main]
    Q1 -->|No| Q2{Is it a new feature<br/>for next release?}
    
    Q2 -->|Yes| Q3{Is it experimental<br/>or future feature?}
    Q2 -->|No| Q4{Is it documentation<br/>or maintenance?}
    
    Q3 -->|Yes| ALPHA[Create feature/*<br/>from release/n+2]
    Q3 -->|No| BETA[Create feature/*<br/>from release/n+1]
    
    Q4 -->|Yes| MAINT[Create feature/*<br/>from appropriate branch]
    
    HOTFIX --> DEPLOY1[Deploy to stable envs]
    BETA --> DEPLOY2[Deploy to beta envs]
    ALPHA --> DEPLOY3[Deploy to alpha envs]
    MAINT --> DEPLOY4[Deploy based on target]
```

## Branch Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created: Create branch
    Created --> Development: Start development
    Development --> Testing: Deploy to test env
    Testing --> Review: Create PR
    Review --> Merged: Approved & merged
    Review --> Development: Changes requested
    Merged --> [*]: Delete branch
    
    note right of Testing
        BLD → INT → PRE → PRD
        (based on branch type)
    end note
```

## Environment Promotion Gates

```mermaid
graph LR
    BLD[BLD Environment] -->|✅ Automated Tests Pass<br/>✅ Code Review Approved| INT[INT Environment]
    INT -->|✅ Integration Tests Pass<br/>✅ QA Sign-off| PRE[PRE Environment]
    PRE -->|✅ Pre-prod Validation Pass<br/>✅ Change Approval Board| PRD[PRD Environment]
    
    style BLD fill:#e3f2fd
    style INT fill:#bbdefb
    style PRE fill:#90caf9
    style PRD fill:#64b5f6
```

