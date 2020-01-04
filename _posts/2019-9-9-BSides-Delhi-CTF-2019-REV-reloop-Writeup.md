---
title: [Writeup] BSides Delhi CTF 2019 REV reloop
---

Hi,
 
Challenge file is [here](https://drive.google.com/file/d/1JRm-E1HRVU8EmAlhz1ajADmUEyk6hpF4/view)
  
Following is my Angr-based solver: 

<!--more-->

```python

import angr
import commands

bin_name = "level"

print commands.getstatusoutput('tar xvf %s.tar; chmod +x %s'%(bin_name,bin_name))

iter = 0
key_addr = 0x10000
while True:
    print("Iteration %d"%iter)
    b = angr.Project(bin_name)
    state = b.factory.blank_state(addr=0x400865)
    state.regs.rdi = key_addr

    key_len = 0x15
    for i in range(key_len):
        v = state.solver.BVS('x{}'.format(i), 8)
        state.memory.store(state.regs.rdi + i,v)

    sm = b.factory.simulation_manager(state)

    try:
        correct_addr = 0x400000 | open(bin_name).read().index("\xEB\x05\xB8\x01\x00\x00\x00")
    except:
        print "FLAG: %s"%(commands.getstatusoutput('./level')[1])
        quit()

    sm.explore(find=correct_addr+2)

    key_state = sm.found[0]
    
    key_data = key_state.memory.load(key_addr, key_len - 1)

    for i in range(key_len):
        const_0 = key_data.get_byte(i) >= ord('a')
        const_1 = key_data.get_byte(i) <= ord('z')
        const_2 = key_data.get_byte(i) >= ord('A')
        const_3 = key_data.get_byte(i) <= ord('Z')
        const_4 = key_state.solver.And(const_0, const_1)
        const_5 = key_state.solver.And(const_2, const_3)
        key_state.add_constraints(key_state.solver.Or(const_4, const_5))      

    key = key_state.solver.eval(key_data, cast_to=bytes).strip(b'\0\n')
    print key
    print commands.getstatusoutput('chmod +x level; echo %s | ./level; tar xvf drop'%(key))
    iter+=1

```

Thanks :))
