Hi !

On executing the file, the process will be duplicated 400 times. Each process creates its child by a command line that contains 3 arguments. The first and second arguments are numbers used in a hashing function and used also to generate the next process arguments. The third argument is the process number.
So, the task typically is about finding which child process contains the flag (or what are the corrects args to pass to the program).
My approach to solve the tass was first to generate the correct arguments sequence numbers, then try to pass them to the hashing function "sub_401070" with a string at .text:00401149 , and try to figure out the result.
<!--more-->
The script I created to generate the sequence was a little bit complicated, the result was :

    3026539702 3580248161 2
    466510610 2867152813 3
    2580226910 609694577 4
    3729770 4114200461 5
    ....
    2941701190 2496288805 397
    1288739010 2151119537 398
    2134084791 912261120 399
    684983738 221932960 400
    994078838 1090513760 401


Then I wrote the hashing function that will test all the arguments triple :

The C program 

```C
#include
#include
#include

//.data:00407030 =  flag
//sub_401070 = hashing function

unsigned int sub_401050(unsigned int value)
{
    int shift = 5;
      if ((shift &= sizeof(value)*8 - 1) == 0)
      return value;
    return (value <> (sizeof(value)*8 - shift));
} 

unsigned int sub_401060(unsigned int value)
{
    int shift = 11;
      if ((shift &= sizeof(value)*8 - 1) == 0)
      return value;
    return (value <> (sizeof(value)*8 - shift));
}

unsigned char *hash(unsigned char *a1, unsigned int a2, unsigned int a3)
{
  unsigned int x;
  int y,i,j; 
 unsigned char *H; 
  x = 27;  
  H = (unsigned char*)malloc(x * sizeof(unsigned char));  
  memset(H, 0, 4 * (x >> 2));  
    y = (int)((char *)H + 4 * (x >> 2));
  for ( i = x & 3; i; --i )
    *(unsigned char *)y++ = 0;         
  for ( j = 0; j < x - 1; j += 2 )
  {
    *((unsigned char *)H + j) = a2 ^ a1[j];
    a2 = sub_401050(a2) ^ 0x2F;        
    *((unsigned char *)H + j + 1) = a3 ^ a1[j + 1];            
    a3 = a2 ^ sub_401060(a3);  
  }  
  return H;
}

unsigned int sub_401280(unsigned int a1,  unsigned int a2)
{
   unsigned int  i; 
   unsigned int  v4;
  v4 = 1;
  for ( i = 0; i < a2; ++i )
    v4 *= a1;
  return v4;
}

main()
{
unsigned int arg1, arg2, arg3;

unsigned char flag[27] = {0x0F, 0x8E , 0x9E , 0x39 , 0x3D , 0x5E ,0x3F ,0xA8 , 0x7A , 0x68 ,0x0C ,0x3D, 0x8B, 0xAD, 0xC5,  0xD0,
0x7B,0x09, 0x34, 0xB6, 0xA3, 0xA0, 0x3E, 0x67,  0x5D, 0xD6};

   FILE *flagsf = fopen("flags", "a");
   FILE *argfile =  fopen("clone_args.txt", "r");   
       if (!argfile)
    {
       printf("\nArg File Open Unsuccessful\n");
       exit (0);
    }   
   char line[100]; 
          while ( fgets( line, 100, argfile ) != NULL ) 
            { 
              sscanf(line, "%d %d %d", &arg1, &arg2, &arg3);
               fprintf(flagsf, "flag = %s (%u, %u)\n", hash(flag, arg1, arg2), arg1, arg2);                            
            } 
fclose(argfile);
fclose(flagsf);
system("pause");    
}
```

The execution result gives a good result using the couple (2937051214, 477169888)  and the flag is :
**And Now His Watch is Ended**

Thanks
