# 問題
◼(Q1) GEM5 + NVMAIN BUILD-UP

◼(Q2) Enable L3 last level cache in GEM5 + NVMAIN

◼(Q3) Config last level cache to  2-way and full-way associative cache and test performance(必須跑benchmark quicksort在 2-way跟 full way)
  
◼(Q4) Modify last level cache policy based on frequency based replacement policy

◼(Q5) Test the performance of write back and write through policy based on 4-way associative cache with 
isscc_pcm(必須跑benchmark multiply 在 write through 跟 write back ( gem5 default 使用 write back，可以用 write request的數量判斷write through是否成功)
    
◼Bonus:
Design last level cache policy to reduce the energy consumption of pcm_based main memory 
(Baseline:LRU)

---
# 實作方式
## Q1. GEM5 + NVMAIN BUILD-UP
version : Ubuntu18.04
  
1.安裝編譯工具:  
`sudo apt install build-essential git m4 scons zlib1g zlib1g-dev libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev python3-dev python3-six python libboost-all-dev pkg-config`  

2.安裝GEM5  
  
3.解壓GEM5丟進HOME後編譯:    
`scons build/X86/gem5.opt -j 4`  

4.用GIT安裝NVMAIN:  
`git clone https://github.com/SEAL-UCSB/NVmain`  
把nvmain跟gem5放到同一個目錄下  

5.修改SCONSCRIPT:  
進入nvmain資料夾點開SConscript 把36行的from gem5_scons import Transform註解掉(記得要存檔)  

6.編譯NVMAIN:  
在NVmain目錄下`scons --build-type=fast`  

7.修改GEM5 OPTIONS:  
在gem5/configs/common/Options.py中第133行加入下段程式(記得要存檔):  
```python
 for arg in sys.argv:
  if arg[:9] == "--nvmain-":
  parser.add_option(arg, type="string", default="NULL", help="Set NVMain configuration value for a parameter")
```

8.把剛剛前面nvmain sconscript註解掉的指令還原  

9.在GEM5目錄下混合NVMAIN編譯GEM5:  
`scons EXTRAS=../NVmain build/X86/gem5.opt`  
  
10.test:  
`./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello --cputype=TimingSimpleCPU --caches --l2cache -mem-type=NVMainMemory --nvmainconfig=../NVmain/Config/PCM_ISSCC_2012_4GB.config`  
出現Hello World，代表成功了  
## Q2.Enable L3 last level cache in GEM5 + NVMAIN
1.在`./config/common/Caches.py`增加L3 Cache的class:  
(直接抄檔案裡的L2Cache)  
```python
class L3Cache(Cache):
    assoc = 8
    tag_latency = 20
    data_latency = 20
    response_latency = 20
    mshrs = 20
    tgts_per_mshr = 12
    write_buffers = 8
```

2.修改`./config/common/CachesConfig.py`，增加l3_cache_class以及對應的O3_ARM_v7aL3:  
```python
        dcache_class, icache_class, l2_cache_class, l3_cache_class, walk_cache_class = \
            O3_ARM_v7a_DCache, O3_ARM_v7a_ICache, O3_ARM_v7aL2, O3_ARM_v7aL3, \
            O3_ARM_v7aWalkCache
    else:
        dcache_class, icache_class, l2_cache_class, l3_cache_class, walk_cache_class = \
            L1_DCache, L1_ICache, L2Cache, L3Cache, None
```
以及在原本只有L2cache的情況下再增加L3的情況:    
```python
    if options.l2cache and  options.l3cache:
        system.l2 = l2_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l2_size,
                                   assoc=options.l2_assoc)
	system.l3 = l3_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l3_size,
                                   assoc=options.l3_assoc)
 
        system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
	system.tol3bus = L3XBar(clk_domain = system.cpu_clk_domain)
 
        system.l2.cpu_side = system.tol2bus.master
        system.l2.mem_side = system.tol3bus.slave
 
        system.l3.cpu_side = system.tol3bus.master
        system.l3.mem_side = system.membus.slave
```

3.修改./src/mem/XBar.py(同樣抄L2):  
```python
class L3XBar(CoherentXBar):
    # 256-bit crossbar by default
    width = 32

    # Assume that most of this is covered by the cache latencies, with
    # no more than a single pipeline stage for any packet.
    frontend_latency = 1
    forward_latency = 0
    response_latency = 1
    snoop_response_latency = 1

    # Use a snoop-filter by default, and set the latency to zero as
    # the lookup is assumed to overlap with the frontend latency of
    # the crossbar
    snoop_filter = SnoopFilter(lookup_latency = 0)

    # This specialisation of the coherent crossbar is to be considered
    # the point of unification, it connects the dcache and the icache
    # to the first level of unified cache.
    point_of_unification = True
```

4.在./src/cpu/BaseCPU.py裡import剛剛增加的L3Xbar:  
```python
from XBar import L3XBar
```
以及增加l3的Hierarchy(同樣抄L2):  
```python
def addThreeLevelCacheHierarchy(self, ic, dc, l3c, iwc=None, dwc=None, xbar=None):
    self.addPrivateSplitL1Caches(ic, dc, iwc, dwc)
    self.toL3Bus = xbar if xbar else L3XBar()
    self.connectCachedPorts(self.toL3Bus)
    self.l3cache = l3c
    self.toL3Bus.master = self.l3cache.cpu_side
    self._cached_ports = ['l3cache.mem_side']
```

5.最後在./configs/common/Options.py增加可以enable l3cache的指令就大功告成了:  
```python
parser.add_option("--l3cache", action="store_true")
```

6.編譯完後跑  
`./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config`  
然後發現stat.txt裡多了l3的資料代表成功了  
## Q3.Config last level cache to  2-way and full-way associative cache and test performance  
1.先編譯產生執行檔:  
`gcc --static qicksort.c -o quicksort`  

2.可以直接透過command修改L3的associate以及其他層的size:  
像這樣:`--l1d_assoc=4` 、 `--l3_size=1MB`  
  
2-way指令:`./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config `  

full-way指令:`./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=16384 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config`  
  
備註:網路上好像寫assoc = 1的時候是full-way，但實際測試發現改成1的時候 # of replacements反而更高，猜測assoc = 1的時候可能不是full-way  
## Q4.Modify last level cache policy based on frequency based replacement policy  
1.可以先在./src/mem/cache/replacement_policies/ReplacementPolicies.py中找到GEM5有支援的policies  

2.根據助教給的"FREQUENCY BASED REPLACEMENT POLICY"圖，推測應該是least frequently used algorithm，對應到支援policies中的LFURP，直接在./config/common/Caches.py中L3cache的class底下新增:  
```python
replacement_policy = Param.BaseReplacementPolicy(LFURP(),"Replacement policy")
```  
可以在config.ini中確認是否有成功更改  
## Q5.Test the performance of write back and write through policy based on 4-way associative cache with isscc_pcm  
1.觀察./src/mem/cache/base.cc:  
可以看到原本writeBack的作法:  
```cpp
    if (pkt->isClean() && blk && blk->isDirty()) {
        // A cache clean opearation is looking for a dirty
        // block. If a dirty block is encountered a WriteClean
        // will update any copies to the path to the memory
        // until the point of reference.
        DPRINTF(CacheVerbose, "%s: packet %s found block: %s\n",
                __func__, pkt->print(), blk->print());
        PacketPtr wb_pkt = writecleanBlk(blk, pkt->req->getDest(), pkt->id);
        writebacks.push_back(wb_pkt);
        pkt->setSatisfied();
    }
```  
他的作法是遇到dirty block且收到isClean()封包時，就製造一個新的writeclean封包(wb_pkt)，將內容寫到下層記憶體(pkt->req->getDest())然後清除這個dirtyblock。  
所以，我們可以仿照他的作法在BaseCache::access中修改他的write行為(在1073行中加入):  
```cpp
if (blk->isWritable()) {
    PacketPtr writeclean_pkt = writecleanBlk(blk, pkt->req->getDest(), pkt->id);
    writebacks.push_back(writeclean_pkt);
}
```  
意思相當於每次只要找到可寫入的block，就產生一個writeClean封包，把資料往下層傳，所以會一直傳到記憶體，便實現了writeThrough  

2.驗證:可以觀察修改前後跑multiply的log檔，會發現writeRequest不同









