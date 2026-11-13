# Zynq-7000 + YT8521 + lwIP220 裸机千兆网修复导出

导出日期：2026-11-13
工程：D:\XilinxProjects\ps_eth_lwip
板卡：HZ-ZYNQ_EVM / Zynq-7010
PHY：YT8521，RGMII，MDIO 地址实际为 7
软件：Vitis Unified IDE 2025.2，standalone，lwIP 2.2.0（lwip220），lwip_echo_server

## 1. 最终可用配置

- PC：192.168.1.89/24
- 板卡：192.168.1.10/24
- 网关：192.168.1.1
- 链路：1 Gbps，经千兆交换机
- TCP echo 端口：6001
- MIO16..27：slow slew + 12 mA
- MIO_PIN_16 = 0x1402
- MIO_PIN_22 = 0x1403
- YT8521 A001 = 0x8160（保持 RX clock delay，UTP_TO_RGMII）
- YT8521 A003 = 0x00FD（1000M TX delay = 13 * 150 ps = 1.95 ns）
- lwIP 静态 IP，应用不启动 DHCP
- BSP 中 lwip220_dhcp 仍为 true，仅用于满足 lwip_echo_server 模板校验

最终实测：

    Ping 50 次：50/50，0% 丢包
    Ping 10 次：10/10
    TCP 6001 回显：发送 HELLO_YT8521_123，接收 HELLO_YT8521_123

## 2. 根本原因

不是 DHCP、IP 网段、ARP、lwIP 或 PHY 地址问题。

根因是 Zynq GEM RGMII TX 的 MIO 驱动能力偏弱：

    MIO 原始/之前配置：slow slew + 8 mA
    + 22 欧姆串阻和板级走线
    -> 125 MHz DDR TX 信号在 YT8521 输入端眼图过小
    -> PHY 统计出现 TX nosfd=1 / TX err
    -> 1 Gbps 大量丢包，100 Mbps 正常

将 MIO 驱动强度改为 12 mA 后，TX 边沿改善，PHY 端连续统计为：

    GEM TXSR=00000000 YT_TX valid=1 err=0 frag=0 nosfd=0

## 3. 修改过的文件

### 3.1 应用 main.c

当前文件：

    files/main.c

原工程路径：

    D:\XilinxProjects\ps_eth_lwip\vitis\lwip_echo_server\src\main.c

关键修改一：增加头文件

    #include "xil_io.h"
    #include "xemacps.h"

关键修改二：init_platform() 后设置 GEM0 RGMII MIO 为 slow slew + 12 mA

    {
        u32 i;

        Xil_Out32(0xF8000008U, 0x0000DF0DU);  /* SLCR unlock */

        for (i = 16U; i <= 27U; i++) {
            u32 mio = 0xF8000700U + (4U * i);

            /*
             * bit8    = 0  : slow slew
             * bit10:9 = 10 : 12 mA drive
             */
            Xil_Out32(mio, (Xil_In32(mio) & ~0x00000700U) | 0x00000400U);
        }

        Xil_Out32(0xF8000004U, 0x0000767BU);  /* SLCR lock */

        xil_printf("[APP] GEM0 MIO16..27 slew=slow drive=12mA, MIO_PIN_16=%08x MIO_PIN_22=%08x\r\n",
                   Xil_In32(0xF8000740U), Xil_In32(0xF8000758U));
    }

关键修改三：静态 IP

    IP4_ADDR(&ipaddr,  192, 168, 1, 10);
    IP4_ADDR(&netmask, 255, 255, 255, 0);
    IP4_ADDR(&gw,      192, 168, 1, 1);

应用不调用 dhcp_start()。

### 3.2 BSP PHY 驱动 xemacpsif_physpeed.c

当前文件：

    files/xemacpsif_physpeed.c

原工程路径：

    D:\XilinxProjects\ps_eth_lwip\vitis\platform\ps7_cortexa9_0\standalone_ps7_cortexa9_0\bsp\libsrc\lwip220\src\lwip-2.2.0\contrib\ports\xilinx\netif\xemacpsif_physpeed.c

新平台对应路径：

    <新platform>\ps7_cortexa9_0\standalone_ps7_cortexa9_0\bsp\libsrc\lwip220\src\lwip-2.2.0\contrib\ports\xilinx\netif\xemacpsif_physpeed.c

该文件包含 YT8521 支持：

- PHY ID：ID1=0x0000，ID2=0x011A
- YT8521 扩展寄存器访问（0x1E/0x1F）
- soft reset
- 关闭 auto-sleep
- RXC clock 配置
- A001 / A003 RGMII 配置
- A003 = 0x00FD
- 自协商、等待 AN 完成、读取速度
- 当前还保留了调试打印和 PHY packet generator 诊断代码（功能上可保留，正式版可清理）

注意：不建议照搬 YT8511 文章的 A001 bit8=0 方案。该实验在本板实测失败，A001 应保持 0x8160。

### 3.3 xemacpsif_dma.c

当前文件：

    files/xemacpsif_dma.c

该文件不是联网必需修改项，只包含调试统计：

- GEM RX / GEM SEND 打印
- ARP 十六进制打印
- GEM TXSR 打印
- YT8521 TX valid/err/frag/nosfd 统计

正式版本可以移除这些调试代码。若要继续排查，可保留。

## 4. Patch 文件

    patches/main.c.patch
    patches/xemacpsif_physpeed.c.patch

其中：

- main.c.patch 是从早期 main.c 到当前 main.c 的差异
- xemacpsif_physpeed.c.patch 是从原始 Xilinx lwIP 文件到当前 YT8521 版本+调试代码的完整差异

由于 patch 里包含调试代码，正式移植时建议以 README 中的“最小修改”说明为准，或者先从当前文件复制后自行清理诊断打印。

## 5. 下一次新建工程的步骤

### 情况 A：继续使用当前 platform/BSP

只需要：

1. 新建 lwip_echo_server 应用
2. 在 main.c 中加入 MIO slow slew + 12 mA 配置
3. 设置静态 IP
4. 不调用 dhcp_start
5. 编译、下载

### 情况 B：新建 platform/BSP

1. 新建 platform，选择 XSA
2. standalone_ps7_cortexa9_0
3. 启用 lwip220
4. BSP 参数：
   - lwip220_dhcp = true（模板校验要求，应用仍可用静态 IP）
   - lwip220_pbuf_pool_size 按需设置（当前工程为 16384）
   - lwip220_mem_size 按需设置（当前工程为 524288）
   - 使能 XILTIMER interval timer
5. 将 files/xemacpsif_physpeed.c 复制到新 BSP 对应路径
6. 在 main.c 中加入 MIO 12 mA 配置
7. 设置静态 IP，不调用 dhcp_start
8. 编译 BSP 的 liblwip220.a
9. 如果使用导出 platform 方式，同步 liblwip220.a 到 export platform 的 lib 目录
10. 重新编译并下载应用

## 6. 编译命令（本工程实际使用）

BSP：

    cd D:\XilinxProjects\ps_eth_lwip\vitis\platform\ps7_cortexa9_0\standalone_ps7_cortexa9_0\bsp\libsrc\build_configs\gen_bsp
    D:\Xilinx\2025.2\Vitis\bin\ninja.exe install

同步导出 platform 的 lwip 库：

    Copy-Item -LiteralPath "D:\XilinxProjects\ps_eth_lwip\vitis\platform\ps7_cortexa9_0\standalone_ps7_cortexa9_0\bsp\lib\liblwip220.a" `
      -Destination "D:\XilinxProjects\ps_eth_lwip\vitis\platform\export\platform\sw\standalone_ps7_cortexa9_0\lib\liblwip220.a" -Force

应用：

    cd D:\XilinxProjects\ps_eth_lwip\vitis\lwip_echo_server\build
    D:\Xilinx\2025.2\Vitis\bin\ninja.exe

下载：

    D:\Xilinx\2025.2\Vitis\bin\xsct.bat D:\XilinxProjects\eth_yt8521\flash.tcl

## 7. 验证命令

PowerShell：

    arp.exe -d 192.168.1.10
    ping.exe -n 50 192.168.1.10

串口应看到：

    MIO_PIN_16=00001402 MIO_PIN_22=00001403
    YT8521 A001=8160 A003=00FD
    link speed for phy address 7: 1000
    GEM TXSR=00000000 YT_TX valid=1 err=0 frag=0 nosfd=0

TCP 回显：

    Test-NetConnection 192.168.1.10 -Port 6001

## 8. 重要坑点

1. MIO RGMII IO 电压是 1.8 V，不是 LVCMOS25，不要改成 2.5 V。
2. 不要为了让静态 IP 生效而关闭 lwip220_dhcp；lwip_echo_server 模板要求该 BSP 参数为 true。
3. 不要在修改 BSP 源码后直接 Regenerate BSP 而不备份，否则 YT8521 驱动修改可能丢失。
4. 本板 PHY 实际 MDIO 地址是 7，不是原理图看起来的 001；以实际扫描/读取结果为准。
5. A001 不要改成 0x8060；本板实测关闭 RX clock delay 后千兆丢包更严重。
6. A003 的 0xFC/0xFD/0xFE 在 12 mA 下都可通，最终使用标准 0xFD（约 1.95 ns）。
7. 当前测试经过千兆交换机；直连 PC 还需要再跑一次 50 次 Ping 做最终确认。
应用 patch 时建议使用：

    git apply --ignore-space-change --ignore-whitespace patches/xemacpsif_physpeed.c.patch
    git apply --ignore-space-change --ignore-whitespace patches/main.c.patch

如果 patch 因原文件版本或行尾不同无法应用，直接使用 files/ 下的完整文件替换更可靠。
## 9. Ping 延迟 8ms 的原因

8ms 延迟不是网络、PHY 或 lwIP 协议栈造成的，而是逐包诊断打印造成的。

原 xemacpsif_dma.c 中，每个接收/发送报文都会执行：

    GEM RX[...] ...
    GEM SEND[...] ...
    GEM TXSR=... YT_TX valid=...

同时 GEM TX 完成中断里还会通过 MDIO 读取 4 个 YT8521 统计寄存器。

串口波特率是 115200，约 86.8 us/字符。一行 50~70 字符需要约 4~6 ms，一个 Ping 的 RX+SEND 两行打印合计约 8~12 ms。因此 Ping 显示 8ms 左右；第一次 Ping 还会因为 ARP 和更多打印达到 30~42ms。

已在 xemacpsif_dma.c 增加：

    #define YT_PACKET_DIAG 0

关闭逐包 UART/MDIO 诊断后，实测：

    Ping 20 次：20/20
    Minimum = 0ms
    Maximum = 1ms
    Average = 0ms

TCP 6001 回显仍然正常：

    LATENCY_TEST_OK

正式版本建议保持 YT_PACKET_DIAG=0；需要重新抓包分析时再改为 1。