---
title: "AXI总线协议学习"
date: 2026-11-13
draft: false
tags: ["AXI", "NPU", "数字IC"]
categories: ["日志"]
summary: "AXI的几种方法"
---

## 基本概念
AXI 是 ARM AMBA 总线协议中的一种接口协议，核心机制是：`VALID && READY`，只有在同一个时钟周期中均为1才算一次数据传输。

AXI 的第四代协议版本，即 AXI4。

AXI4-Lite表示简化版，主要特点是：不支持`burst`（突发传输）；接口比完整 AXI4 简单；常用于**CPU 配置寄存器、控制状态寄存器、外设寄存器访问**。

AXI4-burst表示可以突发传输。
普通单次传输每次只传一个 beat：地址 0x1000 -> 数据 A
地址 0x1004 -> 数据 B，如果每次都重新发送完整的地址控制信息，效率较低。

Burst 允许先发送一次起始地址和传输配置，然后连续传输多个beat，例如：

```
一次 burst：
起始地址：0x1000
beat 数量：4
每 beat：4 bytes

beat0: 0x1000 -> A
beat1: 0x1004 -> B
beat2: 0x1008 -> C
beat3: 0x100C -> D
```

适合**内存、DMA（Direct Memory Access 的缩写，中文叫直接内存访问，它是一种让硬件模块在不需要 CPU 逐个搬运数据的情况下，直接在内存或外设之间传输数据的机制）、高吞吐数据**场景。

## AXI4-Lite

包括五个通道：
| 通道 | 方向 | 作用 |
|---|---|---|
| AW | Master → Slave | 写地址 |
| W | Master → Slave | 写数据和 WSTRB |
| B | Slave → Master | 写响应 |
| AR | Master → Slave | 读地址 |
| R | Slave → Master | 读数据和读响应 |

> 由于读数据和读响应都是从（Slave）到主（Master），故其实可以放到一起。

> `WSTRB` 是 AXI 写数据通道中的 `Write Strobe`，中文可以叫写字节使能。它用来表示 `WDATA` 中哪些字节有效，哪些字节应该保持目标寄存器或内存中的原值

{{< collapse summary="展开参考代码" >}}
```verilog
//                 传输必须对齐到数据总线宽度(本例 4B 对齐)
// ============================================================================
module axi4_lite_slave #(
    parameter ADDR_WIDTH = 8,      // 字节地址宽度 -> 2^8 = 256B
    parameter DATA_WIDTH = 32
)(
    input  wire                    ACLK,
    input  wire                    ARESETn,

    // ---- Write Address Channel ----
    input  wire [ADDR_WIDTH-1:0]   AWADDR,
    input  wire                    AWVALID,
    output wire                    AWREADY,

    // ---- Write Data Channel ----
    input  wire [DATA_WIDTH-1:0]   WDATA,
    input  wire [DATA_WIDTH/8-1:0] WSTRB,
    input  wire                    WVALID,
    output wire                    WREADY,

    // ---- Write Response Channel ----
    output reg  [1:0]              BRESP,
    output reg                     BVALID,
    input  wire                    BREADY,

    // ---- Read Address Channel ----
    input  wire [ADDR_WIDTH-1:0]   ARADDR,
    input  wire                    ARVALID,
    output wire                    ARREADY,

    // ---- Read Data Channel ----
    output wire [DATA_WIDTH-1:0]   RDATA,
    output reg  [1:0]              RRESP,
    output reg                     RVALID,
    input  wire                    RREADY
);

    localparam NBYTE = DATA_WIDTH/8;
    localparam DEPTH = 1<<ADDR_WIDTH;
    localparam OKAY  = 2'b00;
    localparam SLVERR= 2'b10;

    // 以字节为单位建模, 才能干净地支持 WSTRB
    reg [7:0] mem [0:DEPTH-1];
    integer i;
    initial begin                      // FPGA 可综合; ASIC 下请删除, 改由复位/扫描链处理
        for (i=0; i<DEPTH; i=i+1) mem[i] = 8'h00;
    end

    reg [ADDR_WIDTH-1:0] awaddr_r;
    reg [DATA_WIDTH-1:0] wdata_r;
    reg [NBYTE-1:0]      wstrb_r;
    reg                  aw_got;       // 地址已到、数据未到
    reg                  w_got;        // 数据已到、地址未到

    reg [ADDR_WIDTH-1:0] araddr_r;
    reg                  ar_ok_r;
    reg                  r_active;     // 有一笔读事务在处理中

    // ------------------------------------------------------------------
    // Backpressure: READY = f(内部状态), 与 VALID 无关
    //   AXI4-Lite 常规做法是"一次只受理一笔事务", 因此上一笔没吐完响应
    //   就把对应通道 READY 拉低 —— 这正是 slave 侧背压的来源。
    //   (若想支持多笔 outstanding, 把下面的标志换成 FIFO 的 !full 即可)
    // ------------------------------------------------------------------
    assign AWREADY = !aw_got && !BVALID;
    assign WREADY  = !w_got  && !BVALID;
    assign ARREADY = !r_active;

    wire aw_hs = AWVALID & AWREADY;
    wire w_hs  = WVALID  & WREADY;
    wire b_hs  = BVALID  & BREADY;
    wire ar_hs = ARVALID & ARREADY;
    wire r_hs  = RVALID  & RREADY;

    // 地址合法性: 对齐 + 不越界
    wire aw_ok = (AWADDR[1:0] == 2'b00) && (AWADDR <= DEPTH-NBYTE);
    wire ar_ok = (ARADDR[1:0] == 2'b00) && (ARADDR <= DEPTH-NBYTE);

    // ------------------------------------------------------------------
    // 1) AW / W 独立到达: 两条通道各自握手、各自缓存
    // ------------------------------------------------------------------
    always @(posedge ACLK or negedge ARESETn) begin
        if (!ARESETn) begin
            aw_got <= 1'b0; w_got <= 1'b0;
            awaddr_r <= {ADDR_WIDTH{1'b0}};
            wdata_r  <= {DATA_WIDTH{1'b0}};
            wstrb_r  <= {NBYTE{1'b0}};
        end else begin
            if (aw_hs) begin awaddr_r <= AWADDR; aw_got <= 1'b1; end
            if (w_hs)  begin wdata_r  <= WDATA;  wstrb_r <= WSTRB; w_got <= 1'b1; end
            if (aw_got && w_got) begin          // 两路凑齐 -> 消费掉
                aw_got <= 1'b0;
                w_got  <= 1'b0;
            end
        end
    end

    // ------------------------------------------------------------------
    // 2) WSTRB: WSTRB[n] 对应 WDATA[8n+7 : 8n], 为 0 的字节保持原值
    // ------------------------------------------------------------------
    always @(posedge ACLK or negedge ARESETn) begin
        if (!ARESETn) begin
            BVALID <= 1'b0;
            BRESP  <= OKAY;
        end else if (aw_got && w_got) begin          // 地址+数据都已就位
            for (i=0; i<NBYTE; i=i+1)
                if (wstrb_r[i]) mem[awaddr_r + i] <= wdata_r[8*i +: 8];
            BVALID <= 1'b1;
            BRESP  <= (awaddr_r[1:0]==2'b00 && awaddr_r<=DEPTH-NBYTE) ? OKAY : SLVERR;
        end else if (b_hs) begin
            BVALID <= 1'b0;
        end
    end

    // ------------------------------------------------------------------
    // 读路径: AR 锁存 -> 下一拍给出 RDATA/RVALID -> 等 RREADY
    // ------------------------------------------------------------------
    always @(posedge ACLK or negedge ARESETn) begin
        if (!ARESETn) begin
            r_active <= 1'b0; RVALID <= 1'b0; RRESP <= OKAY;
            araddr_r <= {ADDR_WIDTH{1'b0}}; ar_ok_r <= 1'b0;
        end else if (ar_hs) begin
            araddr_r <= ARADDR;
            ar_ok_r  <= ar_ok;
            RVALID   <= 1'b1;
            RRESP    <= ar_ok ? OKAY : SLVERR;
            r_active <= 1'b1;
        end else if (r_hs) begin
            RVALID   <= 1'b0;
            r_active <= 1'b0;
        end
    end

    wire [ADDR_WIDTH-1:0] ar_base = ar_ok_r ? araddr_r : {ADDR_WIDTH{1'b0}};

    genvar g;
    generate
        for (g=0; g<NBYTE; g=g+1) begin : gen_rdata
            assign RDATA[8*g +: 8] = mem[ar_base + g];
        end
    endgenerate

endmodule
```
{{< /collapse >}}

## AXI4-burst
- **Beat**：数据通道一次 `VALID && READY` 握手所传送的数据。背压时一个 beat 可等待多个时钟周期。
- **Burst**：一次 AW 或 AR 地址握手所描述的一串 beat；`LEN` 表示 beat 数减一。例如 `LEN=2` 表示 3 个 beat。
- **Outstanding**：请求地址已握手、对应事务尚未完成。例如读 AR 已握手但尚未接受最后一拍 R；写 AW 已握手但尚未接受 B 响应。
- **ID**：请求的 `AWID`/`ARID` 会在对应响应的 `BID`/`RID` 中返回，用于区分并发事务。相同 ID 的响应保持相应顺序；不同 ID 的响应可以乱序。单笔事务也可以携带 ID。
- 

## DMA
一个内存到内存 DMA 可以抽象为：

```text
源地址 SRC
    │ AXI Read Address
    ▼
读数据 RDATA/RVALID/RLAST
    │
    ▼
内部 FIFO 或 skid buffer
    │
    ▼
写数据 WDATA/WVALID/WLAST
    │
    ▼
目标地址 DST
```

更高吞吐量的实现可以让读和写并行，并用异步 FIFO 或同步 FIFO 解耦两端。