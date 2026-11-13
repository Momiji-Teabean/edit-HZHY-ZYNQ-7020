# Zynq-7020 + YT8521 RGMII + lwIP220 裸机千兆网修复

## 1. 适用环境

- 板卡：HZ-ZYNQ_EVM，Zynq-7020
- PHY：YT8521，RGMII，MDIO 地址实际为 7
- MAC：Zynq PS GEM0，MIO16..27
- IP：板卡 192.168.1.10/24，PC 192.168.1.89/24，网关 192.168.1.1
- TCP echo：端口 6001
- 链路：1 Gbps，直连 PC 和千兆交换机均已验证

## 2. 软件环境

- Vitis Unified IDE 2025.2
- standalone_ps7_cortexa9_0
- lwIP 2.2.0（lwip220），lwip_echo_server 模板
- BSP 使用 System Device Tree（SDT），编译参数包含 `-DSDT`
- XILTIMER interval timer 使用 TTC0
- `lwip220_pbuf_pool_size = 16384`，`lwip220_mem_size = 524288`
- UART 115200，8N1
- BSP 的 `lwip220_dhcp` 保持 `true`，用于满足 `lwip_echo_server` 模板校验；应用代码使用静态 IP，不调用 `dhcp_start()`

## 3. 结论

本次问题不是 IP 网段、DHCP、ARP 或 PHY 地址错误，而是两个问题叠加：

1. **软件问题：SDT 编译分支导致 GEM 时钟分频代码被编译掉。**
2. **硬件问题：GEM RGMII TX 的 MIO 驱动能力偏弱，1 Gbps 信号余量不足。**

最终配置：

- MIO16..27：slow slew + 12 mA
- `MIO_PIN_16 = 0x1402`
- `MIO_PIN_22 = 0x1403`
- YT8521 `A001 = 0x8160`
- YT8521 `A003 = 0x00FD`，1000M TX delay 约 1.95 ns
- 静态 IP，不启动 DHCP

实测结果：

- Ping 50 次：50/50，0% 丢包
- Ping 10 次：10/10
- TCP 6001 回显正常
- 关闭逐包诊断后，Ping 平均延迟小于 1 ms

## 4. 软件问题：SDT 宏与 GEM 时钟分频

Vitis 2025.2 Unified IDE 生成的 BSP 编译参数中包含：

```text
-DSDT
```

原始 Xilinx `xemacpsif_physpeed.c` 中的 `SetUpSLCRDivisors()` 使用了旧的 XPAR 宏，并且整个有效函数体位于：

```c
#ifndef SDT
...
#endif
```

因此在 SDT 构建中，这个函数实际上不会更新 GEM0 的 `SLCR_GEM0_CLK_CTRL`。当 PHY 链路速度变化时，MAC TX 时钟可能与 PHY 侧不匹配，表现为：

- PHY 能识别链路
- GEM 能收包、能发 ARP
- 1 Gbps 或降速后通信不稳定/无法 ping
- 串口仍显示默认 1000M 或错误的速度配置

修复方式是在 `xemacpsif_physpeed.c` 中增加 `#ifdef SDT` 分支，直接配置 Zynq-7000 的 GEM0 时钟分频：

```c
#ifdef SDT
    /* 1000M: div0=8, div1=1
     * 100M : div0=8, div1=5
     * 10M  : div0=8, div1=50
     */
    Xil_Out32(SLCR_UNLOCK_ADDR, SLCR_UNLOCK_KEY_VALUE);
    gem_clk_ctrl = Xil_In32(SLCR_GEM0_CLK_CTRL_ADDR);
    gem_clk_ctrl &= ~0x03F03F00U;
    gem_clk_ctrl |= (div1 << 20) | (div0 << 8);
    gem_clk_ctrl |= 0x00000001U;
    Xil_Out32(SLCR_GEM0_CLK_CTRL_ADDR, gem_clk_ctrl);
    Xil_Out32(SLCR_LOCK_ADDR, SLCR_LOCK_KEY_VALUE);
#endif
```

同时保留原有 `#ifndef SDT` 分支，兼容非 SDT 工程。不要简单地删除 `-DSDT` 来绕过问题，否则会改变 BSP 的设备树/驱动编译路径。

## 5. 硬件问题：MIO 驱动能力不足

原始 GEM0 MIO16..27 配置为：

```text
slow slew + 8 mA
```

板级 RGMII TX 路径包含约 22 欧姆串阻和 PCB 走线。在 125 MHz DDR 的千兆 RGMII TX 下，8 mA 驱动能力导致 YT8521 输入端信号余量不足，PHY 统计出现 TX 错误/无 SFD，表现为 1 Gbps 大量丢包，而 100 Mbps 可能正常。

将 MIO16..27 改为：

```text
slow slew + 12 mA
```

对应关键值：

```text
MIO_PIN_16 = 0x1402
MIO_PIN_22 = 0x1403
```

修改位置在 `main.c` 的 `init_platform()` 之后：

```c
Xil_Out32(0xF8000008U, 0x0000DF0DU);  /* SLCR unlock */

for (i = 16U; i <= 27U; i++) {
    u32 mio = 0xF8000700U + (4U * i);
    Xil_Out32(mio, (Xil_In32(mio) & ~0x00000700U) | 0x00000400U);
}

Xil_Out32(0xF8000004U, 0x0000767BU);  /* SLCR lock */
```

注意：本板 RGMII IO 是 **1.8 V**，不要改成 LVCMOS25/2.5 V。

## 6. 关键文件与修改

### 6.1 `files/main.c`

- 增加 `xil_io.h`、`xemacps.h`
- 在 `init_platform()` 后配置 GEM0 MIO16..27 为 slow slew + 12 mA
- 设置静态 IP：
  - `192.168.1.10`
  - `255.255.255.0`
  - `192.168.1.1`
- 不调用 `dhcp_start()`

### 6.2 `files/xemacpsif_physpeed.c`

- 识别 YT8521：ID1=`0x0000`，ID2=`0x011A`
- 支持 YT8521 扩展寄存器访问（0x1E/0x1F）
- 配置 A001/A003 RGMII 参数
- 增加 SDT 下的 GEM0 时钟分频配置
- 完成自协商并读取链路速度

### 6.3 `files/xemacpsif_dma.c`

该文件不是联网必需修改，主要用于调试：

- 打印 GEM RX/TX 状态
- 打印 ARP 信息
- 统计 YT8521 TX 错误

正常使用时保持：

```c
#define YT_PACKET_DIAG 0
```

逐包 UART 打印和 TX 完成时读取 PHY 统计寄存器会增加 Ping 延迟，关闭后延迟可降到 1 ms 以内。

## 7. 移植步骤

1. 备份当前 BSP 中的 `xemacpsif_physpeed.c`。
2. 将 `files/xemacpsif_physpeed.c` 复制到：

   ```text
   <platform>\ps7_cortexa9_0\standalone_ps7_cortexa9_0\bsp\libsrc\lwip220\src\lwip-2.2.0\contrib\ports\xilinx\netif\xemacpsif_physpeed.c
   ```

3. 将 `files/main.c` 中的 MIO 12 mA 配置和静态 IP 配置合并到应用。
4. 保持 BSP 的 `lwip220_dhcp = true`，但应用不要调用 `dhcp_start()`。
5. 重新编译 BSP 的 `liblwip220.a`。
6. 如果工程使用导出的 platform，还需要把新的 `liblwip220.a` 同步到导出 platform 的 `lib` 目录。
7. 重新编译并下载应用。

不要直接 Regenerate BSP 而不备份，否则手工修改的 YT8521 驱动可能被覆盖。

## 8. 验证

串口应看到：

```text
MIO_PIN_16=00001402 MIO_PIN_22=00001403
YT8521 A001=8160 A003=00FD
link speed for phy address 7: 1000
```

网络验证：

```powershell
arp.exe -d 192.168.1.10
ping.exe -n 50 192.168.1.10
Test-NetConnection 192.168.1.10 -Port 6001
```

预期结果：

- Ping 50/50
- TCP 6001 能回显
- 关闭 `YT_PACKET_DIAG` 后 Ping 平均小于 1 ms

## 9. 重要注意事项

1. 本板 PHY 实际 MDIO 地址是 **7**，不要按原理图误配为 1。
2. 本板 RGMII IO 是 **1.8 V**，不是 LVCMOS25/2.5 V。
3. 不要将 YT8521 `A001` 改成 `0x8060`；本板实测关闭 RX clock delay 后千兆丢包更严重。
4. `A003 = 0x00FD` 是本板最终使用的 1000M TX delay 配置，约 1.95 ns。
5. `lwip220_dhcp` 保持 `true` 是模板要求，不代表应用必须使用 DHCP。
6. 修改 BSP 源码后要先备份，再重新编译；不要直接 Regenerate BSP。