# 给下一个对话的启动提示

我在继续 Zynq-7000（Zynq-7010）+ YT8521 RGMII + Vitis Unified IDE + lwIP220 裸机工程。

已经定位并修复千兆网问题，最终有效配置是：

- GEM0 MIO16..27：slow slew + 12 mA，MIO_PIN_16=0x1402，MIO_PIN_22=0x1403
- YT8521 A001=0x8160，A003=0x00FD（1000M TX delay 约 1.95 ns）
- 静态 IP：192.168.1.10/24，网关 192.168.1.1，PC 192.168.1.89
- 应用不启动 DHCP；BSP 的 lwip220_dhcp 仍保持 true 以满足 lwip_echo_server 模板校验
- TCP echo 端口：6001
- 当前测试经千兆交换机：Ping 50/50，TCP 回显正常；关闭逐包诊断后 Ping 平均 <1ms

根本原因是 MIO 原先只有 8 mA 驱动能力，1 Gbps RGMII TX 眼图不足。不是 DHCP、IP 网段、ARP 或 lwIP 问题。

请先读取：

    D:\XilinxProjects\eth_yt8521\export_yt8521_lwip_fix_20261113\README.md
    D:\XilinxProjects\eth_yt8521\export_yt8521_lwip_fix_20261113\files\main.c
    D:\XilinxProjects\eth_yt8521\export_yt8521_lwip_fix_20261113\files\xemacpsif_physpeed.c

重要约束：

- 一次只执行一个明确步骤
- 不要 Regenerate BSP，除非先备份 YT8521 驱动修改
- 不要把 MIO 改成 LVCMOS25/2.5V；本板 RGMII 是 1.8V
- 不要把 A001 改成 0x8060；这个实验在本板失败
- 修改源码后更新时间戳，确保 Ninja 重新编译
- 当前 BSP 源码路径：
  D:\XilinxProjects\ps_eth_lwip\vitis\platform\ps7_cortexa9_0\standalone_ps7_cortexa9_0\bsp\libsrc\lwip220\src\lwip-2.2.0\contrib\ports\xilinx\netif\xemacpsif_physpeed.c
- 当前应用路径：
  D:\XilinxProjects\ps_eth_lwip\vitis\lwip_echo_server\src\main.c