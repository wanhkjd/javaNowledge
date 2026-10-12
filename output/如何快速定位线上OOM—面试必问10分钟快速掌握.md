# 如何快速定位线上 OOM

> 来源主题：B站视频 BV1fm4y1R7or《如何快速定位线上OOM—面试必问》｜ 主讲：徐庶老师
> **说明**：该视频无 AI 字幕可提取，本文**非逐帧提取**，而是基于通用知识对标该主题整理，结构/细节可能与原视频不完全一致。

---

## 一、什么是 OOM

`OutOfMemoryError`：JVM 无法再分配所需内存、且 GC 也无法腾出足够空间时抛出的**错误（Error，非 Exception）**。先判断是哪一类，才能对症排查：

| OOM 类型 | 报错关键字 | 典型原因 |
|----------|-----------|----------|
| 堆溢出 | `Java heap space` | 对象过多/内存泄漏/堆设太小 |
| GC 开销超限 | `GC overhead limit exceeded` | GC 耗时 >98% 却回收 <2% 堆 |
| 元空间 | `Metaspace` | 动态生成大量类（反射/CGLIB/热部署） |
| 无法建线程 | `unable to create new native thread` | 线程泄漏、线程池无界 |
| 直接内存 | `Direct buffer memory` | NIO/Netty 堆外内存未释放 |

---

## 二、事前准备：让 OOM「可被定位」

线上 OOM 往往一闪而过，**没有堆快照就只能靠猜**。启动参数务必提前配好：

```bash
# OOM 时自动导出堆快照（最关键）
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump/heap.hprof

# GC 日志（JDK 8）
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/log/gc.log
# GC 日志（JDK 9+）
-Xlog:gc*:file=/data/log/gc.log:time,uptime

# 堆大小建议 Xms = Xmx，避免运行时抖动
-Xms2g -Xmx2g
```

---

## 三、定位四步法（核心流程）

### Step 1 — 看日志，判断 OOM 类型
在应用日志中搜 `java.lang.OutOfMemoryError`，确认是堆 / 元空间 / 线程 / 直接内存哪一类；同时看监控面板的 **CPU、内存、GC 频率与耗时、线程数** 变化趋势。

### Step 2 — 看进程与 GC 状态
```bash
jps -l                      # 找到 Java 进程 PID
jstat -gcutil <pid> 1000    # 每 1s 打印各区使用率与 GC 次数/耗时
jmap -heap <pid>            # 查看堆各区容量与使用（新版 JDK 用 jhsdb jmap --heap --pid <pid>）
```
**判断信号**：`FGC`（Full GC）次数飙升、每次耗时变长，且老年代 `O` 使用率贴近 100% → 堆 OOM 的典型征兆。

### Step 3 — 抓堆快照（Heap Dump）
- **优先用自动生成的 dump**（第二节的参数）：对线上无额外影响。
- 手动导出（会触发 STW，大堆慎用）：
  ```bash
  jmap -dump:live,format=b,file=/data/dump/heap.hprof <pid>
  ```
- 若是**线程类 OOM**（`unable to create new native thread`），改用 jstack 看线程数与堆栈，排查线程泄漏：
  ```bash
  jstack <pid> > /data/dump/thread.txt
  ```

### Step 4 — 分析 dump，锁定根因
用 **MAT（Eclipse Memory Analyzer）** 或 **Arthas / JProfiler / VisualVM** 打开堆快照：

| MAT 功能 | 作用 |
|----------|------|
| Histogram | 按类统计实例数 / 占用内存，找「谁最多」 |
| Dominator Tree | 找「谁支配了最多内存」 |
| Leak Suspects 报告 | 自动给出可疑泄漏点 |
| GC Roots 引用链 | 顺链找到对象**为什么回收不掉** |

---

## 四、Arthas 在线排查（不停机）

线上不能重启、需动态观察时，用 Arthas 实时诊断：

```bash
dashboard        # 实时查看内存/GC/线程总览
memory           # 查看各内存区使用
thread -n 3      # 查看最忙的 3 个线程
heapdump /tmp/dump.hprof   # 在线导出堆快照
```

---

## 五、常见根因与对策

| 根因 | 现象 | 对策 |
|------|------|------|
| 内存泄漏 | 老年代持续上涨、FGC 后降不下来 | 静态集合只增不减、流/连接未关、`ThreadLocal` 未 `remove`、缓存无上限 → 修复引用、加容量上限与过期、必要时用弱引用 |
| 内存不足（非泄漏） | 一次性加载大数据 / 大对象 | 调大堆、分页分批、流式处理 |
| Metaspace | 动态类暴涨（反射/CGLIB/热部署） | 控制类生成、调大 `-XX:MetaspaceSize` |
| 线程 OOM | 线程数异常多 | 用**有界**线程池、复用线程、谨慎调小 `-Xss` |
| 直接内存 | 堆外内存涨、堆却正常 | 排查 `DirectByteBuffer`、设置 `-XX:MaxDirectMemorySize` |

---

## 六、面试回答模板

> 线上 OOM 我一般按**四步**走：① 先看日志里的 `OutOfMemoryError` 关键字确定是堆/元空间/线程哪一类，并结合监控看 GC 和内存趋势；② 用 `jps`+`jstat -gcutil` 看 GC 频率和老年代占用，用 `jmap -heap` 看堆分布；③ 拿堆快照——线上我们都提前配了 `-XX:+HeapDumpOnOutOfMemoryError` 自动 dump，必要时 `jmap -dump`；④ 用 MAT 看 Histogram 和 Dominator Tree，顺着 GC Roots 引用链定位泄漏对象。如果不能停机，就用 Arthas 的 `dashboard`/`heapdump` 在线分析。最后根据根因做修复，比如一次我们遇到的就是静态 Map 缓存只增不减导致的泄漏，加了容量上限和过期后解决。

---

## 七、总结

- **可定位的前提**：启动参数提前加 `-XX:+HeapDumpOnOutOfMemoryError` + GC 日志。
- **四步法**：看日志定类型 → `jstat`/`jmap` 看 GC → 抓 Heap Dump → MAT/Arthas 析根因。
- **不停机**首选 Arthas；**找泄漏**首选 MAT 的 Dominator Tree + GC Roots 引用链。

---

> 本文档由 AI 基于通用知识整理（非该视频字幕提取），命令与参数以你所用 JDK 版本为准，线上操作（尤其 `jmap -dump`）注意 STW 影响。

