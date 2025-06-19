◼(Q1) GEM5 + NVMAIN BUILD-UP    

◼(Q2) Enable L3 last level cache in GEM5 + NVMAIN

◼(Q3) Config last level cache to  2-way and full-way associative cache and test performance(必須跑benchmark quicksort在 2-way跟 full way)
  
◼(Q4) Modify last level cache policy based on frequency based replacement policy

◼(Q5) Test the performance of write back and write through policy based on 4-way associative cache with 
isscc_pcm(必須跑benchmark multiply 在 write through 跟 write back ( gem5 default 使用 write back，可以用 write request的數量判斷write through是否成功)
    
◼Bonus:
Design last level cache policy to reduce the energy consumption of pcm_based main memory 
(Baseline:LRU)
