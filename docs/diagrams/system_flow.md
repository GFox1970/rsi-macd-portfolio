%%{init: {'theme':'base', 'themeVariables': { 'fontSize':'14px'}}}%%
graph TB
    subgraph "🔄 GitHub Actions"
        A[Schedule-orchestrator.yml<br/>Cron Trigger]
    end
    
    subgraph "🐳 Docker Containers"
        B[Daily Orchestrator<br/>Container]
        C[Intraday Orchestrator<br/>Container]
        D[Trading Bot<br/>Container]
        G[Webserver Container<br/>Monitoring]
    end
    
    subgraph "📊 Data & ML Pipeline"
        E[Data Fetching &<br/>Processing]
        F[ML Integration &<br/>Pipeline]
    end
    
    subgraph "💰 Capital & Risk Management"
        H[Capital Allocation<br/>Manager]
        I[Capital Risk<br/>Manager]
        J[Day Trading<br/>Manager]
    end
    
    subgraph "📝 Logging & Monitoring"
        K[Enhanced Decision<br/>Logger]
    end
    
    A -->|"1. Triggers daily_orchestrator.py"| B
    B -->|"2. Prepares data, trains models,<br/>generates candidates"| E
    B -->|"3. Runs ML training pipeline"| F
    B -->|"4. Triggers intraday<br/>orchestrator start"| C
    C -->|"5. Manages trading bot<br/>lifecycle"| D
    E -->|"6. Provides data to"| D
    F -->|"7. Provides ML scoring<br/>and feedback"| D
    D -->|"8. Executes trades,<br/>manages positions"| H
    D -->|"9. Executes trades,<br/>manages risk"| I
    D -->|"10. Executes trades,<br/>manages day trading"| J
    D -->|"11. Logs decisions<br/>and trades"| K
    K -->|"12. Logs accessible by"| G
    
    classDef triggerStyle fill:#e1f5ff,stroke:#0288d1,stroke-width:2px
    classDef containerStyle fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef dataStyle fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef capitalStyle fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    classDef logStyle fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    
    class A triggerStyle
    class B,C,D,G containerStyle
    class E,F dataStyle
    class H,I,J capitalStyle
    class K logStyle