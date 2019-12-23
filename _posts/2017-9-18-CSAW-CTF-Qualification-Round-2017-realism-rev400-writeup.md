
Hi,

This is a quick and dirty writeup for realism task.

The binary is a 16-bits bootloader.

The flag checking function is based on PSADBW instruction.

According to documentation, the instruction computes the absolute value of the difference of 8 unsigned byte integers from the source operand (second operand) and from the destination operand (first operand). These 8 differences are then summed to produce an unsigned word integer result that is stored in the destination operand.

In the given program the instruction operates on xmm2 and xmm5 registers.

When operating on 128-bit operands, two packed results are computed. Here, the 8 low-order bytes of the source and destination operands are operated on to produce a word result that is stored in the low word of the destination operand, and the 8 high-order bytes are operated on to produce a word result that is stored in bits 64 through 79 of the destination operand. The remaining bytes of the destination operand are cleared.

Then the computed sum of absolute differences are compared against excpected result. 2 chars are compared at one time.  

Following is my solver.

```python
from z3 import *

def psadbw(v1,v2):
    s = 0
    for i in range(len(v1)):
        s+= abs(v1[i] - v2[i])
    return s
    
def zabs(x):
    return If(x >= 0,x,-x)
   
def symabs(v1,v2):
    s = 0
    for i in range(len(v1)):
        s+= zabs(v1[i] - v2[i])
    return s
    
m1 = [0x22, 0x0f, 0x02, 0xc8, 0x83, 0xfb, 0xe0, 0x83] 
m2 = [0xc0, 0x20, 0x0f, 0x10, 0xcd, 0x00, 0x13, 0xb8]

X = [34,15,2,-324,131,251,224,0]
      
r = psadbw(X, m1)
print "0x%x"%r

s = Solver()

f1 = Int('f1') 
f2 = Int('f2') 
f3 = Int('f3') 
f4 = Int('f4') 
f5 = Int('f5') 
f6 = Int('f6') 
f7 = Int('f7') 
f8 = Int('f8')
g1 = Int('g1') 
g2 = Int('g2') 
g3 = Int('g3') 
g4 = Int('g4') 
g5 = Int('g5') 
g6 = Int('g6') 
g7 = Int('g7') 
g8 = Int('g8')

'''
s.add(f1 >= 32,f1 <=126)
s.add(f2 >= 32,f2 <=126)
s.add(f3 >= 32,f3 <=126)
s.add(f4 >= 32,f4 <=126)
s.add(f5 >= 32,f5 <=126)
s.add(f6 >= 32,f6 <=126)
s.add(f7 >= 32,f7 <=126)
s.add(f8 >= 32,f8 <=126)
'''
'''
0x2df028f
0x290025d
0x2090221
0x27b0278
0x1f90233
0x25e0291
0x2290255
0x2110270
'''

s.add( zabs(f1-m1[0])+zabs(f2-m1[1])+zabs(f3-m1[2])+zabs(f4-m1[3])+zabs(f5-m1[4])+zabs(f6-m1[5])+zabs(f7-m1[6])+zabs(m1[7]) == 0x28f  )
s.add( zabs(f1)+zabs(f2)+zabs(f3)+zabs(f4)+zabs(f5)+zabs(f6)+zabs(0x2)+zabs(f8-0x8f) == 0x25d  )
s.add( zabs(f1)+zabs(f2)+zabs(f3)+zabs(f4)+zabs(f5)+zabs(0)+zabs(f7-0x2)+zabs(f8-0x5d) == 0x221  )
s.add( zabs(f1)+zabs(f2)+zabs(f3)+zabs(f4)+zabs(0)+zabs(f6)+zabs(f7-0x2)+zabs(f8-0x21) == 0x278  )
s.add( zabs(f1)+zabs(f2)+zabs(f3)+zabs(0)+zabs(f5)+zabs(f6)+zabs(f7-0x2)+zabs(f8-0x78) == 0x233  )
s.add( zabs(f1)+zabs(f2)+zabs(0)+zabs(f4)+zabs(f5)+zabs(f6)+zabs(f7-0x2)+zabs(f8-0x33) == 0x291  )
s.add( zabs(f1)+zabs(0)+zabs(f3)+zabs(f4)+zabs(f5)+zabs(f6)+zabs(f7-0x2)+zabs(f8-0x91) == 0x255  )
s.add( zabs(0)+zabs(f2)+zabs(f3)+zabs(f4)+zabs(f5)+zabs(f6)+zabs(f7-0x2)+zabs(f8-0x55) == 0x270  )


s.add( zabs(g1-m2[0])+zabs(g2-m2[1])+zabs(g3-m2[2])+zabs(g4-m2[3])+zabs(g5-m2[4])+zabs(g6-m2[5])+zabs(g7-m2[6])+zabs(m2[7]) == 0x2df  )
s.add( zabs(g1)+zabs(g2)+zabs(g3)+zabs(g4)+zabs(g5)+zabs(g6)+zabs(0x2)+zabs(g8-0xdf) == 0x290  )
s.add( zabs(g1)+zabs(g2)+zabs(g3)+zabs(g4)+zabs(g5)+zabs(0)+zabs(g7-0x2)+zabs(g8-0x90) == 0x209  )
s.add( zabs(g1)+zabs(g2)+zabs(g3)+zabs(g4)+zabs(0)+zabs(g6)+zabs(g7-0x2)+zabs(g8-0x09) == 0x27b  )
s.add( zabs(g1)+zabs(g2)+zabs(g3)+zabs(0)+zabs(g5)+zabs(g6)+zabs(g7-0x2)+zabs(g8-0x7b) == 0x1f9  )
s.add( zabs(g1)+zabs(g2)+zabs(0)+zabs(g4)+zabs(g5)+zabs(g6)+zabs(g7-0x1)+zabs(g8-0xf9) == 0x25e  )
s.add( zabs(g1)+zabs(0)+zabs(g3)+zabs(g4)+zabs(g5)+zabs(g6)+zabs(g7-0x2)+zabs(g8-0x5e) == 0x229  )
s.add( zabs(0)+zabs(g2)+zabs(g3)+zabs(g4)+zabs(g5)+zabs(g6)+zabs(g7-0x2)+zabs(g8-0x29) == 0x211  )

s.add(g1 >= 32,g1 <=126)
s.add(g2 >= 32,g2 <=126)
s.add(g3 >= 32,g3 <=126)
s.add(g4 >= 32,g4 <=126)
s.add(g5 >= 32,g5 <=126)
s.add(g6 >= 32,g6 <=126)
s.add(g7 >= 32,g7 <=126)
s.add(g8 >= 32,g8 <=126)
s.add(f1 >= 32,f1 <=126)
s.add(f2 >= 32,f2 <=126)
s.add(f3 >= 32,f3 <=126)
s.add(f4 >= 32,f4 <=126)
s.add(f5 >= 32,f5 <=126)
s.add(f6 >= 32,f6 <=126)
s.add(f7 >= 32,f7 <=126)
s.add(f8 >= 32,f8 <=126)



'''
s.add( pabs(g1-m2[0])+pabs(g2-m2[1])+pabs(g3-m2[2])+pabs(g4-m2[3])+pabs(g5-m2[4])+pabs(g6-m2[5])+pabs(g7-m2[6])+pabs(0x00-m2[7]) == 0x2df)
s.add(  (pabs(g1-m2[0])+pabs(g2-m2[1])+pabs(g3-m2[2])+pabs(g4-m2[3])+pabs(g5-m2[4])+pabs(g6-m2[5])+pabs(0x00-m2[6])+pabs(g8-m2[7]))  == 0x290)
s.add(  (pabs(g1-m2[0])+pabs(g2-m2[1])+pabs(g3-m2[2])+pabs(g4-m2[3])+pabs(g5-m2[4])+pabs(0x00-m2[5])+pabs(g7-m2[6])+pabs(g8-m2[7]))  == 0x209)
s.add  ((pabs(g1-m2[0])+pabs(g2-m2[1])+pabs(g3-m2[2])+pabs(g4-m2[3])+pabs(0x00-m2[4])+pabs(g6-m2[5])+pabs(g7-m2[6])+pabs(g8-m2[7])) == 0x27b)
s.add(  (pabs(g1-m2[0])+pabs(g2-m2[1])+pabs(g3-m2[2])+pabs(0x00-m2[3])+pabs(g5-m2[4])+pabs(g6-m2[5])+pabs(g7-m2[6])+pabs(g8-m2[7]))  == 0x1f9)
s.add(  (pabs(g1-m2[0])+pabs(g2-m2[1])+pabs(0x00-m2[2])+pabs(g4-m2[3])+pabs(g5-m2[4])+pabs(g6-m2[5])+pabs(g7-m2[6])+pabs(g8-m2[7]))  == 0x25e)
s.add( (pabs(g1-m2[0])+pabs(0x00-m2[1])+pabs(g3-m2[2])+pabs(g4-m2[3])+pabs(g5-m2[4])+pabs(g6-m2[5])+pabs(g7-m2[6])+pabs(g8-m2[7])) == 0x229)
#s.add( (pabs(0x00-m2[0])+pabs(g2-m2[1])+pabs(g3-m2[2])+pabs(g4-m2[3])+pabs(g5-m2[4])+pabs(g6-m2[5])+pabs(g7-m2[6])+pabs(g8-m2[7])) == 0x211)

#s.add(g1 >= -256,g1 <=256)
s.add(g1 == 0x7d, g1 == -0x7d)
#s.add(g2 >= -256,g2 <=256)
#s.add(g3 >= -256,g3 <=256)
#s.add(g4 >= -256,g4 <=256)
#s.add(g5 >= -256,g5 <=256)
#s.add(g6 >= -256,g6 <=256)
#s.add(g7 >= -256,g7 <=256)
#s.add(g8 >= -256,g8 <=256)
'''

s.check()

print s.model()
```

The flag is **flag{4r3alz_m0d3_y0}**


Cheers :))
