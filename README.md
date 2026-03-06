````markdown
```mermaid
graph LR
    %% 顶刊视觉配色方案定义 (Nature / IEEE 风格)
    classDef logic fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#1b5e20,font-weight:bold;
    classDef reg fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100,font-weight:bold;
    classDef input fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#0d47a1,font-weight:bold;
    classDef output fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#4a148c,font-weight:bold;
    classDef clk fill:#ffebee,stroke:#c62828,stroke-width:2px,color:#c62828,font-weight:bold;
    classDef boundary fill:none,stroke:#78909c,stroke-width:2px,stroke-dasharray: 5 5;
    
    %% ================= 输入区 =================
    subgraph CDC ["CDC 跨域隔离舱"]
        direction TB
        i_in("i_latch<br>(32-bit)"):::input
        tau_in("tau_latch<br>(48-bit)"):::input
        req("req_rising<br>(控制信号)"):::input
    end
    
    %% ================= 第一级乘法 =================
    subgraph Stage1 ["第一级乘法模块"]
        direction TB
        mult_ii["i² 乘法器<br>(i × i)"]:::logic
        mult_itau["i×τ 乘法器<br>(i × τ)"]:::logic
    end
    
    i_in ==> mult_ii
    i_in ==> mult_itau
    tau_in ==> mult_itau
    
    %% ================= 流水线墙 =================
    subgraph Pipeline ["流水线寄存器切割 (Pipeline Cut)"]
        direction TB
        dff1["DFF<br>(mult_ii_pipe)"]:::reg
        dff2["DFF<br>(mult_itau_pipe)"]:::reg
        dff3["DFF<br>(i_pipe)"]:::reg
        dff4["DFF<br>(τ_pipe)"]:::reg
        dff5["DFF<br>(常数1)"]:::reg
    end
    
    mult_ii ==> dff1
    mult_itau ==> dff2
    i_in --> dff3
    tau_in --> dff4
    req --> dff5
    
    %% ================= 累加树 =================
    subgraph Stage2 ["第二级并行的宽位深加法树与反馈寄存器"]
        direction TB
        acc_ii["Σi² 累加器<br>(加法器+Reg)"]:::logic
        acc_itau["Σiτ 累加器<br>(加法器+Reg)"]:::logic
        acc_i["Σi 累加器<br>(加法器+Reg)"]:::logic
        acc_tau["Στ 累加器<br>(加法器+Reg)"]:::logic
        acc_n["N 累加器<br>(加法器+Reg)"]:::logic
    end
    
    dff1 ==> acc_ii
    dff2 ==> acc_itau
    dff3 ==> acc_i
    dff4 ==> acc_tau
    dff5 ==> acc_n
    
    %% 内部反馈环路表示
    acc_ii -.->|"时钟触发累加"| acc_ii
    acc_itau -.->|"时钟触发累加"| acc_itau
    acc_i -.->|"时钟触发累加"| acc_i
    acc_tau -.->|"时钟触发累加"| acc_tau
    acc_n -.->|"时钟触发累加"| acc_n
    
    %% ================= 输出区 =================
    FSM("至 FSM 串行求解阵列"):::output
    
    acc_ii == "Σi²" ==> FSM
    acc_itau == "Σiτ" ==> FSM
    acc_i == "Σi" ==> FSM
    acc_tau == "Στ" ==> FSM
    acc_n == "N" ==> FSM
    
    %% ================= 全局时钟 =================
    CLK(("100MHz 时钟")):::clk
    CLK -.-> Pipeline
    CLK -.-> Stage2

    class CDC,Stage1,Pipeline,Stage2 boundary;
