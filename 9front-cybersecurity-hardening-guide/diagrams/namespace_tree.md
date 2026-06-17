# Plan 9 Namespace Hierarchy

```mermaid
graph TD
    ROOT[/ - root device #/] --> BIN[/bin - executables]
    ROOT --> LIB[/lib - libraries]
    ROOT --> USR[/usr - user homes]
    ROOT --> NET[/net - network #I]
    ROOT --> PROC[/proc - processes #p]
    ROOT --> DEV[/dev - devices #c #d]
    ROOT --> SRV[/srv - named servers]
    ROOT --> MNT[/mnt - mount points]
    MNT --> FACT[/mnt/factotum - auth agent]

    style ROOT fill:#1A1A2E,color:#C9A84C
    style FACT fill:#CC3333,color:#fff
    style PROC fill:#2D2D44,color:#E8E8F0
    style SRV fill:#2D2D44,color:#E8E8F0
```

**Security note:** A process with rfork(RFCNAMEG) starts with an empty namespace.
Every entry above must be explicitly bound. Resources not bound do not exist.
