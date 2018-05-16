title: ftrace调试经验
author: Joy Xu
date: 2018-05-16 10:34:06
tags: frace
---

# 基础命令
		#如果没有mount debugfs,否则跳过
		mkdir /debugfs
		mount -t debugfs nodev debugfs/
		cd debugfs/tracing/

		cd /sys/kernel/debug/tracing

		echo > /sys/kernel/debug/tracing/trace
		echo > /sys/kernel/debug/tracing/set_ftrace_filter
		echo funcgraph-proc > trace_options
		echo function_graph > /sys/kernel/debug/tracing/current_tracer
		echo kvm_vcpu_ioctl *pl011* > set_ftrace_filter
		echo 'kvm:*' 'irq:irq_handler_entry' 'irq:irq_handler_exit' 'sched:*' > /sys/kernel/debug/tracing/set_event

		echo 1 > /sys/kernel/debug/tracing/tracing_on
		cd ~/
		taskset 0x0080 qemu-system-aarch64 -m 1024 -cpu host -M virt -smp 1 -nographic -kernel Image -initrd mini-rootfs-arm64.cpio.gz --enable-kvm &
		iperf -c 192.168.11.2
		echo 0 > /sys/kernel/debug/tracing/tracing_on  
		cat  /sys/kernel/debug/tracing/trace > ~/trace.log
		cat ~/trace.log

