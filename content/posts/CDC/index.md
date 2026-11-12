---
title: "CDC跨时钟域学习"
date: 2026-11-12
draft: false
tags: ["CDC", "NPU", "数字IC"]
categories: ["日志"]
summary: "CDC的几种方法"
---

## 为什么需要CDC
CDC顾名思义为跨时钟域的意思：
一个时钟域产生的信号，被另一个时钟域采样时，无法天然保证采样安全和数据正确。

主要有三类问题：

|问题|	为什么发生|	可能的后果|
|---|---|---|
|亚稳态|信号恰好在接收端时钟边沿附近变化，违反触发器的建立或保持时间	| 触发器输出不能及时稳定，影响后续逻辑|
|事件丢失|	快时钟域产生的脉冲太短，慢时钟域还没采样，脉冲就结束了|	“开始”“完成”等事件没有被接收到|
|多位数据不一致|总线各位的传播时间不同，接收端可能混合采到新旧数据	| 数据错误 |

> **建立时间: 时钟信号来临时，输入信号需要保持稳定的时间。**

> **保持时间：时钟信号来临后，输入信号需要保持稳定的时间。**

## 单比特CDC
对于单 bit 慢变电平信号（目标时钟域采样时，源信号必须至少维持 2~3 个目标周期）场景。

常用方法是2DFF，即两个触发器。

试想直接用一个触发器采样会发生什么，假设采样时，输入数据在接收端时钟边沿附近变化，这样采样出来的数据就是亚稳态（输出电压可能暂时停留在高低电平之间），无法判断是0或者是1。

如果采用两个触发器，哪怕第一级可能进入亚稳态。第二级不会立刻采第一级的输出，而是通常要等到下一个时钟边沿才采样。这样，第一级就有接近一个时钟周期的时间稳定下来，可以极大缓解亚稳态的现象。

参考代码如下：
{{< collapse summary="展开参考代码" >}}
```verilog
module sync_2ff #(
    parameter bit RESET_VAL = 1'b0  // 复位后的默认初始值
) (
    input  logic clk,       // 目标时钟域 (Destination Clock)
    input  logic rst_n,     // 目标时钟域异步复位 (Active-low)
    input  logic din,       // 源时钟域异步输入 (Async Input)
    output logic dout       // 目标时钟域同步后输出 (Synced Output)
);

    // 工业级后端布局约束：强制放置在相邻位置
    (* ASYNC_REG = "TRUE" *)
    logic q1, q2;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            q1 <= RESET_VAL;
            q2 <= RESET_VAL;
        end else begin
            q1 <= din;
            q2 <= q1;
        end
    end

    assign dout = q2;

endmodule
```
{{< /collapse >}}

那如果是个短脉冲信号呢？这样可能会出现慢时钟域还没采样，脉冲就结束了的现象。

**在“脉冲事件足够稀疏，源脉冲之间必须留出足够间隔，让目标域至少采到每次翻转”的场景下**，我们可以采取如下的方案：采用一个翻转器持久记录是否有脉冲。
参考代码如下：
{{< collapse summary="展开参考代码" >}}
```verilog
module pulse_sync (
    // 源时钟域 (Source Clock Domain)
    input  logic clk_src,
    input  logic rst_n_src,
    input  logic pulse_src,     // 源时钟域单周期输入脉冲

    // 目标时钟域 (Destination Clock Domain)
    input  logic clk_dst,
    input  logic rst_n_dst,
    output logic pulse_dst      // 目标时钟域还原出的单周期脉冲
);

    //-------------------------------------------------------------------------
    // 1. 源时钟域：脉冲转电平翻转 (Pulse-to-Toggle)
    //-------------------------------------------------------------------------
    logic src_toggle_reg;

    always_ff @(posedge clk_src or negedge rst_n_src) begin
        if (!rst_n_src) begin
            src_toggle_reg <= 1'b0;
        end else if (pulse_src) begin
            src_toggle_reg <= ~src_toggle_reg; // 每来一个脉冲，电平翻转一次
        end
    end

    //-------------------------------------------------------------------------
    // 2. 跨时钟域：双触发器同步 (2-Flop Synchronizer)
    //-------------------------------------------------------------------------
    (* ASYNC_REG = "TRUE")
    logic dst_sync_q1;

    (* ASYNC_REG = "TRUE")
    logic dst_sync_q2;

    always_ff @(posedge clk_dst or negedge rst_n_dst) begin
        if (!rst_n_dst) begin
            dst_sync_q1 <= 1'b0;
            dst_sync_q2 <= 1'b0;
        end else begin
            dst_sync_q1 <= src_toggle_reg;
            dst_sync_q2 <= dst_sync_q1;
        end
    end

    //-------------------------------------------------------------------------
    // 3. 目标时钟域：双边沿检测还原脉冲 (Toggle-to-Pulse via XOR)
    //-------------------------------------------------------------------------
    logic dst_sync_q3;

    always_ff @(posedge clk_dst or negedge rst_n_dst) begin
        if (!rst_n_dst) begin
            dst_sync_q3 <= 1'b0;
        end else begin
            dst_sync_q3 <= dst_sync_q2; // 再延时一拍，用于边沿比对
        end
    end

    // 异或门：只要最新拍(q2)与前一拍(q3)不相等，说明电平发生了翻转，还原出脉冲！
    assign pulse_dst = dst_sync_q2 ^ dst_sync_q3;

endmodule
```
{{< /collapse >}}

上述只是介绍了两个场景的CDC。

补充：
> 事件稀疏，允许有最小间隔：toggle synchronizer 这类 pulse CDC 十分常见，硬件简单、延迟也有限。

> 源端能等目标端确认，每次只发一个事件-> 握手

> 源端短时间内会连续产生事件，目标端之后再逐个处理-> FIFO 缓冲

> 源端持续产生速度高于目标端处理速度-> 任何有限 FIFO 最终都会满；必须反压、丢弃，或提高目标端吞吐量

## 多bitCDC
在异步FIFO场景中，存在读写指针存在多bit跨时钟域：一个时钟域的读，另一个时钟域的写，一个简单思路是能不能多个2DFF？

答案是不能，在芯片的实际物理版图上，每个 bit 经过的金属走线长度不同、存在由于延时不同导致的bit翻转时间不一致，即数据倾斜问题，产生幽灵信号。

那怎么解决这个问题呢？我保证一次无法多个bit翻转不就行了吗！自然而然想到格雷码，格雷码能保证每次只变化一个bit位。

目标时钟去采样时，这个跳变的 bit 就算发生了亚稳态决断：
- 决断为新值：目标端看到了最新指针，正确！
- 决断为旧值：目标端看到了上一拍的旧指针，即维持原样，不会产生错误！写端看到的是旧的读指针。实际上读端可能在这 2 拍里又读走了几个数据，FIFO 实际已经没那么满了。提前挂起停止写入。只会损失极微小的性能，但绝对不会发生“写溢出（Overflow）”！读端看到的是旧的写指针。读端误以为空了，多等了一两拍才开读。→绝对不会发生“读下溢（Underflow）”！

{{< collapse summary="展开参考代码" >}}
```verilog
module async_fifo #(
    parameter int DATA_WIDTH = 32,
    parameter int ADDR_WIDTH = 4   // Depth = 2^ADDR_WIDTH = 16
) (
    // 写时钟域 (Write Domain)
    input  logic                  wclk,
    input  logic                  wrst_n,
    input  logic                  wr_en,
    input  logic [DATA_WIDTH-1:0] wdata,
    output logic                  wfull,

    // 读时钟域 (Read Domain)
    input  logic                  rclk,
    input  logic                  rrst_n,
    input  logic                  rd_en,
    output logic [DATA_WIDTH-1:0] rdata,
    output logic                  rempty
);

    localparam int DEPTH = 1 << ADDR_WIDTH;

    //-------------------------------------------------------------------------
    // 1. 双端口存储阵列 (Dual-Port Memory)
    //-------------------------------------------------------------------------
    logic [DATA_WIDTH-1:0] mem [0:DEPTH-1];

    logic [ADDR_WIDTH:0] wbin, rbin;
    logic [ADDR_WIDTH:0] wgray, rgray;
    logic [ADDR_WIDTH:0] wgray_sync, rgray_sync;

    // 写存储
    always_ff @(posedge wclk) begin
        if (wr_en && !wfull) begin
            mem[wbin[ADDR_WIDTH-1:0]] <= wdata;
        end
    end

    // 读存储
    assign rdata = mem[rbin[ADDR_WIDTH-1:0]];

    //-------------------------------------------------------------------------
    // 2. 写时钟域：二进制与格雷码写指针推进
    //-------------------------------------------------------------------------
    logic [ADDR_WIDTH:0] wbin_next, wgray_next;

    assign wbin_next  = wbin + (wr_en & ~wfull);
    assign wgray_next = wbin_next ^ (wbin_next >> 1); // Bin to Gray

    always_ff @(posedge wclk or negedge wrst_n) begin
        if (!wrst_n) begin
            wbin  <= '0;
            wgray <= '0;
        end else begin
            wbin  <= wbin_next;
            wgray <= wgray_next;
        end
    end

    //-------------------------------------------------------------------------
    // 3. 读时钟域：二进制与格雷码读指针推进
    //-------------------------------------------------------------------------
    logic [ADDR_WIDTH:0] rbin_next, rgray_next;

    assign rbin_next  = rbin + (rd_en & ~rempty);
    assign rgray_next = rbin_next ^ (rbin_next >> 1); // Bin to Gray

    always_ff @(posedge rclk or negedge rrst_n) begin
        if (!rrst_n) begin
            rbin  <= '0;
            rgray <= '0;
        end else begin
            rbin  <= rbin_next;
            rgray <= rgray_next;
        end
    end

    //-------------------------------------------------------------------------
    // 4. 格雷码指针跨时钟域同步 (双触发器 2DFF)
    //-------------------------------------------------------------------------
    // 将写格雷码指针 wgray 同步到读时钟域 (rclk)
    (* ASYNC_REG = "TRUE" *) logic [ADDR_WIDTH:0] wgray_q1, wgray_q2;
    always_ff @(posedge rclk or negedge rrst_n) begin
        if (!rrst_n) begin
            wgray_q1 <= '0;
            wgray_q2 <= '0;
        end else begin
            wgray_q1 <= wgray;
            wgray_q2 <= wgray_q1;
        end
    end
    assign wgray_sync = wgray_q2;

    // 将读格雷码指针 rgray 同步到写时钟域 (wclk)
    (* ASYNC_REG = "TRUE" *) logic [ADDR_WIDTH:0] rgray_q1, rgray_q2;
    always_ff @(posedge wclk or negedge wrst_n) begin
        if (!wrst_n) begin
            rgray_q1 <= '0;
            rgray_q2 <= '0;
        end else begin
            rgray_q1 <= rgray;
            rgray_q2 <= rgray_q1;
        end
    end
    assign rgray_sync = rgray_q2;

    //-------------------------------------------------------------------------
    // 5. 空 / 满标志判定
    //-------------------------------------------------------------------------
    // 读空判定：读格雷码指针与同步过来的写格雷码指针全等
    assign rempty = (rgray == wgray_sync);

    // 写满判定：最高两位相反，低位全等
    assign wfull  = (wgray == {~rgray_sync[ADDR_WIDTH:ADDR_WIDTH-1], 
                                rgray_sync[ADDR_WIDTH-2:0]});

endmodule
```
{{< /collapse >}}