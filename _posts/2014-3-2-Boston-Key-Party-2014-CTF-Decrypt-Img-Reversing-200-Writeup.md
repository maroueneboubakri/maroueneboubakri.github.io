
The task :

We encrypted an image that we drew in paint, but lost the original! Can you recover it for us?  
[Challenge file](http://bostonkeyparty.net/challenges/decryptimg-a921005aad6a6b6b445d0d754d54a311.zip)

First the program is compiled under Win8 using new Win8 API, so it can be executed only under Win8 or in some cases Wine. I don't have a Win8 machine so I tried to write a program that uses the Crypt DLL to perform encryption.

The encryption function signature is the following:

```C
void encrypt(unsigned int8[] file, int32 length, unsigned int8[] key, int32 keylen);
```

In our case the image file is encrypted using a 54 bytes key length.

<!--more-->

Here is an example program that encrypt a raw data.

```C
#include
#include
#include
#include
#include

typedef void(*pfunc)( char file[], int length, char key[], int keylen);
pfunc encrypt;
int main(int argc, char **argv)
{
HINSTANCE hLib = LoadLibrary("cryptxll.dll");

if(hLib==NULL)
{
cout << "Error! Can't open dll!";
return 1;
}
char dllpath[70];

GetModuleFileName((HMODULE)hLib,(LPTSTR)dllpath,70);

encrypt = (pfunc)GetProcAddress((HMODULE)hLib, "_encrypt@16");
if(encrypt == NULL)
{
cout << "Can't load encrypt function!" << endl;
system("pause");
FreeLibrary((HMODULE)hLib);
return 1;
}

char key[] = "azertyuiopmlkjhgfdswxcvbn123456789ABCDFEFGHIJKLMNOPQRS";
char filename[] = "test.bmp.encr";

FILE *fp;
long lSize;
char *buffer;

fp = fopen ( "c:\\test.bmp" , "rb" );
if( !fp ) perror("thefile"),exit(1);

fseek( fp , 0L , SEEK_END);
lSize = ftell( fp );
buffer = ( char *)malloc((lSize + 1)*sizeof( char));
rewind( fp );
if( !buffer ) fclose(fp),fputs("memory alloc fails",stderr),exit(1);
if( 1!=fread( buffer , lSize, 1 , fp) )
fclose(fp),free(buffer),fputs("entire read fails",stderr),exit(1);

int keylen = 54;
encrypt(buffer, lSize, key, keylen);
FILE* outf = fopen( filename, "wb" );
fwrite(buffer, 1, lSize, outf);
fclose(outf);

fclose(fp);
free(buffer);

FreeLibrary((HMODULE)hLib);
// system("pause");
return 0;
}
```

Now I have a test file with its encryted file to work on it.

After that I tried to reverse the encryption function and the following is the implementation of the cryption function :

```C
void derivk( char*key, int keylen)
{
long s;
int i,j;

for ( i = 0; i < keylen; i++ )
{
s += 2147483647 * key[i] + s / 715827883 + key[i];
}
srand(s);
for ( j = 0; j < keylen; j++ )
key[j] = rand() % 255;
}

void encrypt( char *file, int length, char *key, int keylen)
{

int kj, ki, i, fj;
char x, c;
char *keyi, *filei;

kj = 0;
ki = 1;
if ( keylen > 1 )
{
i = 1;
do
{
key[i] = (ki++) ^ key[i-1] ^ key[i];
i++;
}
while ( ki < keylen );
}


fj = 0;


if ( length > 0 )
{
keyi = key;
filei = file;
do
{
filei[fj] ^= keyi[kj++];
if ( kj == keylen )
{
derivk(keyi, keylen);
keyi = key;
filei = file;
kj = 0;
}
fj+=1;
}
while ( fj < length );
}
}
```

Generally, the key is computed like this :

for the first 54 Bytes:

    thekey[0] = plain[0] XOR crypted[0];  
    thekey[1] = plain[1] XOR crypted[1] XOR thekey[0] XOR 1;
    thekey[2] = plain[2] XOR crypted[2] XOR thekey[1] XOR 2;  
    ...  

After that the new derivated keys will be used, All we need to know is the first Key (the original one)

So to get the key we must know some parts of the plain data (the first 54 bytes)

Its a bitmap file, we know the structure of the BMP header file and some fields remain constants (like the type, bitmap offset, reserved.. )

The only three variables that we don't know them, is only the width, the height and the bitmap size of the picture, and we compute theme like this : We know that the file size is 1 440 054 bytes if we subtract the header size (54 bytes) it still 1 440 000 which can give us a 800 x 600 image size. Bitmap size can be computed now.  
Like this we discovered 54 bytes of the plain text and it is sufficient to calculate the key.

To do this, I coded this program :

```C
`#include  
#include  
#include  
  
main()  
{  
FILE *fp;  
long lSize;  
char *buffer;  
char *bufferor;  
  
fp = fopen ( "c:\\decryptme.bmp.bkenc" , "rb" );  
if( !fp ) perror("thefile"),system("pause"),exit(1);  
  
fseek( fp , 0L , SEEK_END);  
lSize = ftell( fp );  
  
buffer = ( char *)malloc((lSize)*sizeof( char));  
rewind( fp );  
if( !buffer ) fclose(fp),fputs("memory alloc fails",stderr),system("pause"),exit(1);  
  
if( 1!=fread( buffer , lSize, 1 , fp) )  
fclose(fp),free(buffer),fputs("entire read fails",stderr),system("pause"),exit(1);  
  
  
int keylen = 54;  
  
struct bmp_file_header {  
short type;  
long size;  
long reserved;  
long bitmap_offset;  
long header_size;  
signed long width;  
signed long height;  
short planes;  
short bits_per_pixel;  
long compression;  
long bitmap_size;  
signed long horizontal_resolution;  
signed long vertical_resolution;  
long num_colors;  
long num_important_colors;  
} __attribute__ ((packed));  
  
struct bmp_file_header *H = (struct bmp_file_header *) malloc(sizeof(struct bmp_file_header));  
  
H->type = 0x4D42;  
//const  
H->size = 0x15F936;  
  
H->reserved = 0; //const  
H->bitmap_offset = 0x36; //const  
H->header_size = 0x28; //const  
  
//60000  
H->width = 800;  
H->height = 600;  
  
H->planes = 0x1; //const  
H->bits_per_pixel = 0x18; //const  
H->compression = 0x0; //const  
H->bitmap_size = H->width * H->height * H->bits_per_pixel /8; // const  
  
H->horizontal_resolution = 0; //const  
H->vertical_resolution = 0; //const  
H->num_colors = 0; //const  
H->num_important_colors = 0; //const  
  
char thekey[55];  
  
int len = sizeof(bmp_file_header) + 2;  
  
char * raw_header = ( char *) malloc(len * sizeof( char));  
  
raw_header = reinterpret_cast(H);  
  
thekey[0] = raw_header[0] ^ buffer[0];  
thekey[1] = raw_header[1] ^ buffer[1] ^ thekey[0] ^ 1;  
raw_header[55] = 0;  
  
for (int i=0; i<54; i++)  
thekey[i] = buffer[i] ^ raw_header[i];  
for (int i=54; i>=0; i--)  
thekey[i] = thekey[i] ^ thekey[i-1] ^ i;  
  
thekey[54] = '\0';  
printf("\nthe key = %s\n", thekey);  
  
fclose(fp);  
free(buffer);  
system("pause");  
return 0;  
}`
```

This program will output the key :  
djkfah8Fh2389jksdhfSDHFJKLf894hlasdfhalsdf789h4JKASDHF

Since its a Symetric Encryption alorithm to get the plain image file we simply encrypt it with the key shown below. (Using the same program)

As an output we get the plain image file.

And the flag for a 200 PTS is : **DLL_importing_is_ez!**

Thanks !
