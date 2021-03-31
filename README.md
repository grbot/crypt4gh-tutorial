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
cd htslib; ./configure --prefix=$HOME/opt/samtools; make && make install
```

Install samtools
```
cd ../samtools; ./configure --prefix=$HOME/opt/samtools --with-htslib=$HOME/opt/samtools LDFLAGS="-Wl,-R$HOME/opt/samtools/lib"; make && make install
```

Install htslib-crypt4g
```
cd ../htslib-crypt4gh; ./configure --prefix=$HOME/opt/samtools; make && make install
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

Grab a file:
```
curl 'ftp://ftp.sra.ebi.ac.uk/vol1/run/ERR531/ERR5313570/MILK-10285F1.210210_A01250_0009_AH32GCDRXY.2t316.cram' > /tmp/MILK-10285F1.cram
```
Convert to BAM and encrypt:
```
$HOME/opt/samtools/bin/samtools view -b -o crypt4gh:/tmp/secret.bam /tmp/MILK-10285F1.cram
```
You can use the htsfile utility to find out what you made:
```
$HOME/opt/samtools/bin/htsfile /tmp/secret.bam
/tmp/secret.bam: crypt4gh data
```
```
$HOME/opt/samtools/bin/htsfile crypt4gh:/tmp/secret.bam
crypt4gh:/tmp/secret.bam: BAM version 1 compressed sequence data
```
You can work on it with samtools:
```
$HOME/opt/samtools/bin/samtools view -H /tmp/secret.bam
@HD VN:1.6 SO:coordinate
... etc ...
```

When reading files, recent versions of HTSlib (1.11+) should be able to detect crypt4gh files on reading and automatically pass them to the plugin to decrypt. Older versions will need a bit of help, so you have to include the 'crypt4gh:' prefix.

When you've finished work, typing `exit` in the shell will quit it and close down the agent process.

One other trick you can do with the agent is to give it a command to run,
for example:
```
$HOME/opt/samtools/bin/crypt4gh-agent -k my_key.pub -- $HOME/opt/samtools/bin/samtools view -b -o crypt4gh:/tmp/secret.bam /tmp/MILK-10285F1.cram
```
should start the agent, run samtools in it to encrypt the file and then shut everything down again. You won't have to type a passphrase in this case as you didn't give it a secret key. In theory you can run entire pipelines like this, as long as they don't try to run any remote processes.
