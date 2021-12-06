# COMPSCI 377 LAB: Xv6 Filesystem

## Purpose

This lab is designed to familiarize students with the Xv6 learning operating system and more specifically familiarize students with the Xv6 filesytem. Please make sure that all of your answers to questions in these labs come from work done on the Edlab environment – otherwise, they may be inconsistent results and will not receive points.

Please submit your answers to this lab on Gradescope in the assignment marked “Lab #3’. All answers are due by the time specified on Gradescope. The TA present in your lab will do a brief explanation of the various parts of this lab, but you are expected to answer all questions by yourself. Please raise your hand if you have any questions during the lab section – TAs will be notified you are asking a question. Questions and Parts have a number of points marked next to them to signify their weight in this lab’s final grade. Labs are weighted equally, regardless of their total points.

Once you have logged in to Edlab, you can clone this lab repo using

```bash
git clone https://github.com/jbettencourt10/377-lab-filesystem_xv6.git
```

Then, clone the Xv6 repo into the 377-lab-gdb_xv6 directory using

```bash
git clone https://github.com/mit-pdos/xv6-public.git
```

Finally, you can use `cd` to open the directory you just cloned:

```bash
cd 377-lab-gdb_xv6
```

The xv6 OS includes a Makefile that allow you to locally compile the OS run all the sample code listed in this tutorial. You can compile them by running `make`. Feel free to modify the source files yourself, after making changes you can run `make` again to build new binaries from your modified files. You can also use `make clean` to remove all the built files, this command is usually used when something went wrong during the compilation so that you can start fresh.

## Part 1: xv6 filesystem (5 Points)

Xv6 is a small operating system designed by MIT for students to learn how operating systems work. It includes several features we have or will learn about, like a scheduler, processes, file system, and system calls. For the sake of this lab, we will be investigating how Xv6 implements an ordinary filesystem by looking through the source code available to us.

To run xv6, first use the 'make' command in the directory. Then, after completion, use the 'make qemu-nox' command in the same directory. A shell will appear, and you will be using the Xv6 OS.


A snippet from proc.c
```c++
// Blocks.

// Allocate a zeroed disk block.
static uint
balloc(uint dev)
{
  int b, bi, m;
  struct buf *bp;

  bp = 0;
  for(b = 0; b < sb.size; b += BPB){
    bp = bread(dev, BBLOCK(b, sb));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        bp->data[bi/8] |= m;  // Mark block in use.
        log_write(bp);
        brelse(bp);
        bzero(dev, b + bi);
        return b + bi;
      }
    }
    brelse(bp);
  }
  panic("balloc: out of blocks");
}

// Free a disk block.
static void
bfree(int dev, uint b)
{
  struct buf *bp;
  int bi, m;

  bp = bread(dev, BBLOCK(b, sb));
  bi = b % BPB;
  m = 1 << (bi % 8);
  if((bp->data[bi/8] & m) == 0)
    panic("freeing free block");
  bp->data[bi/8] &= ~m;
  log_write(bp);
  brelse(bp);
}
```

fs.h
```c++
// On-disk file system format.
// Both the kernel and user programs use this header file.


#define ROOTINO 1  // root i-number
#define BSIZE 512  // block size

// Disk layout:
// [ boot block | super block | log | inode blocks |
//                                          free bit map | data blocks]
//
// mkfs computes the super block and builds an initial file system. The
// super block describes the disk layout:
struct superblock {
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes.
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
  uint inodestart;   // Block number of first inode block
  uint bmapstart;    // Block number of first free map block
};

#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEV only)
  short minor;          // Minor device number (T_DEV only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};

// Inodes per block.
#define IPB           (BSIZE / sizeof(struct dinode))

// Block containing inode i
#define IBLOCK(i, sb)     ((i) / IPB + sb.inodestart)

// Bitmap bits per block
#define BPB           (BSIZE*8)

// Block of free map containing bit for block b
#define BBLOCK(b, sb) (b/BPB + sb.bmapstart)

// Directory is a file containing a sequence of dirent structures.
#define DIRSIZ 14

struct dirent {
  ushort inum;
  char name[DIRSIZ];
};

```

From the above files, and the files listed on gradescope, we believe that looking through the source code should be a sufficient amount of information to answer the questions on Gradescope. However, there exists a wide range of documentation about Xv6 on the internet, so feel free to look for that if you find it necessary. A useful page is https://github.com/YehudaShapira/xv6-explained/blob/master/Explanations.md.

## Part 2: Programming an xv6 filesystem function (5 Points)

Here, we provide a snippet for you to implement in xv6 to print free and allocated blocks in the file system. The final result should look like something resembling the following:

```
Free blocks:
728  729  730  731  732  733  734  735  736  737  738  739  740  741  742  743  
744  745  746  747  748  749  750  751  752  753  754  755  756  757  758  759  
........
968  969  970  971  972  973  974  975  976  977  978  979  980  981  982  983  
984  985  986  987  988  989  990  991  992  993  994  995  996  997  998  999  
Total free blocks = 272

Alocated blocks:
0  1  2  3  4  5  6  7  8  9  10  11  12  13  14  15  
16  17  18  19  20  21  22  23  24  25  26  27  28  29  30  31  
........
704  705  706  707  708  709  710  711  712  713  714  715  716  717  718  719  
720  721  722  723  724  725  726  727  
Total allocated blocks = 728
```


It is imperative that you understand how the filesystem works in xv6 before attempting this, so be very clear with what is written in `fs.c` and `fs.h` before attempting a direct implementation.

Print File System code snippet
```c++
int
pfs()
{
  int b, bi, m;
  struct buf *bp;
  int countt = 0, countf = 0, counta = 0, bn = 0, nb=0;;
  bp = 0;
  cprintf("\nFree blocks:\n");
  for(b = 0; b < sb.size; b += BPB){
    nb++;
    bp = bread(1, BBLOCK(b, sb));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      ++countt;
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        //!!!YOU DO THIS!!!
        // What should we do if a block is free?
      }
      bn++;
    }
    brelse(bp);
  }
  cprintf("\nTotal free blocks = %d", countf );

  cprintf("\nAlocated blocks:\n");
  bn = 0;
  
  //!!!!!!YOU DO THIS!!!!!!!!!
}
```

Our hope is that you will finish the implementation wherever you see "YOU DO THIS" and your output will match the output listed above. Remember to ask your TA for help if you find that you need help during the lab with your implementation.

## Please submit your answers to these questions on gradescope!
