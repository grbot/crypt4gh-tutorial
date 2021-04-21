# crypt4gh-tutorial

The first part of the tutorial is on how to use the C implementation of crypt4gh to encrypt and read encrypted files using samtools. Most information was shared with by Robert Davies.

The second part of the tutorial will handle crypt4gh on FUSE mounts. This will allow for encryption and reading / manipulating of encrypted files by any tools on the system.

## Test samtools on C implementation of crypt4gh

Get source code
```
mkdir $HOME/opt; cd $HOME/opt
git clone https://github.com/samtools/htslib.git
git clone https://github.com/samtools/samtools.git
git clone https://github.com/samtools/htslib-crypt4gh.git
```

Install htslib
```
cd htslib; git submodule update --init --recursive; autoreconf -i; ./configure --prefix=$HOME/opt/samtools --enable-plugins; make && make install
```

Install samtools
```
cd ../samtools; autoreconf -i; ./configure --prefix=$HOME/opt/samtools --with-htslib=$HOME/opt/samtools LDFLAGS="-Wl,-R$HOME/opt/samtools/lib"; make && make install
```

Install htslib-crypt4g
```
cd ../htslib-crypt4gh; autoreconf -i; ./configure --prefix=$HOME/opt/samtools; make && make install
```

To generate some keys, you can use:
```
$HOME/opt/samtools/bin/crypt4gh-agent -g my_key
```
It will ask you for a passphrase twice, then generate the keys. You should end up in a new shell, with a CRYPT4GH_AGENT environment variable set in it. This is used by HTSlib to talk to the agent.

If you already have the keys, you can start it with:
```
$HOME/opt/samtools/bin/crypt4gh-agent -k my_key.pub -k my_key.sec
```

You only actually need both keys if you want to read and write files reading needs my_key.sec and writing my_key.pub. If you only want to do one of these, you can leave out the other key for slightly improved security. An example of where this might be useful is where you want to encrypt the data produced by your sequencing machine. The process doing this only needs the public key, which means you can keep the secret part locked away somewhere secure. Anyone compromising the process would be able to see the data being worked on at the time, but would not be able to read any of the other files that you've made. It also means you don't have to enter a passphrase, as they're only needed for secret keys.

Once you're in the shell started by the agent, you can read and write encrypted files using samtools, for example:

Set the HTS_PATH to the plugin first
```
export HTS_PATH=$HOME/opt/htslib-crypt4gh/plugin/
```

Grab a file:
```
curl 'ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR531/ERR5313570/MILK-10285F1.210210_A01250_0009_AH32GCDRXY.2t316.cram' > /tmp/MILK-10285F1.cram
curl 'ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR531/ERR5313536/MILK-11786A3.210210_A01250_0009_AH32GCDRXY.2t282.cram' > /tmp/MILK-11786A3.cram
```
Convert to BAM and encrypt:
```
$HOME/opt/samtools/bin/samtools view -b -o crypt4gh:/tmp/secret.bam /tmp/MILK-11786A3.cram
```
You can use the htsfile utility to find out what you made:
```
$HOME/opt/samtools/bin/htsfile /tmp/secret.bam
/tmp/secret.bam:  crypt4gh data
```
```
$HOME/opt/samtools/bin/htsfile crypt4gh:/tmp/secret.bam
crypt4gh:/tmp/secret.bam: BAM version 1 compressed sequence data
```
You can work on it with samtools:
```
$HOME/opt/samtools/bin/samtools view -H /tmp/secret.bam
@HD	VN:1.6	SO:coordinate
@SQ	SN:MN908947.3	LN:29903	UR:https://www.ncbi.nlm.nih.gov/nuccore/MN908947.3?report=fasta	AS:MN908947.3	M5:105c82802b67521950854a851fc6eefd	SP:SARS-CoV-2 isolate Wuhan-Hu-1
@PG	PN:bwa	ID:bwa	VN:0.7.17-r1188	CL:bwa mem -t 4 MN908947.3.fa 36316_2#316_1_val_1.fq.gz 36316_2#316_2_val_2.fq.gz
@PG	PN:dehumanizer	ID:dehumanizer.20210215	VN:0.8.1	CL:/lustre/scratch121/esa-analysis-20200609/tmp/kl2/miniconda3/bin/dehumanise /lustre/scratch121/esa-analysis-20200609/tmp/kl2/ftc/manifest.txt /dev/stdin --preset sr --bam -o /dev/stdout --trash-minalen 25 --log dhlog.log	PP:bwa
@PG	ID:samtools	PN:samtools	PP:dehumanizer.20210215	VN:1.11	CL:/software/pkgg/samtools/1.11.0/bin/samtools view -h -
@PG	ID:samtools.1	PN:samtools	PP:samtools	VN:1.11	CL:/software/pkgg/samtools/1.11.0/bin/samtools view -b -
@PG	ID:samtools.2	PN:samtools	PP:samtools.1	VN:1.11	CL:/software/pkgg/samtools/1.11.0/bin/samtools view -C -
@PG	ID:samtools.3	PN:samtools	PP:samtools.2	VN:1.12-5-ge2d9a70	CL:/home/gerrit/opt/samtools/bin/samtools view -b -o crypt4gh:/tmp/secret.bam /tmp/MILK-10285F1.cram
@PG	ID:samtools.4	PN:samtools	PP:samtools.3	VN:1.12-5-ge2d9a70	CL:/home/gerrit/opt/samtools/bin/samtools view -H /tmp/secret.bam
```

When reading files, recent versions of HTSlib (1.11+) should be able to detect crypt4gh files on reading and automatically pass them to the plugin to decrypt. Older versions will need a bit of help, so you have to include the 'crypt4gh:' prefix.

When you've finished work, typing `exit` in the shell will quit it and close down the agent process.

One other trick you can do with the agent is to give it a command to run,
for example:
```
$HOME/opt/samtools/bin/crypt4gh-agent -k my_key.pub -- $HOME/opt/samtools/bin/samtools view -b -o crypt4gh:/tmp/secret.bam /tmp/MILK-10285F1.cram
```
should start the agent, run samtools in it to encrypt the file and then shut everything down again. You won't have to type a passphrase in this case as you didn't give it a secret key. In theory you can run entire pipelines like this, as long as they don't try to run any remote processes.

Also can add `--verbosity=8` to the samtools command to get more details on the plugin added.

## Test samtools on crypt4ghfs

I followed the steps here: https://github.com/EGA-archive/crypt4ghfs with slight modifications.

Install necessary packages
```
sudo apt-get install ca-certificates pkg-config git gcc make automake autoconf libtool bzip2 zlib1g-dev libssl-dev libedit-dev ninja-build cmake udev libc6-dev
```

I have a Python 3.7 conda environment already set up

```
conda activate py3.7
```

```
conda install meson pytest
```

Install libfuse
```
git clone https://github.com/libfuse/libfuse.git
cd libfuse
git checkout fuse-3.10.0
mkdir build
cd build
meson ..
ninja
ninja install
```

Install sshfs
```
git clone https://github.com/libfuse/sshfs.git
cd sshfs
git checkout sshfs-3.7.0
mkdir build
cd build
meson ..
ninja
ninja install
```

Setup library path
```
export LD_LIBRARY_PATH=/usr/local/lib/x86_64-linux-gnu/
```

Install cryptfhfs
```
pip install crypt4ghfs
```

Also install crypt4gh to encrypt a test file. It seems that encrypted files generated with htslib-crypt4gh  are not compatible with crypt4ghfs.
```
pip install crypt4gh
```

Check my keys and config
```
ls $HOME/crypt4gh-tutorial
crypt4ghfs.conf  my_key.pub  my_key.sec
```

Enabled user_allow_other in `/usr/local/etc/fuse.conf`

Set permissions on configuration file (example of config file is [here](crypt4ghfs.conf))
```
chmod 600 $HOME/crypt4gh-tutorial/crypt4ghfs.conf
```
Now lets encrypt a CRAM file

First get the CRAM file
```
curl 'ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR531/ERR5313536/MILK-11786A3.210210_A01250_0009_AH32GCDRXY.2t282.cram' > tmp/MILK-11786A3.cram
```

And encrypt using crypt4gh

```
crypt4gh encrypt --sk $HOME/crypt4gh-tutorial/my_key.sec --recipient_pk $HOME/crypt4gh-tutorial/my_key.pub < tmp/MILK-11786A3.cram > encrypted-files/MILK-11786A3.c4gh
```

Now lets mount crypt4ghfs

```
crypt4ghfs --conf $HOME/crypt4gh-tutorial/crypt4ghfs.conf $HOME/crypt4gh-tutorial/clear-files/
```

Lets view with samtools
```
$HOME/opt/samtools/bin/samtools view clear-files/MILK-11786A3 | head -n2
A01250:9:H32GCDRXY:2:2250:1145:29121	2145	MN908947.3	18	60	94H58M69H	=	28263	28447	TCCCAGGTAACAAACCAACCAACTTTCGATCTCTTGTAGATCTGTTCTCTAAACGAAC	FFFFFFFFFFFF:FFFFFFFFFFFF:F:F:F:FFFFFFFFF:FFFFF:FFFFFFFFFF	MC:Z:8M1D193M	AS:i:58	XS:i:0	SA:Z:MN908947.3,27447,+,94M127S,60,0;MN908947.3,28255,+,141S16M1D64M,60,4;	MD:Z:58	NM:i:0
A01250:9:H32GCDRXY:2:2152:1723:4445	2145	MN908947.3	18	60	94H58M69H	=	28263	28447	TCCCAGGTAACAAACCAACCAACTTTCGATCTCTTGTAGATCTGTTCTCTAAACGAAC	FFFFFFFFFFFF:FFFFFFFFFF:FFFFFFFF:FFF::FFFFFFFFFFFF:FFFFF,F	MC:Z:8M1D193M	AS:i:58	XS:i:0	SA:Z:MN908947.3,27447,+,94M127S,60,0;MN908947.3,28255,+,141S16M1D64M,60,4;	MD:Z:58	NM:i:0
```
